# 01 - Foundry Local Core Agent Concepts

This document is the first hands-on Microsoft Agent Framework guide in the series. It is designed so students can copy, paste, run, and then study the architecture afterward.

---

## 1. What this document teaches

By the end of this document, students should understand:

- what Agent Framework is responsible for
- what Foundry Local is responsible for
- why an OpenAI-compatible REST interface is the key interoperability point
- how to structure an agent solution as a **client/server** system
- how to use **.NET 10** and **Aspire** to run the distributed pieces together
- how an `AIAgent` is created from an OpenAI-compatible chat client

This document intentionally focuses on **core concepts and architecture**. A later document should go deeper on tool calling, RAG, and multi-agent orchestration.

---

## 2. Source-checked baseline

This document is based on the following verified patterns:

- Microsoft Agent Framework .NET samples target **`net10.0`**
- Agent Framework supports creating agents from OpenAI chat clients
- The OpenAI .NET SDK supports overriding the endpoint with `OpenAIClientOptions.Endpoint`
- Microsoft publishes a sample showing the OpenAI client pointed at a Microsoft Foundry-hosted endpoint through that endpoint override

That matters because it gives us a clean teaching model:

- **Foundry Local** provides the local runtime and REST API
- **OpenAI-compatible schema** provides the protocol contract
- **Agent Framework** sits above that protocol and turns a chat client into an agent

In practice, this means a Foundry Local endpoint can be used from C# by configuring an OpenAI-compatible client to talk to the local base URL.

---

## 3. The most important mental model

Students often mix these ideas together:

- model
- model runtime
- REST API
- agent
- application

They are not the same thing.

### 3.1 Model runtime

The runtime is the server process that actually hosts the model and answers HTTP requests.

With Foundry Local, that runtime exposes a REST API that follows the OpenAI-compatible schema.

### 3.2 OpenAI-compatible REST interface

This is the protocol boundary. If a runtime implements the expected endpoints and request/response shapes, any language can call it.

That is significant because it means:

- Python is not special here
- C# is not blocked by runtime branding
- your code depends on the **HTTP contract**, not on a language-specific runtime SDK

### 3.3 Agent Framework

Agent Framework does not host the model. It gives you:

- the `AIAgent` abstraction
- orchestration patterns
- instruction handling
- sessions and conversations
- tool integration
- workflow patterns

### 3.4 Your application

Your app decides:

- which runtime endpoint to use
- which model to call
- what instructions to send
- how to expose the agent to users
- what guardrails and telemetry to apply

---

## 4. Why use a client/server design

Even for local-first development, teach students to separate the system into at least two app roles:

- **client**
- **agent server**

### Client responsibilities

- capture user input
- display responses
- manage user interaction flow

### Agent server responsibilities

- create and configure the agent
- connect to Foundry Local
- centralize prompts and policies
- expose a stable API to clients
- own logging, validation, and observability

### Why this is significant

This separation teaches good habits early:

- clients can change without rewriting agent logic
- multiple clients can reuse one server
- secrets and runtime endpoints stay on the server side
- Aspire can manage the distributed services cleanly

---

## 5. Package list to install

These are the packages students must install for the solution in this document.

### 5.1 Agent server packages

```xml
<PackageReference Include="Microsoft.Agents.AI" Version="1.13.0" />
<PackageReference Include="Microsoft.Agents.AI.OpenAI" Version="1.13.0" />
<PackageReference Include="OpenAI" Version="2.12.0" />
```

### 5.2 Aspire AppHost package

```xml
<PackageReference Include="Aspire.Hosting.AppHost" Version="13.4.6" />
```

### Why these packages are significant

- `Microsoft.Agents.AI` provides the core `AIAgent` abstractions
- `Microsoft.Agents.AI.OpenAI` provides the Agent Framework bridge for OpenAI-style clients
- `OpenAI` provides the OpenAI-compatible C# client that can target a custom REST endpoint
- `Aspire.Hosting.AppHost` manages the distributed application locally

