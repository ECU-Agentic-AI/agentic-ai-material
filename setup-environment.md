
# Installation Guide: Ollama + Foundry Local + .NET SDK 10  
**Platforms:** Windows 11 & macOS (Apple Silicon)  
**Assumption:** Nothing is installed yet — including .NET, Ollama, or Foundry Local.  Also, I'm assuming that MacOS already has cURL installed.

## Table of Contents

- [1. Windows Installation](#1-windows-installation-gpuaccelerated)
    - [1.0 Install curl](#10-install-curl)
    - [1.1 Install .NET SDK 10](#11-install-net-sdk-10)
    - [1.2 Install Foundry Local](#12-install-foundry-local-windows)
    - [1.3 Install Ollama](#13-install-ollama-windows)
    - [1.4 Configure .NET 10 App](#14-configure-net-10-app-to-use-both-backends)
- [2. macOS Installation](#2-macos-installation-aneaccelerated)
    - [2.1 Install .NET SDK 10](#21-install-net-sdk-10)
    - [2.2 Install Foundry Local](#22-install-foundry-local-macos)
    - [2.3 Install Ollama](#23-install-ollama-macos)
    - [2.4 Configure .NET 10 App](#24-configure-net-10-app-same-code-as-windows)
- [3. Using REST Client in VS Code](#3-using-rest-client-in-vs-code)


---

# 1. WINDOWS INSTALLATION (GPU‑Accelerated)


## 1.0 Install curl

Install using winget

```powershell
winget install curl.curl
```

Confirm installation

```powershell
curl --version
```

## 1.1 Install .NET SDK 10
Download and install:

- https://dotnet.microsoft.com/en-us/download/dotnet/10.0

Verify:

```powershell
dotnet --version
```

You should see:

```
10.x.x
```

---

## 1.2 Install Foundry Local (Windows)

> Note: Foundry Local is a native AI solution that is embedded into your application.  The following install is to provide Ollama-like REST functionality, but it is not required for native AI embedding.

Download the Windows installer:

- https://www.foundrylocal.ai

Run installer → accept defaults.

Verify installation:

```powershell
foundry --version
```

Pull a model:

```powershell
foundry model download qwen2.5-0.5b
```

View downloaded models

```powershell
foundry cache list
```

Test:

```powershell
foundry model run qwen2.5-0.5b
```

```
Interactive mode, please enter your prompt
> what is 12 squared?
```

```
🧠 Thinking...
🤖 12 squared is calculated as 12^2, which equals:

12^2 = 144

This means that the square of 12 is 144.
```

Test REST API:

```powershell
foundry service status
```

Start the local server:

```powershell
foundry service start
```

Default endpoint:

```
No default endpoint.  Need to pay attention to console output for port or explicitly set the endpoint configuration if you want to use REST API.
```


```bash
curl http://127.0.0.1:54070/, -d "{ \"model\": \"qwen2.5-0.5b-instruct-openvino-npu:5\", \"messages\": [ { \"role\": \"system\", \"content\": \"You are a NFL football analyst\" }, { \"role\": \"user\", \"content\": \"Who will win the super bowl in 2027\" } ], \"response_format\": { \"type\": \"json_object\" } }"
```
```powershell
curl http://127.0.0.1:54070/ -d "{ ""model"": ""qwen2.5-0.5b-instruct-openvino-npu:5"", ""messages"": [ { ""role"": ""system"", ""content"": ""You are a NFL football analyst"" }, { ""role"": ""user"", ""content"": ""Who will win the super bowl in 2027"" } ], ""response_format"": { ""type"": ""json_object"" } }"
```

> Note: You must specify the specific model variant when using Foundry Local as a REST API.


Example response:
```json
{
    "model": "qwen2.5-0.5b-instruct-openvino-npu:5",
    "choices": [
        {
            "delta": {
                "role": "assistant",
                "content": "It is not possible to predict who will win the Super Bowl in any given year ...",
                "tool_calls": [
                ]
            },
            "message": {
                "role": "assistant",
                "content": "It is not possible to predict who will win the Super Bowl in any given year ...",
                "tool_calls": [
                ]
            },
            "index": 0,
            "finish_reason": "stop"
        }
    ],
    "created": 1782535384,
    "CreatedAt": "2026-06-27T04:43:04+00:00",
    "id": "chat.id.6",
    "IsDelta": false,
    "Successful": true,
    "HttpStatusCode": 0,
    "object": "chat.completion"
}

```
---

## 1.3 Install Ollama (Windows)
Download:

- https://ollama.com/download/windows

Install → restart terminal.

Verify:

```powershell
ollama --version
```

Start Ollama service:

```powershell
ollama serve
```

Default endpoint:

```
http://localhost:11434/v1
```

Pull a model:

```powershell
ollama pull llama3
```

Test:

```bash
curl http://localhost:11434/api/generate -d "{ \"model\": \"llama3\", \"prompt\": \"Hello\" }"
```
```powershell
curl http://localhost:11434/api/generate -d "{ ""model"": ""llama3"", ""prompt"": ""Hello"" }"
```
The REST calls above are not OpenAI compliant.  You must use OpenAI compliant calls when using Microsoft Agent Framework.

Here are examples of the same prompt using OpenAI compliant schema.

```bash
curl http://localhost:11434/v1/chat/completions `
  -H "Content-Type: application/json" `
  -d "{\"model\":\"llama3\",\"messages\":[{\"role\":\"user\",\"content\":\"Explain vector databases.\"}]}"
```

```powershell
curl http://localhost:11434/v1/chat/completions `
  -H "Content-Type: application/json" `
  -d "{""model"":""llama3"",""messages"":[{""role"":""user"",""content"":""Explain vector databases.""}]}"
```

> Note: You can use the model alias with Ollama.

---

## 1.4 Configure .NET 10 App to Use Both Backends

Create project:

```powershell
dotnet new console -n LocalAI
cd LocalAI
```

Add OpenAI client:

```powershell
dotnet add package Microsoft.Extensions.AI.OpenAI
```

Open project is VS Code
```powershell
code .
```

Replace contents of `Program.cs` with the code below:

```csharp
using OpenAI;

var ollama = new OpenAIClient(
    new System.ClientModel.ApiKeyCredential("local"),
    new OpenAIClientOptions()
    {
        Endpoint = new Uri("http://localhost:11434/v1")
    });

var foundry = new OpenAIClient(
    new System.ClientModel.ApiKeyCredential("local"),
    new OpenAIClientOptions()
    {
        Endpoint = new Uri("http://localhost:22334/v1")
    });

Console.WriteLine("Ollama:");
var ollamaResponse = await ollama
                            .GetChatClient("llama3")
                            .CompleteChatAsync("Say hello from Ollama.");

Console.WriteLine(ollamaResponse.Value.Content[0].Text);

Console.WriteLine("Foundry Local:");

var foundryResponse = await foundry
                            .GetChatClient("qwen2.5-0.5b-instruct-openvino-npu:5")
                            .CompleteChatAsync("Say hello from Foundry Local.");
                            
Console.WriteLine(foundryResponse.Value.Content[0].Text);
```

Run:

```powershell
dotnet run
```

---


# 2. MACOS INSTALLATION (ANE‑Accelerated)


## 2.1 Install .NET SDK 10
Download:

- https://dotnet.microsoft.com/en-us/download/dotnet/10.0

Verify:

```bash
dotnet --version
```

---

## 2.2 Install Foundry Local (macOS)

> Note: Foundry Local is a native AI solution that is embedded into your application.  The following install is to provide Ollama-like REST functionality, but it is not required for native AI embedding.

Download macOS installer:

- https://www.foundrylocal.ai

Install → allow permissions if prompted.

Verify:

```bash
foundry --version
```
Pull a model:

```bash
foundry model download qwen2.5-0.5b
```

View downloaded models

```bash
foundry cache list
```

Test:

```bash
foundry model run qwen2.5-0.5b
```

```
Interactive mode, please enter your prompt
> what is 12 squared?
```

```
🧠 Thinking...
🤖 12 squared is calculated as 12^2, which equals:

12^2 = 144

This means that the square of 12 is 144.
```

Test REST API:

```bash
foundry service status
```

Start the local server:

```bash
foundry service start
```

Default endpoint:

```
No default endpoint.  Need to pay attention to console output for port or explicitly set the endpoint configuration if you want to use REST API.
```

```bash
curl http://127.0.0.1:54070/, -d "{ \"model\": \"qwen2.5-0.5b-instruct-openvino-npu:5\", \"messages\": [ { \"role\": \"system\", \"content\": \"You are a NFL football analyst\" }, { \"role\": \"user\", \"content\": \"Who will win the super bowl in 2027\" } ], \"response_format\": { \"type\": \"json_object\" } }"
```

> Note: You must specify the specific model variant when using Foundry Local as a REST API.


Example response:
```json
{
    "model": "qwen2.5-0.5b-instruct-openvino-npu:5",
    "choices": [
        {
            "delta": {
                "role": "assistant",
                "content": "It is not possible to predict who will win the Super Bowl in any given year ...",
                "tool_calls": [
                ]
            },
            "message": {
                "role": "assistant",
                "content": "It is not possible to predict who will win the Super Bowl in any given year ...",
                "tool_calls": [
                ]
            },
            "index": 0,
            "finish_reason": "stop"
        }
    ],
    "created": 1782535384,
    "CreatedAt": "2026-06-27T04:43:04+00:00",
    "id": "chat.id.6",
    "IsDelta": false,
    "Successful": true,
    "HttpStatusCode": 0,
    "object": "chat.completion"
}

```

---

## 2.3 Install Ollama (macOS)
Download:

- https://ollama.com/download/mac

Install → macOS will prompt for security approval.

Verify:

```bash
ollama --version
```

Start service:

```bash
ollama serve
```

Pull a model:

```bash
ollama pull llama3
```

Test:

```bash
curl http://localhost:11434/api/generate -d "{ \"model\": \"llama3\", \"prompt\": \"Hello\" }"
```

The REST call above is not OpenAI compliant.  You must use OpenAI compliant calls when using Microsoft Agent Framework.

Here are examples of the same prompt using OpenAI compliant schema.

```bash
curl http://localhost:11434/v1/chat/completions `
  -H "Content-Type: application/json" `
  -d "{\"model\":\"llama3\",\"messages\":[{\"role\":\"user\",\"content\":\"Explain vector databases.\"}]}"
```

> Note: You can use the model alias with Ollama.

---

## 2.4 Configure .NET 10 App (Same Code as Windows)

Create project:

```bash
dotnet new console -n LocalAI
cd LocalAI
```

Add OpenAI client:

```bash
dotnet add package Microsoft.Extensions.AI.OpenAI
```

Open project is VS Code
```bash
code .
```

Replace contents of `Program.cs` with the code below:

```csharp
using OpenAI;

var ollama = new OpenAIClient(
    new System.ClientModel.ApiKeyCredential("local"),
    new OpenAIClientOptions()
    {
        Endpoint = new Uri("http://localhost:11434/v1")
    });

var foundry = new OpenAIClient(
    new System.ClientModel.ApiKeyCredential("local"),
    new OpenAIClientOptions()
    {
        Endpoint = new Uri("http://localhost:22334/v1")
    });

Console.WriteLine("Ollama:");
var ollamaResponse = await ollama
                            .GetChatClient("llama3")
                            .CompleteChatAsync("Say hello from Ollama.");

Console.WriteLine(ollamaResponse.Value.Content[0].Text);

Console.WriteLine("Foundry Local:");

var foundryResponse = await foundry
                            .GetChatClient("qwen2.5-0.5b-instruct-openvino-npu:5")
                            .CompleteChatAsync("Say hello from Foundry Local.");
                            
Console.WriteLine(foundryResponse.Value.Content[0].Text);
```

Run:

```powershell
dotnet run
```

---

---


# 3. USING REST CLIENT IN VS CODE


This guide includes REST API checks for Foundry Local and Ollama. You can run those requests from the command line with `curl`, or directly inside VS Code with the REST Client extension.

Install the extension:

1. Open VS Code.
2. Select **Extensions**.
3. Search for **REST Client**.
4. Install **REST Client** by Huachao Mao.

You can also install it from the command line:

```powershell
code --install-extension humao.rest-client
```

After installing it, open `test-rest-harness.http`. Each request has a **Send Request** link above it. Start the local service you want to test, then select **Send Request** to call the endpoint and view the response inside VS Code.

A `.http` file is a plain text request file that lets you save and run repeatable HTTP calls. The REST Client extension reads the file and turns each request into something you can execute from VS Code.

Define variables with `@name = value`:

```http
@ollama-host = http://localhost:11434/v1
@model = llama3
```

Use variables with double curly braces:

```http
POST {{ollama-host}}/chat/completions
```

Separate multiple requests with `###`:

```http
###
POST {{ollama-host}}/chat/completions
```

Put headers directly under the request line. Leave one blank line between the headers and the body:

```http
POST {{ollama-host}}/chat/completions
Content-Type: application/json

{
    "model": "{{model}}",
    "messages": [
        {"role": "user", "content": "Hello!"}
    ]
}
```

That blank line matters because it tells REST Client where the headers stop and where the request body starts.

The REST harness defines these local endpoints:

```http
@foundry-host = http://localhost:22334/v1
@ollama-host = http://localhost:11434/v1
```

If Foundry Local starts on a different port, update `@foundry-host` in `test-rest-harness.http` to match the port shown in the Foundry service output.