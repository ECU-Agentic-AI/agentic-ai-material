# 02 - Building Agent Tools in C# for Foundry Local

This document builds directly on `01-foundry-local-core-agent-concepts.md`.

Document 01 established the architecture:

- Foundry Local provides the local OpenAI-compatible REST endpoint
- the OpenAI client talks to that endpoint
- Agent Framework turns the chat client into an `AIAgent`
- Aspire manages the distributed services

This document adds the next critical capability:

> giving the agent tools it can call from your C# code.

---

## 1. What this document teaches

By the end of this document, students should understand:

- what a function tool is
- how Agent Framework exposes a C# method as a tool
- why tool descriptions and parameter descriptions matter
- how to organize tools in classes instead of random static methods
- how to inject dependencies into tool classes
- how to validate tool inputs and return explicit results
- how to extend the solution from Document 01 with production-shaped tool organization

---

## 2. Why tools matter

Without tools, an agent can only generate text from its model knowledge and context window.

With tools, an agent can:

- look up live or local data
- perform deterministic calculations
- call your business logic
- use internal services
- act as a real application component instead of only a chatbot

### Why this is significant

This is the moment where students stop thinking of the model as “the application” and start seeing the model as one part of a larger system.

That shift is essential for serious agent engineering.

---

## 3. Core mental model for function tools

In Agent Framework, a function tool is just a C# method the model is allowed to call.

The usual flow looks like this:

1. The user asks a question.
2. The model decides whether a tool is needed.
3. Agent Framework exposes the tool schema to the model.
4. The model selects a tool and proposes arguments.
5. Your server-side code runs the actual method.
6. The result goes back into the agent loop.
7. The model writes the final answer.

### Why this is significant

The model does **not** execute your code directly.

Your application remains in control:

- you decide which methods are tools
- you decide what they do
- you decide what validation happens
- you decide what the model is allowed to see

That is one of the most important safety and architecture boundaries in agent systems.

---

## 4. Source-backed tool pattern

The current Agent Framework guidance for C# function tools uses:

- `AIFunctionFactory.Create(...)`
- `System.ComponentModel.DescriptionAttribute`

This is the pattern we will use here because it is the clearest way to teach tool creation in C#.

### Why this is significant

Students need a pattern that is:

- easy to read
- easy to extend
- consistent with current framework documentation

---

## 5. Packages to install

If students completed Document 01, no new packages are required.

These are still the required package versions for this solution:

```xml
<PackageReference Include="Microsoft.Agents.AI" Version="1.13.0" />
<PackageReference Include="Microsoft.Agents.AI.OpenAI" Version="1.13.0" />
<PackageReference Include="OpenAI" Version="2.12.0" />
<PackageReference Include="Aspire.Hosting.AppHost" Version="13.4.6" />
```

### Why this is significant

Students should see that tools are a **framework capability**, not a separate tool package they bolt on later.

---

## 6. Namespaces for tool work

These namespaces are the important ones for this document.

### 6.1 Agent server and provider

```csharp
using System.ClientModel;
using Microsoft.Agents.AI;
using Microsoft.Extensions.AI;
using OpenAI;
```

### 6.2 Tool metadata

```csharp
using System.ComponentModel;
```

### 6.3 Client

```csharp
using System.Net.Http.Json;
```

### Why this is significant

Students should associate:

- `Microsoft.Extensions.AI` with tool construction
- `System.ComponentModel` with descriptions and schema hints

Those two details are especially important when debugging tool selection behavior.

---

## 7. Tool design rules students should follow

Before showing code, teach these design rules explicitly.

### Rule 1 - Keep tools narrow

A tool should do one clear thing.

Good:

- `GetCampusWeather`
- `AddNumbers`
- `GetOfficeHours`

Bad:

- `HandleStudentQuestion`
- `DoEverything`
- `FindStuffAndMaybeSummarize`

### Rule 2 - Make the signature obvious

If the model cannot easily infer what arguments to provide, it will choose badly.

### Rule 3 - Add descriptions

Descriptions are not optional decoration. They are part of the model’s planning surface.

### Rule 4 - Validate inside the tool

The model can hallucinate arguments or pass incomplete data.

### Rule 5 - Return explicit, useful results

Do not make tools return vague text if the result should be structured or concrete.

### Why this is significant

A badly designed tool is often worse than no tool. It increases agent unpredictability.

---

## 8. Full solution walkthrough

This solution keeps the same project structure as Document 01:

- `AgentFrameworkFoundryLocal.AgentServer`
- `AgentFrameworkFoundryLocal.Client`
- `AgentFrameworkFoundryLocal.AppHost`
- `AgentFrameworkFoundryLocal.ServiceDefaults`

The difference is that the server now registers and exposes tools.

---

## 9. Step 1 - Keep the same solution from Document 01