---

## 6. Namespaces students should recognize

### 6.1 Agent server

```csharp
using System.ClientModel;
using Microsoft.Agents.AI;
using OpenAI;
```

### 6.2 Web API and client

```csharp
using System.Net.Http.Json;
```

### 6.3 Why this is significant

Students should learn to identify namespaces by responsibility:

- `OpenAI` = REST client/provider access
- `Microsoft.Agents.AI` = agent abstraction
- `System.ClientModel` = credentials used by the client SDK

That makes code reading much easier later.

---

## 7. Full solution walkthrough

This example creates a complete solution with:

- an **agent server**
- a **console client**
- an **Aspire AppHost**
- **ServiceDefaults**

The server talks to a Foundry Local OpenAI-compatible endpoint.

---

## 8. Step 1 - Create the solution

Run these commands from the repository root:

```powershell
dotnet new sln -n AgentFrameworkFoundryLocal

dotnet new web -n AgentFrameworkFoundryLocal.AgentServer -f net10.0
dotnet new console -n AgentFrameworkFoundryLocal.Client -f net10.0
dotnet new aspire-apphost -n AgentFrameworkFoundryLocal.AppHost -f net10.0
dotnet new aspire-servicedefaults -n AgentFrameworkFoundryLocal.ServiceDefaults -f net10.0

dotnet sln AgentFrameworkFoundryLocal.sln add AgentFrameworkFoundryLocal.AgentServer
dotnet sln AgentFrameworkFoundryLocal.sln add AgentFrameworkFoundryLocal.Client
dotnet sln AgentFrameworkFoundryLocal.sln add AgentFrameworkFoundryLocal.AppHost
dotnet sln AgentFrameworkFoundryLocal.sln add AgentFrameworkFoundryLocal.ServiceDefaults
```

### Why this is significant

Students see immediately that an agent system is more than one project. That helps them think in systems, not just single files.

---

## 9. Step 2 - Install packages

```powershell
dotnet add AgentFrameworkFoundryLocal.AgentServer package Microsoft.Agents.AI --version 1.13.0
dotnet add AgentFrameworkFoundryLocal.AgentServer package Microsoft.Agents.AI.OpenAI --version 1.13.0
dotnet add AgentFrameworkFoundryLocal.AgentServer package OpenAI --version 2.12.0
dotnet add AgentFrameworkFoundryLocal.AppHost package Aspire.Hosting.AppHost --version 13.4.6
```

### Why this is significant

Students need to see the two-layer dependency story:

1. OpenAI-compatible client package for HTTP communication
2. Agent Framework package for agent orchestration

That is the architectural core of this course section.

---

## 10. Step 3 - Add project references

```powershell
dotnet add AgentFrameworkFoundryLocal.AgentServer reference AgentFrameworkFoundryLocal.ServiceDefaults
dotnet add AgentFrameworkFoundryLocal.AppHost reference AgentFrameworkFoundryLocal.AgentServer
dotnet add AgentFrameworkFoundryLocal.AppHost reference AgentFrameworkFoundryLocal.Client
```

### Why this is significant

This shows that Aspire does not replace your services. It coordinates them.

---

## 11. Step 4 - Replace the generated files

Use the following complete files.

---

## 12. Agent server code

File: `AgentFrameworkFoundryLocal.AgentServer\Program.cs`

