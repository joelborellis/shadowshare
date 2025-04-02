Great, I’ll dive into the Azure Function App repo and prepare a markdown report covering:

- Endpoint routes and how they work
- Request parameters and formats
- Response structure
- Example calls using `curl`, `fetch`, and C#/ASP.NET

This will all be geared toward helping a frontend dev integrate with the API in a dev environment.
I'll let you know once the report is ready for review.

# Shadow FastAPI Chat API Integration Guide

This guide explains how to integrate a frontend chat UI with the **Shadow** Azure Function App API (a FastAPI-based function) that serves as the chat backend. It covers available endpoints, request/response formats, and example calls from various environments. The function app is intended for development use (running locally or in a test environment), so it currently allows open access (no auth) and wide CORS for convenience.

## Endpoint Routes

The Shadow Function App exposes two POST endpoints for chat interactions (no GET endpoints are defined). Both endpoints accept the same request body schema but differ in how they return responses:

- **`POST /shadow-sk`** – *Streaming Chat Endpoint*. Accepts a chat query and returns responses as a streaming Server-Sent Events (SSE) stream. This endpoint is designed to stream back partial response chunks (for a real-time typing effect) ([shadow-fastapi/ShadowFunction/__init__.py at main · joelborellis/shadow-fastapi · GitHub](https://github.com/joelborellis/shadow-fastapi/blob/main/ShadowFunction/__init__.py#:~:text=%40app.post%28%22%2Fshadow)). Each chunk is sent as an SSE `data:` event (see Response format below).

- **`POST /shadow-sk-no-stream`** – *Non-Streaming Chat Endpoint*. Accepts the same chat query but returns a single complete response in JSON format (no streaming) ([shadow-fastapi/ShadowFunction/__init__.py at main · joelborellis/shadow-fastapi · GitHub](https://github.com/joelborellis/shadow-fastapi/blob/main/ShadowFunction/__init__.py#:~:text=%40app.post%28%22%2Fshadow)). Use this if you prefer to get the full answer in one go (at the cost of waiting for the entire response to be generated).

Both endpoints are defined with anonymous access (no API key or auth token needed) and have no URL prefix (the Azure Functions `routePrefix` is set to empty, so the routes are exactly as above with no `/api` prefix) ([shadow-fastapi/host.json at main · joelborellis/shadow-fastapi · GitHub](https://github.com/joelborellis/shadow-fastapi/blob/main/host.json#:~:text=)) ([shadow-fastapi/function_app.py at main · joelborellis/shadow-fastapi · GitHub](https://github.com/joelborellis/shadow-fastapi/blob/main/function_app.py#:~:text=import%20azure)). In development, the function app typically runs at `http://localhost:7071`, so the full URLs would be:

- http://localhost:7071/shadow-sk  
- http://localhost:7071/shadow-sk-no-stream  

*(If deployed to Azure, the base domain would differ, but the route paths remain the same since `routePrefix` is disabled.)*

## Request Structure

All endpoints expect requests to be JSON formatted with content type `application/json`. The JSON body should include the following fields ([shadow-fastapi/ShadowFunction/__init__.py at main · joelborellis/shadow-fastapi · GitHub](https://github.com/joelborellis/shadow-fastapi/blob/main/ShadowFunction/__init__.py#:~:text=class%20ShadowRequest)):

- **`query`** (`string`, required): The user's message or question to send to the chat AI. This is the prompt or query you want the assistant to answer.

- **`threadId`** (`string`, required): An identifier for the conversation thread. Use an empty string (or `""`) if starting a new conversation, and the API will create a new thread ID for you ([shadow-fastapi/ShadowFunction/__init__.py at main · joelborellis/shadow-fastapi · GitHub](https://github.com/joelborellis/shadow-fastapi/blob/main/ShadowFunction/__init__.py#:~:text=query%3A%20str)). If continuing an existing conversation, pass the `threadId` returned from the previous response to maintain context. (Each conversation thread stores the history of messages.)

- **`additional_instructions`** (`string`, optional): Any extra instructions or context for the assistant. This can be used to provide system-level hints or modify the assistant’s behavior for this query (e.g. a role prompt or specific guidelines). If not needed, you can omit this field or set it to `null` ([shadow-fastapi/ShadowFunction/__init__.py at main · joelborellis/shadow-fastapi · GitHub](https://github.com/joelborellis/shadow-fastapi/blob/main/ShadowFunction/__init__.py#:~:text=threadId%3A%20str)).

**Example Request JSON:**

```json
{
  "query": "Hello, how can I track my order?",
  "threadId": "",
  "additional_instructions": null
}
```

In this example, `threadId` is empty, indicating a new chat thread will be started. The API will generate a new `threadId` and include it in the response. On subsequent messages in the same conversation, use the returned `threadId` to continue the thread.

## Response Structure

The structure of the response depends on which endpoint you call:

### Streaming Response (`/shadow-sk`)

The streaming endpoint responds with **Server-Sent Events (SSE)**. The HTTP response has content type `text/event-stream` ([shadow-fastapi/ShadowFunction/__init__.py at main · joelborellis/shadow-fastapi · GitHub](https://github.com/joelborellis/shadow-fastapi/blob/main/ShadowFunction/__init__.py#:~:text=return%20StreamingResponse%28event_stream%28%29%2C%20media_type%3D%22text%2Fevent)), and the body is an event stream that sends incremental chat answer data. Each SSE message (event) contains a JSON payload in its `data` field. The JSON structure for each event is:
```json
{
  "data": "<partial_response_text>",
  "threadId": "<thread_id>"
}
```
Each `data` is a piece of the assistant’s answer text, and `threadId` is the conversation ID (same for all events in this response) ([shadow-fastapi/ShadowFunction/__init__.py at main · joelborellis/shadow-fastapi · GitHub](https://github.com/joelborellis/shadow-fastapi/blob/main/ShadowFunction/__init__.py#:~:text=data%20%3D%20json.dumps%28)). As the AI generates the answer, multiple SSE events will be sent, each with a segment of the answer. The client should concatenate the `data` segments in order to reconstruct the full answer. Once the full response is sent, the server will close the stream (end of response).

**Notes:**

- The `threadId` in the SSE events should be noted for continuing the conversation. It will be identical for all events in one response and matches the thread ID you sent (or a new one if you started a new thread).

- In case of an error during processing, the stream will include an event with `event: error` and a data payload like `{"error": "...error message..."}` ([shadow-fastapi/ShadowFunction/__init__.py at main · joelborellis/shadow-fastapi · GitHub](https://github.com/joelborellis/shadow-fastapi/blob/main/ShadowFunction/__init__.py#:~:text=)). The client should handle this by listening for events named `"error"` or checking if the parsed data contains an `error` field.

- **Consuming SSE on the client:** Standard web clients can read this stream incrementally. (See the example in the JavaScript section below for how to parse SSE events from the fetch stream.)

### Non-Streaming Response (`/shadow-sk-no-stream`)

The non-streaming endpoint returns a traditional JSON response (content type `application/json`). The response body will be a single JSON object with the following structure:

```json
{
  "data": "<full_response_text>",
  "threadId": "<thread_id>"
}
```

Here, `data` is the complete answer from the assistant (as a single concatenated string), and `threadId` is the conversation ID for this thread ([shadow-fastapi/ShadowFunction/__init__.py at main · joelborellis/shadow-fastapi · GitHub](https://github.com/joelborellis/shadow-fastapi/blob/main/ShadowFunction/__init__.py#:~:text=json_response%20%3D%20)). The client can display the `data` directly once received. Use the returned `threadId` in the next request to continue the conversation thread.

If an error occurs, the JSON response will contain an `error` field instead of `data`. For example, you might get:

```json
{ 
  "error": "Empty response from the agent.", 
  "threadId": "<thread_id>" 
}
```

or in other error cases: 

```json
{ "error": "Some error message" }
``` 

The presence of an `"error"` key indicates the request was not successful ([shadow-fastapi/ShadowFunction/__init__.py at main · joelborellis/shadow-fastapi · GitHub](https://github.com/joelborellis/shadow-fastapi/blob/main/ShadowFunction/__init__.py#:~:text=except%20HTTPException%20as%20exc%3A)). The `threadId` may or may not be present in error cases (if a thread had already been created). Clients should check for an `error` field in the response and handle it (e.g., display an error message to the user).

## Development Environment & Access

When running this Azure Function App in a local development environment (e.g., via Azure Functions Core Tools), note the following:

- **Base URL:** By default, the function app listens on `http://localhost:7071`. The `host.json` is configured with an empty route prefix, so you do **not** need an `/api` prefix in the URL ([shadow-fastapi/host.json at main · joelborellis/shadow-fastapi · GitHub](https://github.com/joelborellis/shadow-fastapi/blob/main/host.json#:~:text=)). Use the exact routes as listed above (e.g., `http://localhost:7071/shadow-sk`).

- **Authentication:** The HTTP trigger is set to "anonymous" access, meaning no function key or token is required to call the endpoints ([shadow-fastapi/function_app.py at main · joelborellis/shadow-fastapi · GitHub](https://github.com/joelborellis/shadow-fastapi/blob/main/function_app.py#:~:text=import%20azure)). This is convenient for development and testing. (In a production scenario, you might want to secure the endpoint, but currently it’s open.)

- **CORS:** The app enables CORS to allow calls from any origin in development. All origins, headers, and methods are permitted ([shadow-fastapi/ShadowFunction/__init__.py at main · joelborellis/shadow-fastapi · GitHub](https://github.com/joelborellis/shadow-fastapi/blob/main/ShadowFunction/__init__.py#:~:text=allow_origins%3D%5B)). This means you can call the API from your frontend (e.g. `localhost:3000` or any other origin) without running into CORS issues. (For production, you’d typically restrict this, but it’s wide open here for ease of testing.)

- **Environment Variables:** The function relies on certain environment variables (e.g., `ASSISTANT_ID`) internally to retrieve the AI model/agent. When running locally, make sure any required environment variables are set (for example, via a `local.settings.json` file). This is a back-end concern – as a frontend developer you just need to ensure the backend is configured properly. If the backend is running and returning responses, then it’s configured correctly.

## Example API Calls

Below are example calls to the Shadow API using different methods. In all examples, adjust the URL as needed (for local dev, use `http://localhost:7071`; for a deployed Azure Function, use its host URL). The request payload and response handling shown correspond to a simple query *"Hello"* (starting a new thread).

### Using cURL (Command Line)

You can test the API quickly with **cURL**. Make sure the function app is running, then use the following commands:

- **Streaming example (SSE):** This will initiate a chat query and stream back the response in chunks. We use `-N` (`--no-buffer`) to disable cURL’s output buffering so you can see the chunks as they arrive.

  ```bash
  curl -N -X POST http://localhost:7071/shadow-sk \
       -H "Content-Type: application/json" \
       -d '{"query": "Hello", "threadId": "", "additional_instructions": null}'
  ```

  **Output:** You should see a series of lines starting with `data: ` as the response streams. For example:

  ```text
  data: {"data":"Hi there!","threadId":"12345-abcde-..."}

  data: {"data":" How can I assist you today?","threadId":"12345-abcde-..."}

  ```
  Each `data:` line is one chunk of the answer in JSON format. In this case, the assistant responded with `"Hi there! How can I assist you today?"` split into two chunks. The `threadId` (`"12345-abcde-..."` in this example) is the new conversation ID. The stream ends when the full answer has been sent (indicated by no more `data:` lines and cURL returning to the prompt).

- **Non-streaming example:** If you want the full response in one go, use the no-stream endpoint:

  ```bash
  curl -X POST http://localhost:7071/shadow-sk-no-stream \
       -H "Content-Type: application/json" \
       -d '{"query": "Hello", "threadId": "", "additional_instructions": null}'
  ```

  **Output:** cURL will print a single JSON object, for example:
  ```json
  {"data":"Hi there! How can I assist you today?","threadId":"12345-abcde-..."}
  ```
  This contains the complete answer in the `data` field and the generated `threadId`. (If you pipe this output to `jq` or similar, you can pretty-print the JSON.)

### Using JavaScript (Fetch API)

In a web front-end, you can use the **Fetch API** to call these endpoints. Below are examples for both the streaming and non-streaming scenarios.

**1. Non-Streaming (simple fetch):**

For a straightforward request/response (no streaming), you can do a standard fetch POST and parse the JSON:

```js
const payload = {
  query: "Hello",
  threadId: "",
  additional_instructions: null
};

fetch("http://localhost:7071/shadow-sk-no-stream", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify(payload)
})
  .then(response => response.json())
  .then(data => {
    if (data.error) {
      console.error("API Error:", data.error);
    } else {
      console.log("Assistant response:", data.data);
      console.log("Thread ID:", data.threadId);
      // Use data.threadId for the next message in this conversation
    }
  })
  .catch(err => console.error("Request failed:", err));
```

This will log the full response when it arrives. If `data.error` is present, it logs an error. Otherwise, `data.data` contains the assistant's answer and `data.threadId` is the conversation ID.

**2. Streaming (SSE via fetch):**

For streaming, you need to read the response as a stream. The fetch API allows access to a **ReadableStream** of the body. You can read from it and parse SSE events manually. For example:

```js
const payload = {
  query: "Hello",
  threadId: "",
  additional_instructions: null
};

const response = await fetch("http://localhost:7071/shadow-sk", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify(payload)
});

// Get a reader for the response body stream
const reader = response.body.getReader();
const decoder = new TextDecoder("utf-8");
let buffer = "";

// Function to handle each complete event chunk
function handleEventChunk(chunk) {
  // chunk will include lines like "data: {...}\n\n"
  if (chunk.startsWith("data:")) {
    const jsonStr = chunk.slice("data:".length).trim();
    if (jsonStr !== "") {
      const event = JSON.parse(jsonStr);
      if (event.error) {
        console.error("API Error:", event.error);
      } else {
        console.log("Partial response:", event.data);
        // TODO: update your chat UI with the new partial text (event.data)
        // The threadId is event.threadId (same for all events in this stream)
      }
    }
  }
  // You could handle other event types if needed (e.g., "event: error"), 
  // but in this implementation, errors also come through as event.error in data.
}

while (true) {
  const { value, done } = await reader.read();
  if (done) break;
  // Decode the received bytes into text
  buffer += decoder.decode(value, { stream: true });
  // Process any complete events in the buffer
  let events = buffer.split("\n\n");  // SSE events are separated by a blank line
  buffer = events.pop() || "";        // any incomplete event remains in buffer
  for (const eventText of events) {
    handleEventChunk(eventText.trim());
  }
}
```

In this example, we post the request to `/shadow-sk`, then continuously read from the response stream. We accumulate text in a `buffer` until we detect one or more complete SSE messages (separated by double newline `\n\n`). Each SSE message begins with `data:` (per the SSE format). We parse the JSON after `data:` and log or handle the partial response.

On the frontend UI, you would append each `event.data` text to the chat output as it arrives, so the user sees the answer being typed out. Once the `reader.read()` loop ends, the response stream is finished (the answer is complete).

**Note:** The above streaming example uses `await/async` for clarity. If you prefer promise style or need older compatibility, you can use `.getReader().read().then(... recursive)` or consider using an `EventSource` polyfill that supports POST. However, since the endpoint is POST-only, the manual fetch approach is required (the native `EventSource` API only works with GET requests).

### Using C# / .NET (HttpClient)

If you are integrating from a .NET front-end (for example, a Blazor client or WPF app), you can use `HttpClient` to call the API.

**Non-Streaming Example (one-shot request):**

```csharp
using System;
using System.Net.Http;
using System.Text;
using System.Text.Json;
using System.Threading.Tasks;

...

HttpClient client = new HttpClient();

// Prepare JSON payload
var payloadObj = new {
    query = "Hello",
    threadId = "",
    additional_instructions = (string)null
};
string jsonPayload = JsonSerializer.Serialize(payloadObj);

// Send POST request
var content = new StringContent(jsonPayload, Encoding.UTF8, "application/json");
HttpResponseMessage response = await client.PostAsync("http://localhost:7071/shadow-sk-no-stream", content);

// Read and parse response
string jsonResponse = await response.Content.ReadAsStringAsync();
if (response.IsSuccessStatusCode) {
    using JsonDocument doc = JsonDocument.Parse(jsonResponse);
    JsonElement root = doc.RootElement;
    if (root.TryGetProperty("error", out JsonElement errElem)) {
        Console.WriteLine($"API Error: {errElem.GetString()}");
    } else {
        string answer = root.GetProperty("data").GetString();
        string threadId = root.GetProperty("threadId").GetString();
        Console.WriteLine($"Assistant response: {answer}");
        Console.WriteLine($"Thread ID: {threadId}");
    }
} else {
    Console.WriteLine($"HTTP Error: {response.StatusCode}");
}
```

This .NET example posts to the `/shadow-sk-no-stream` endpoint and reads the JSON response. It uses `System.Text.Json` to parse the returned JSON. You can then use the `answer` and `threadId` in your application (e.g., display the answer in the UI, store the threadId for the next query).

**Streaming in .NET:** The .NET `HttpClient` can also consume the streaming endpoint. To do so, you would set `HttpCompletionOption.ResponseHeadersRead` in the `PostAsync` call and then read from `response.Content` as a stream. You could use a `StreamReader` to read line-by-line. Each line that starts with `"data:"` will contain a JSON fragment similar to the JavaScript example. You will need to accumulate lines until a blank line is encountered (which signifies the end of one SSE message), then parse the JSON part. This approach is more involved, but the logic is analogous to the JavaScript streaming example above. In practice, if you need real-time updates in a .NET client, you might also consider using web sockets or another mechanism, but SSE can be read with careful stream handling.

---

By following this guide, a frontend developer can confidently integrate their chat UI with the Shadow Function App API. You can start a new conversation by sending an empty `threadId` and then continue the dialogue by including the returned `threadId` with each subsequent message. The streaming endpoint enables live partial responses for a better user experience, while the non-streaming endpoint offers simplicity for cases where real-time updates aren’t needed. Happy coding!