If students already created the solution from Document 01, they should keep it.

If they are starting fresh, they can use the same creation steps from Document 01 first.

### Why this is significant

This reinforces that tools are an extension of the same architecture, not a new architecture.

---

## 10. Step 2 - Add tool support files

We will add three files to the agent server project:

- `ToolDependencies.cs`
- `WeatherTools.cs`
- `AcademicTools.cs`

### Why this is significant

Students should learn early that tools belong in organized classes, not just stuffed into `Program.cs`.

---

## 11. Tool dependency service

File: `AgentFrameworkFoundryLocal.AgentServer\ToolDependencies.cs`

```csharp
namespace AgentFrameworkFoundryLocal.AgentServer;

public sealed class CampusDataService
{
    private static readonly Dictionary<string, string> OfficeHours = new(StringComparer.OrdinalIgnoreCase)
    {
        ["agentic ai"] = "Office hours for Agentic AI are Tuesday 2:00 PM to 4:00 PM in Science 201.",
        ["distributed systems"] = "Office hours for Distributed Systems are Thursday 1:00 PM to 3:00 PM in Engineering 410.",
        ["cloud computing"] = "Office hours for Cloud Computing are Wednesday 10:00 AM to 12:00 PM in Online Teams Room B."
    };

    public string GetOfficeHours(string courseName)
    {
        if (string.IsNullOrWhiteSpace(courseName))
        {
            return "A course name is required.";
        }

        return OfficeHours.TryGetValue(courseName.Trim(), out var hours)
            ? hours
            : $"No office hours were found for '{courseName}'.";
    }
}
```

### Why this is significant

This file introduces the idea that tools can depend on normal application services.

That is important because serious agent systems rarely call raw static methods forever. They usually call services with real dependencies.

---

## 12. Weather tool class

File: `AgentFrameworkFoundryLocal.AgentServer\WeatherTools.cs`

```csharp
using System.ComponentModel;

namespace AgentFrameworkFoundryLocal.AgentServer;

public sealed class WeatherTools
{
    [Description("Get a simple weather summary for a campus or city.")]
    public string GetCampusWeather(
        [Description("The city or campus location to get weather for.")] string location,
        [Description("Temperature unit to use. Valid values are celsius or fahrenheit.")] string unit = "celsius")
    {
        if (string.IsNullOrWhiteSpace(location))
        {
            return "A location is required.";
        }

        var normalizedUnit = unit.Trim().ToLowerInvariant();
        if (normalizedUnit is not ("celsius" or "fahrenheit"))
        {
            return "The unit must be either 'celsius' or 'fahrenheit'.";
        }

        var temperature = normalizedUnit == "fahrenheit" ? "72 F" : "22 C";
        return $"The weather in {location.Trim()} is sunny with a temperature of {temperature}.";
    }
}
```

### Why this is significant

This is the first example of a tool that:

- has a clear purpose
- uses descriptions
- validates inputs
- returns a direct result

Students should notice that the method itself is ordinary C# code. The framework layer is what turns it into a tool.

---

## 13. Academic tool class

File: `AgentFrameworkFoundryLocal.AgentServer\AcademicTools.cs`

```csharp
using System.ComponentModel;

namespace AgentFrameworkFoundryLocal.AgentServer;

public sealed class AcademicTools
{
    private readonly CampusDataService _campusDataService;

    public AcademicTools(CampusDataService campusDataService)
    {
        _campusDataService = campusDataService;
    }

    [Description("Get instructor office hours for a specific course.")]
    public string GetOfficeHours(
        [Description("The course name, such as Agentic AI or Cloud Computing.")] string courseName)
    {
        return _campusDataService.GetOfficeHours(courseName);
    }

    [Description("Add two numbers and return the result.")]
    public string AddNumbers(
        [Description("The first number.")] decimal left,
        [Description("The second number.")] decimal right)
    {
        return $"{left} + {right} = {left + right}";
    }
}
```

### Why this is significant

This class teaches two important patterns at once:

1. **dependency injection**
2. **multiple related tools in one class**

That is a much more realistic teaching pattern than placing every tool in a giant static utility file.

---

## 14. Updated agent server

File: `AgentFrameworkFoundryLocal.AgentServer\Program.cs`