```csharp
using System.ClientModel;
using Microsoft.Agents.AI;
using OpenAI;

var builder = WebApplication.CreateBuilder(args);

builder.AddServiceDefaults();

builder.Services.AddSingleton<AIAgent>(_ =>
{
    var endpoint = builder.Configuration["FoundryLocal:Endpoint"]
        ?? throw new InvalidOperationException("FoundryLocal:Endpoint is required.");

    var apiKey = builder.Configuration["FoundryLocal:ApiKey"] ?? "not-used";
    var model = builder.Configuration["FoundryLocal:Model"]
        ?? throw new InvalidOperationException("FoundryLocal:Model is required.");

    var client = new OpenAIClient(
        credential: new ApiKeyCredential(apiKey),
        options: new OpenAIClientOptions
        {
            Endpoint = new Uri(endpoint)
        });

    return client
        .GetChatClient(model)
        .AsAIAgent(
            name: "FoundryLocalTutor",
            instructions:
                "You are a precise teaching assistant. " +
                "Explain your reasoning step by step, use short paragraphs, " +
                "and give practical examples when useful.");
});

var app = builder.Build();

app.MapDefaultEndpoints();

app.MapPost("/chat", async (ChatRequest request, AIAgent agent) =>
{
    if (string.IsNullOrWhiteSpace(request.Message))
    {
        return Results.BadRequest(new { error = "Message is required." });
    }

    var response = await agent.RunAsync(request.Message);
    return Results.Ok(new ChatResponse(response.ToString() ?? string.Empty));
});

app.Run();

public sealed record ChatRequest(string Message);
public sealed record ChatResponse(string Reply);
```

### Why this is significant

This file shows the exact composition point:

1. read Foundry Local configuration
2. create an OpenAI-compatible client
3. override the endpoint to point at Foundry Local
4. convert the chat client into an `AIAgent`
5. expose the agent through an HTTP endpoint

That is the most important pattern in the document.

---

## 13. Agent server configuration

File: `AgentFrameworkFoundryLocal.AgentServer\appsettings.json`

```json
{
  "FoundryLocal": {
    "Endpoint": "http://localhost:5273/v1",
    "ApiKey": "not-used",
    "Model": "phi-4-mini"
  }
}
```

### Why this is significant

This teaches a critical operational idea:

- the runtime is a configuration concern
- the agent code does not need to change when the endpoint changes

If students later point this same server at another OpenAI-compatible local runtime, they should mostly change configuration, not architecture.

---

## 14. Console client code

File: `AgentFrameworkFoundryLocal.Client\Program.cs`

```csharp
using System.Net.Http.Json;

var serverUrl = Environment.GetEnvironmentVariable("AGENT_SERVER_URL") ?? "http://localhost:5001";

using var http = new HttpClient
{
    BaseAddress = new Uri(serverUrl)
};

Console.WriteLine("Connected to agent server: " + serverUrl);
Console.WriteLine("Type a prompt, or type 'exit' to stop.");

while (true)
{
    Console.Write("> ");
    var input = Console.ReadLine();

    if (string.Equals(input, "exit", StringComparison.OrdinalIgnoreCase))
    {
        break;
    }

    if (string.IsNullOrWhiteSpace(input))
    {
        continue;
    }

    var response = await http.PostAsJsonAsync("/chat", new ChatRequest(input));

    if (!response.IsSuccessStatusCode)
    {
        Console.WriteLine($"Request failed with status {(int)response.StatusCode}.");
        continue;
    }

    var payload = await response.Content.ReadFromJsonAsync<ChatResponse>();
    Console.WriteLine();
    Console.WriteLine("Agent: " + payload?.Reply);
    Console.WriteLine();
}

public sealed record ChatRequest(string Message);
public sealed record ChatResponse(string Reply);
```

### Why this is significant

Students can now see the full client/server split in concrete code.

The client does not know how the model works. It only knows how to call the agent server.

That is good architecture.

---

## 15. Aspire AppHost code

File: `AgentFrameworkFoundryLocal.AppHost\Program.cs`

```csharp
var builder = DistributedApplication.CreateBuilder(args);

var agentServer = builder.AddProject<Projects.AgentFrameworkFoundryLocal_AgentServer>("agentserver");

builder.AddProject<Projects.AgentFrameworkFoundryLocal_Client>("client")
    .WithReference(agentServer)
    .WaitFor(agentServer);

builder.Build().Run();
```

### Why this is significant

This is the first place students see Aspire's role clearly:

