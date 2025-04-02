Perfect, I’ll review the GitHub repository for the Azure Function App at https://github.com/joelborellis/shadow-fastapi-meg, and generate a detailed markdown report. This will include endpoint routes, request and response structures, and code examples using `curl`, JavaScript `fetch`, and C#/ASP.NET for frontend integration.

I’ll let you know as soon as the report is ready.

# Shadow MEG Chat API Documentation

This document describes the **Shadow MEG Chat API**, an Azure Function App running a FastAPI backend for a chat-based assistant. Frontend developers can use this API to send user messages and receive responses from the AI assistant in a conversational thread. All endpoints accept and return JSON data, and CORS is enabled for all domains by default ([github.com](https://github.com/joelborellis/shadow-fastapi-meg/raw/refs/heads/main/ShadowFunction/__init__.py#:~:text=tools,Configure)), making it easy to call these APIs from a web client. No authentication keys are required (the function is deployed with anonymous access) ([shadow-fastapi-meg/function_app.py at main · joelborellis/shadow-fastapi-meg · GitHub](https://github.com/joelborellis/shadow-fastapi-meg/blob/main/function_app.py#:~:text=from%20ShadowFunction%20import%20app%20as,fastapi_app)).

## API Endpoint: POST `/meg-chat`

**Description:** Submits a user query to the Shadow MEG assistant and returns the assistant’s response. This endpoint supports multi-turn conversations by using a `threadId` to track context across calls.

- **Method & URL:** `POST /meg-chat` (e.g. `https://<your-function-app>.azurewebsites.net/meg-chat` for the Azure deployment, or `http://localhost:7071/meg-chat` when running locally).
- **Auth:** None required (function allows anonymous access ([shadow-fastapi-meg/function_app.py at main · joelborellis/shadow-fastapi-meg · GitHub](https://github.com/joelborellis/shadow-fastapi-meg/blob/main/function_app.py#:~:text=from%20ShadowFunction%20import%20app%20as,fastapi_app))).
- **CORS:** All origins allowed (you can call this endpoint from any domain or port) ([github.com](https://github.com/joelborellis/shadow-fastapi-meg/raw/refs/heads/main/ShadowFunction/__init__.py#:~:text=tools,Configure)).

### Request Format (JSON Body)

The request must be a JSON object with the following fields ([github.com](https://github.com/joelborellis/shadow-fastapi-meg/raw/refs/heads/main/ShadowFunction/__init__.py#:~:text=module,if%20they%20are)):

- **`query`** (string, **required**): The user's message or question to send to the assistant.
- **`threadId`** (string, **required**): The conversation thread identifier. For a new conversation, use an empty string `""`. For subsequent messages in an ongoing conversation, provide the `threadId` received from the last response to continue that thread ([github.com](https://github.com/joelborellis/shadow-fastapi-meg/raw/refs/heads/main/ShadowFunction/__init__.py#:~:text=response.%20,async%20for)).
- **`additional_instructions`** (string, optional): Any extra instructions or context for the assistant. This can be omitted or set to `null` if not needed.

**Example Request JSON:** 

```json
{
  "query": "Hello, I need help with my account.",
  "threadId": "",
  "additional_instructions": null
}
```

In this example, `"threadId": ""` indicates a new conversation. The first call will create a new thread ID on the backend. For the next query, use the returned `threadId` from the response so the assistant has the context of the prior conversation.

### Response Format (JSON Body)

On success, the server returns an HTTP 200 response with a JSON body containing the assistant’s reply and the current thread ID. The JSON structure is as follows ([github.com](https://github.com/joelborellis/shadow-fastapi-meg/raw/refs/heads/main/ShadowFunction/__init__.py#:~:text=return%20%7B,json_response)):

- **`data`** (string): The assistant’s response message, which may include multiple sentences or a detailed answer.
- **`threadId`** (string): The conversation thread ID. Use this ID in the next request to continue the conversation thread. If the request provided an empty threadId (starting a new chat), this will be a newly generated ID ([github.com](https://github.com/joelborellis/shadow-fastapi-meg/raw/refs/heads/main/ShadowFunction/__init__.py#:~:text=response.%20,async%20for)). If a threadId was provided, it will usually echo that same ID (unless a new thread was created due to an error).
- **`error`** (string): *Only present if an error occurred.* In case of an error processing the request, the response will contain an `"error"` field with a message, instead of the `data` field.

**Example Success Response JSON:** 

```json
{
  "data": "Hello! I can certainly help you with your account. What do you need assistance with?",
  "threadId": "abc123-def456-gh7890"
}
```

In this example, `data` holds the assistant’s answer, and `threadId` is a unique identifier for this conversation. The frontend should store this `threadId` and send it with subsequent messages to maintain context.

**Example Error Response JSON:** 

```json
{
  "error": "Failed to retrieve the assistant agent."
}
```

An error like the above might occur if the backend failed to initialize the AI assistant or another internal issue happened. Always check for an `"error"` field in the response. On error, no `data` field will be present. In some cases (e.g. if the assistant returns an empty response), the error may include a threadId and message (for example: `{"error": "Empty response from the agent.", "threadId": "abc123-def456-gh7890"}`) ([github.com](https://github.com/joelborellis/shadow-fastapi-meg/raw/refs/heads/main/ShadowFunction/__init__.py#:~:text=additional_instructions%3Dadditional_instructions%29%3A%20if%20message.content.strip%28%29%3A%20,str%28e)).

### Example: Calling `/meg-chat` with `curl`

You can test the endpoint using `curl` from the command line. Make sure to replace `<your-function-app>` with your actual Azure Function App name if calling a deployed instance, or use `localhost:7071` if running locally.

```bash
curl -X POST "https://<your-function-app>.azurewebsites.net/meg-chat" \
     -H "Content-Type: application/json" \
     -d '{
           "query": "Hello, I need help with my account.",
           "threadId": "",
           "additional_instructions": null
         }'
```

**Explanation:** This `curl` command sends a POST request with a JSON body. We specify the content type as JSON. In this example, we ask a question "Hello, I need help with my account." and start a new thread (empty `threadId`). The server will respond with a JSON containing the answer and a new thread ID. You can then take that thread ID and use it in the next curl call by inserting it into the `threadId` field to continue the conversation.

### Example: Calling `/meg-chat` with JavaScript Fetch

For front-end applications, you can use the Fetch API to call the chat endpoint. Below is a JavaScript example (suitable for a browser environment, such as in a React/Vue app or plain HTML page):

```js
const apiUrl = "https://<your-function-app>.azurewebsites.net/meg-chat";  // use the actual URL
const requestData = {
  query: "Hello, I need help with my account.",
  threadId: "",  // empty for first message
  additional_instructions: null
};

fetch(apiUrl, {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify(requestData)
})
  .then(response => response.json())
  .then(data => {
    if (data.error) {
      console.error("API error:", data.error);
      return;
    }
    console.log("Assistant response:", data.data);
    console.log("Thread ID:", data.threadId);
    // You might store data.threadId for the next message in the conversation
  })
  .catch(err => {
    console.error("Request failed:", err);
  });
```

**Explanation:** This code sends a POST request to the `/meg-chat` endpoint with the required JSON body. It then parses the JSON response. If an `error` is returned, it logs the error. Otherwise, it logs the assistant's response and the `threadId`. In a real chat UI, you would display `data.data` to the user and save `data.threadId` (for example, in a state variable) to include in the next request so the conversation context is preserved.

### Example: Calling `/meg-chat` from C# (ASP.NET using HttpClient)

If you are integrating this API in a C# application (such as an ASP.NET frontend or Blazor app), you can use `HttpClient` to send requests. Below is an example of how to construct the request and handle the response in C#:

```csharp
using System.Net.Http;
using System.Net.Http.Headers;
using System.Text;
using System.Text.Json;
using System.Threading.Tasks;

// ...

HttpClient client = new HttpClient();
string apiUrl = "https://<your-function-app>.azurewebsites.net/meg-chat";

// Prepare the request payload as an object or anonymous type
var requestObj = new {
    query = "Hello, I need help with my account.",
    threadId = "",  // start new conversation
    additional_instructions = (string)null
};

// Serialize the object to JSON
string jsonPayload = JsonSerializer.Serialize(requestObj);
HttpContent content = new StringContent(jsonPayload, Encoding.UTF8, "application/json");

try
{
    HttpResponseMessage response = await client.PostAsync(apiUrl, content);
    if (response.IsSuccessStatusCode)
    {
        string responseJson = await response.Content.ReadAsStringAsync();
        // Deserialize the JSON into a dynamic object or a defined class
        using JsonDocument doc = JsonDocument.Parse(responseJson);
        JsonElement root = doc.RootElement;
        if (root.TryGetProperty("error", out JsonElement errorElem))
        {
            string errorMsg = errorElem.GetString();
            Console.WriteLine($"API error: {errorMsg}");
        }
        else
        {
            string assistantData = root.GetProperty("data").GetString();
            string threadId = root.GetProperty("threadId").GetString();
            Console.WriteLine($"Assistant response: {assistantData}");
            Console.WriteLine($"Thread ID: {threadId}");
            // Store threadId for the next request if continuing the conversation
        }
    }
    else
    {
        Console.WriteLine($"HTTP Error: {response.StatusCode}");
    }
}
catch (Exception ex)
{
    Console.WriteLine("Request failed: " + ex.Message);
}
```

**Explanation:** This C# snippet uses `HttpClient` to post a JSON payload to the `/meg-chat` endpoint. The request body is created as an anonymous object and serialized to JSON. We set the `Content-Type: application/json` header by using `StringContent`. After sending the request asynchronously with `PostAsync`, we read the response. We then parse the JSON response: if it contains an `"error"` property, we handle it (e.g., log or display the error). Otherwise, we extract the `data` (assistant's answer) and `threadId`. The `threadId` should be stored (perhaps in session state or a variable) and reused for subsequent calls to maintain the conversation context with the AI. 

---

**Note:** The `threadId` mechanism is crucial for multi-turn conversations. Always use the latest `threadId` returned by the API in the next request’s body so that the assistant can remember prior messages in the conversation ([github.com](https://github.com/joelborellis/shadow-fastapi-meg/raw/refs/heads/main/ShadowFunction/__init__.py#:~:text=response.%20,async%20for)). If you start a new conversation (by sending an empty or new threadId), the assistant will not have the previous context. The API is stateful per `threadId`, but you as the client are responsible for preserving and supplying that ID each time.

Since the function app is built with FastAPI, you can also explore the automatic documentation UI. When the Azure Function is running, navigate to `<your-function-app>.azurewebsites.net/docs` in a browser to view the interactive Swagger UI (or `/redoc` for ReDoc) provided by FastAPI. This interface will list the available endpoint(s) and allow you to test them. The single main operation available is the **POST** `/meg-chat` endpoint as described above ([github.com](https://github.com/joelborellis/shadow-fastapi-meg/raw/refs/heads/main/ShadowFunction/__init__.py#:~:text=%40app.post%28%22%2Fmeg,create%20a%20thread%20ID%20if)).

By following the above details and examples, you should be able to integrate the Shadow MEG chat endpoint into your frontend application quickly and confidently. The API is ready to accept chat queries and return AI-generated responses, enabling a seamless chat experience in your application. Enjoy building your chat UI with Shadow MEG!