```csharp
using System.ClientModel;
using AgentFrameworkFoundryLocal.AgentServer;
using Microsoft.Agents.AI;
using Microsoft.Extensions.AI;
using OpenAI;

var builder = WebApplication.CreateBuilder(args);

builder.AddServiceDefaults();

builder.Services.AddSingleton<CampusDataService>();
builder.Services.AddSingleton<WeatherTools>();
builder.Services.AddSingleton<AcademicTools>();

builder.Services.AddSingleton<AIAgent>(sp =>
{
    var endpoint = builder.Configuration["FoundryLocal:Endpoint"]
        ?? throw new InvalidOperationException("FoundryLocal:Endpoint is required.");

    var apiKey = builder.Configuration["FoundryLocal:ApiKey"] ?? "not-used";
    var model = builder.Configuration["FoundryLocal:Model"]
        ?? throw new InvalidOperationException("FoundryLocal:Model is required.");

    var weatherTools = sp.GetRequiredService<WeatherTools>();
    var academicTools = sp.GetRequiredService<AcademicTools>();

    var tools = new[]
    {
        AIFunctionFactory.Create(weatherTools.GetCampusWeather),
        AIFunctionFactory.Create(academicTools.GetOfficeHours),
        AIFunctionFactory.Create(academicTools.AddNumbers)
    };

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
                "Use tools when the user asks for office hours, weather, or arithmetic. " +
                "If a tool result gives the answer, use that result clearly and directly.",
            tools: tools);
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

This file shows the exact point where normal C# methods become model-callable tools:

```csharp
AIFunctionFactory.Create(weatherTools.GetCampusWeather)
```

That line is the bridge between:

- your deterministic application logic
- the model’s planning behavior

It is one of the most important patterns in the course.

---

## 15. Agent server configuration

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

Nothing about the tool architecture changes the runtime contract. The runtime is still a configurable OpenAI-compatible endpoint.

That consistency is useful for teaching.

---

## 16. Console client

The client can stay the same, but it is included here so the walkthrough is self-contained.

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

---

## 17. Aspire AppHost

The AppHost can also stay the same.

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

Students should notice that tool support changes the **server internals**, not the distributed system topology.

That is a useful architecture lesson.

---

## 18. Step 3 - Run the solution

Start Foundry Local so the OpenAI-compatible endpoint is available, then run:

```powershell
dotnet run --project AgentFrameworkFoundryLocal.AppHost
```

Try these prompts:

- `What are the office hours for Agentic AI?`
- `What is the weather in Greenville in fahrenheit?`
- `Add 14.5 and 3.25`
- `Explain why tools are useful in agent systems.`

### What students should observe

The agent should answer general questions in natural language, but when the question maps to a registered tool, it should use that tool instead of guessing.

That is exactly the behavior we want.

---

## 19. Why descriptions matter so much

These descriptions are not just for humans:

```csharp
[Description("Get instructor office hours for a specific course.")]
public string GetOfficeHours(...)
```

The model uses that description when deciding:

- whether to call the tool
- which tool to choose
- what arguments to provide

### Why this is significant

If descriptions are vague, the model’s tool selection becomes worse.

This is one of the most common reasons tool-using agents feel unreliable.

---

## 20. Why input validation still matters

Even though the framework gives the model a tool schema, the model can still produce imperfect inputs.

Examples:

- empty strings
- wrong units
- incorrect course names
- malformed assumptions

That is why the tool code still validates:

```csharp
if (string.IsNullOrWhiteSpace(location))
{
    return "A location is required.";
}
```

### Why this is significant

Students must understand a critical rule:

> the model is not a trusted caller.

Treat model-supplied arguments as untrusted input.

---

## 21. Why classes are better than many loose methods

Class-based tools are better for teaching and real systems because they support:

- dependency injection
- internal shared state
- grouping related capabilities
- cleaner project structure

### Why this is significant

Once students begin building real tools against databases, APIs, and internal services, this structure becomes necessary.

---

## 22. Common mistakes students will make

### Mistake 1 - Writing tool names and descriptions too vaguely

If the tool purpose is unclear, the model will choose poorly.

### Mistake 2 - Skipping validation because “the model knows”

It does not always know.

### Mistake 3 - Returning fluffy prose from tools

Tools should return clear, useful results.

### Mistake 4 - Hiding all tools inside `Program.cs`

That works for demos but scales badly.

### Mistake 5 - Forgetting that tools are server-side code

The client does not execute tools. The server does.

---

## 23. A simple checklist for good tool design

Before adding a tool, ask:

- Is the tool narrowly scoped?
- Does the tool name clearly describe the action?
- Are the parameter descriptions specific?
- Do I validate inputs?
- Does the tool return a useful result?
- Should the tool live in a class with dependencies?

If the answer to any of those is “no,” improve the tool before shipping it.

---

## 24. Summary

This document introduced the core tool-building pattern for Agent Framework in C#:

1. write normal C# methods
2. describe them clearly
3. validate their inputs
4. register them with `AIFunctionFactory.Create(...)`
5. pass them into `AsAIAgent(...)`

That is the key bridge from local model inference to real application behavior.

---

## 25. What should come next

The next natural document is:

- memory
- retrieval
- grounding
- local RAG pipelines

That will let students combine:

- a local runtime
- an agent abstraction
- tool calling
- external knowledge

which is where agent applications start becoming genuinely useful.