- it is not the model runtime
- it is not Agent Framework
- it is the local distributed application orchestrator

Aspire helps you manage the moving parts around the agent system.

---

## 16. ServiceDefaults

Use the generated `ServiceDefaults` project from the template and keep it referenced by the server.

### Why this is significant

Students do not need to understand every OpenTelemetry or health-check detail yet. They do need to understand that modern distributed apps should start with shared defaults for:

- service discovery
- telemetry
- resilience
- health endpoints

Aspire gives that structure early.

---

## 17. Step 5 - Run Foundry Local

Start Foundry Local so it exposes its OpenAI-compatible REST API.

Your exact startup command may vary depending on your local environment, but the application expects a base URL like:

```text
http://localhost:5273/v1
```

If your local endpoint differs, update `appsettings.json`.

### Why this is significant

Students should understand that the application does not talk to "Foundry Local the concept." It talks to a concrete HTTP endpoint.

That mindset makes troubleshooting far easier.

---

## 18. Step 6 - Run the solution

```powershell
dotnet run --project AgentFrameworkFoundryLocal.AppHost
```

Then use the console client to send prompts to the agent server.

Example prompts:

- `Explain embeddings like I am new to AI.`
- `What is the difference between a model, runtime, and framework?`
- `Why is a client/server split useful for agents?`

---

## 19. What the student should notice while running it

### 19.1 The client is simple

That is intentional. Clients should stay thin.

### 19.2 The server owns the agent definition

That is where instructions, runtime selection, policies, and later tool registration belong.

### 19.3 The runtime is replaceable

As long as the endpoint remains OpenAI-compatible, the architecture stays stable.

### 19.4 Aspire manages the distributed app

This becomes more valuable as soon as you add:

- a web UI
- a vector store
- a tool service
- evaluation infrastructure

---

## 20. Core Agent Framework concepts in this solution

### 20.1 `AIAgent`

This is the main framework abstraction. It gives your application a stable way to run an agent regardless of the underlying provider.

### 20.2 `GetChatClient(model)`

This chooses the model-facing client for the OpenAI-compatible chat surface.

### 20.3 `AsAIAgent(...)`

This is the conversion point where a model client becomes an agent abstraction.

That is one of the most important lines of code in the entire document:

```csharp
client.GetChatClient(model).AsAIAgent(...)
```

### 20.4 Instructions

Instructions define the baseline behavior of the agent. They are not just text decoration. They strongly shape the answers the model gives.

### 20.5 Sessions and state

This example is intentionally single-turn so students can focus on the architecture first. Multi-turn sessions should come next.

---

## 21. Why this document uses chat completion first

For OpenAI-compatible local runtimes, chat-style compatibility is often the clearest starting point.

That makes it ideal for a first document because students can focus on:

- architecture
- endpoint configuration
- agent abstraction
- distributed app structure

without adding unnecessary complexity too early.

---

## 22. Common mistakes to warn students about

### Mistake 1 - Confusing the runtime with the framework

Foundry Local runs the model. Agent Framework wraps model access in agent abstractions.

### Mistake 2 - Hardcoding runtime details everywhere

Keep endpoint and model in configuration.

### Mistake 3 - Putting prompts only in the client

The server should own the authoritative instructions.

### Mistake 4 - Skipping the client/server split

It may feel faster at first, but it makes scaling, testing, and policy enforcement much harder.

---

## 23. Summary

This document established the most important idea in the course section:

> Foundry Local is the local runtime, the OpenAI-compatible REST interface is the contract, and Agent Framework is the orchestration layer that turns that contract into an agent-based application.

Students should leave this document understanding:

- why OpenAI-compatible REST matters
- why .NET 10 works well for this pattern
- why Agent Framework belongs on the server side
- why Aspire is useful once the system becomes distributed

---

## 24. What should come next

The next document should build on this exact solution and add:

- tool calling
- function schemas
- validation
- explicit server-side orchestration rules

That is the natural next step after students understand the runtime/framework/application split.
