# Hands-on Tutorial: IntroToDistributedAI (Copy/Paste Friendly)

This document is a **step-by-step lab** 

---

## 0) Prerequisites

Install/verify:

1. .NET 10 SDK
  - `dotnet --version`
2. Aspire CLI
  - `aspire --version`
3. Aspire Project Templates
  - `dotnet new list --tag='aspire' --columns language`
  - templates are listed alphabetically, look for Aspire templates

```
Template Name                                        Short Name                   Language
---------------------------------------------------  ---------------------------  --------
.NET MAUI for Aspire Service Defaults                maui-aspire-servicedefaults  [C#]
Aspire AppHost                                       aspire-apphost               [C#]
Aspire Empty App                                     aspire                       [C#]
Aspire Service Defaults                              aspire-servicedefaults       [C#]
Aspire Single-File App Host                          aspire-apphost-singlefile    [C#]
Aspire Starter App (ASP.NET Core/Blazor)             aspire-starter               [C#]
Aspire Starter App (ASP.NET Core/React, C# AppHost)  aspire-ts-cs-starter         [C#]
Aspire Test Project (MSTest)                         aspire-mstest                [C#]
Aspire Test Project (NUnit)                          aspire-nunit                 [C#]
Aspire Test Project (xUnit)                          aspire-xunit                 [C#]
Fluent Aspire Starter App                            fluentaspire-starter         [C#]
```
4. Docker Desktop
  - `docker --version`
5. Foundry Local CLI
  - `foundry --version` 
6. Trust Aspire Certs
  - `aspire certs trust` 

---

## Accept and clone hands-on repo

https://classroom.github.com/a/Y64-vqmz

## Create the solution

- Open the hands-on cloned folder in VS Code
- Execute the following commands from the root of the cloned folder

### Create the solution file
```bash
# Create an empty solution file
dotnet new sln -n IntroToDistributedAI
```

### Create the projects that will be used in the solution
```bash
# Create Aspire AppHost and Service Defaults projects 
dotnet new aspire-apphost -n IntroToDistributedAI.AppHost -f net10.0
dotnet new aspire-servicedefaults -n IntroToDistributedAI.ServiceDefaults -f net10.0

# Create worker service project for the Watcher Service
dotnet new worker -n IntroToDistributedAI.WatcherService -f net10.0

# Create minimal web API project for the Tutor Agent
dotnet new web -n IntroToDistributedAI.TutorAgent -f net10.0

# Create console app project for the Console Client 
dotnet new console -n IntroToDistributedAI.ConsoleClient -f net10.0
```

### Add all projects to the solution file
```bash
dotnet sln IntroToDistributedAI.slnx add IntroToDistributedAI.TutorAgent
dotnet sln IntroToDistributedAI.slnx add IntroToDistributedAI.ConsoleClient
dotnet sln IntroToDistributedAI.slnx add IntroToDistributedAI.AppHost
dotnet sln IntroToDistributedAI.slnx add IntroToDistributedAI.ServiceDefaults
dotnet sln IntroToDistributedAI.slnx add IntroToDistributedAI.WatcherService
```
>Note: Some shells require the use of ./ to tell the shell to look in the current directory.  If you received an error message "file not found", etc, then prefix solution and project folder names with ./   e.g.  dotnet sln ./IntroToDistributedAI.slnx add ./IntroToDistributedAI.TutorAgent

### Add project references

```bash
dotnet add IntroToDistributedAI.TutorAgent reference IntroToDistributedAI.ServiceDefaults
dotnet add IntroToDistributedAI.ConsoleClient reference IntroToDistributedAI.ServiceDefaults
dotnet add IntroToDistributedAI.AppHost reference IntroToDistributedAI.TutorAgent
dotnet add IntroToDistributedAI.AppHost reference IntroToDistributedAI.ConsoleClient
dotnet add IntroToDistributedAI.AppHost reference IntroToDistributedAI.WatcherService
```
>Note: Same ./ prefix info as above

### Add missing packages to projects

```bash
dotnet add IntroToDistributedAI.TutorAgent package Microsoft.Agents.AI 
dotnet add IntroToDistributedAI.TutorAgent package Microsoft.Extensions.AI
dotnet add IntroToDistributedAI.TutorAgent package Microsoft.Agents.AI.OpenAI 
dotnet add IntroToDistributedAI.TutorAgent package OpenAI 
dotnet add IntroToDistributedAI.AppHost package Aspire.Hosting.AppHost 

```

### Confirm that solution compiles

From the solution root folder - folder with the .slnx file

```bash
dotnet build
```

Successful build should resemble the following output
```
Restore complete (0.8s)
  IntroToDistributedAI.ServiceDefaults net10.0 succeeded (1.6s) → IntroToDistributedAI.ServiceDefaults\bin\Debug\net10.0\IntroToDistributedAI.ServiceDefaults.dll
  IntroToDistributedAI.WatcherService net10.0 succeeded (1.7s) → IntroToDistributedAI.WatcherService\bin\Debug\net10.0\IntroToDistributedAI.WatcherService.dll
  IntroToDistributedAI.ConsoleClient net10.0 succeeded (0.8s) → IntroToDistributedAI.ConsoleClient\bin\Debug\net10.0\IntroToDistributedAI.ConsoleClient.dll
  IntroToDistributedAI.TutorAgent net10.0 succeeded (1.4s) → IntroToDistributedAI.TutorAgent\bin\Debug\net10.0\IntroToDistributedAI.TutorAgent.dll
  IntroToDistributedAI.AppHost net10.0 succeeded (2.0s) → IntroToDistributedAI.AppHost\bin\Debug\net10.0\IntroToDistributedAI.AppHost.dll

Build succeeded in 6.1s
```

---

## Solution Implementation Steps
1. Wire up projects in AppHost.cs
2. Test Dashboard - should display all projects and inserted configurations
3. Implement Watcher Service
4. Test Watcher Service
5. Implement Tutor Agent
6. Test Tutor Agent with .http file
7. Implement Console Client
8. Test Console Client 

## 1. Wire up projects in AppHost

Replace the contents of the `IntroToDistributedAI.AppHost/AppHost.cs` file with the following Aspire Configuration

```csharp

var builder = DistributedApplication.CreateBuilder(args);

var foundryLocal = builder.AddExternalService("foundry", "http://localhost:53535");

var watcherService =builder.AddProject<Projects.IntroToDistributedAI_WatcherService>("watcher")
    .WithReference(foundryLocal)
    .WaitFor(foundryLocal);

var tutorAgent = builder.AddProject<Projects.IntroToDistributedAI_TutorAgent>("tutor")
    .WithExternalHttpEndpoints()
    .WithReference(watcherService)
    .WaitFor(watcherService)
    .WithReference(foundryLocal)
    .WaitFor(foundryLocal);

var consoleClient =builder.AddProject<Projects.IntroToDistributedAI_ConsoleClient>("console")
    .WithReference(tutorAgent)
    .WaitFor(tutorAgent);

builder.Build().Run();
```

> Note: The Projects namespace inside the < > after each .AddProject() call should list all three projects(watcher, tutor, console).  Delete one of them and type Program "dot" and you should see all three projects list.  Select the project you just deleted to fix the code.  


## 2. Test Dashboard - should display all projects and inserted configurations

Confirm solutions builds
```bash
dotnet build
```

Confirm that Aspire solution runs - note we run the solution with aspire cli, not dotnet cli
```bash
aspire run
```

Open the dashboard by pressing CTRL and left-click the Dashboard link in the console output.

Confirm Aspire is passing configuration to projects referencing other projects by clicking the `...` under Actions and selecting `Export .env`

You should see the specific environment variables that were injected because the project referenced another project. e.g. TUTOR_HTTP=http://localhost:5082

Stop the solution from running by selecting the console running aspire and pressing `CTRL+C`

## 3. Implement Watcher Service

The default Worker project includes a Program.cs and a Worker.cs

The Worker file is a Background Service that will continuously run.

We want to create a dedicated service for controlling Foundry Local and then inject it inside the Worker to use.

- Right-click the WatcherService project folder inside VS Code Explorer pane and select [New File...]
- Name the file `FoundryLocalSupervisor.cs`
- Replace the contents of file with the following code.
```csharp
using System.Diagnostics;
using System.Runtime.InteropServices;

public class FoundryLocalSupervisor
{
    private readonly ILogger<FoundryLocalSupervisor> _logger;
    public string _host;
    public int _port;

    public Uri ServiceUri => new Uri($"http://{_host}:{_port}");

    public FoundryLocalSupervisor(ILogger<FoundryLocalSupervisor> logger, IConfiguration configuration)
    {
        _logger = logger;

        var foundryUriString = Environment.GetEnvironmentVariable("FOUNDRY") ?? throw new InvalidOperationException("FOUNDRY environment variable is not set.");
        var foundryUri = new Uri(foundryUriString);

        _host = foundryUri.Host;
        _port = foundryUri.Port;
    }

    public async Task<Uri> StartAsync()
    {
        _logger.LogInformation($"[WatcherService]: Starting Foundry Local service...");

        await RunAsync($"foundry service set --port {_port}");
        var result = await RunAsync("foundry service start");

        _logger.LogInformation($"[WatcherService]: {result}");
        
        return ServiceUri;
    }

    public async Task StopAsync()
    {
        _logger.LogInformation($"[WatcherService]: Stopping Foundry Local service...");

        var result = await RunAsync("foundry service stop");

        _logger.LogInformation($"[WatcherService]: {result}");
    }

    public async Task<bool> IsRunningAsync()
    {
        var status = await RunAsync("foundry service status");
        return status.Contains("service is running", StringComparison.OrdinalIgnoreCase);
    }

    public async Task<Uri> EnsureRunningAsync()
    {
        if (!await IsRunningAsync())
        {
            _logger.LogWarning($"[WatcherService]: Foundry Local not running. Restarting...");
    
            var result = await RunAsync("foundry service restart");
            
            _logger.LogInformation($"[WatcherService]: {result}");

        }

        return ServiceUri;
    }

    private async Task<string> RunAsync(string command)
    {
        // Determine OS shell
        string shell;
        string shellArgs;

        if (RuntimeInformation.IsOSPlatform(OSPlatform.Windows))
        {
            shell = "cmd.exe";
            shellArgs = $"/c {command}";
        }
        else
        {
            // macOS + Linux
            shell = "/bin/bash";
            shellArgs = $"-c \"{command}\"";
        }

        var psi = new ProcessStartInfo(shell, shellArgs)
        {
            RedirectStandardOutput = true,
            RedirectStandardError = true,
            UseShellExecute = false,
            CreateNoWindow = true
        };

        using var p = Process.Start(psi);

        var output = await p!.StandardOutput.ReadToEndAsync();
        var error = await p.StandardError.ReadToEndAsync();

        if (!string.IsNullOrWhiteSpace(error))
        {
            return error;
        }

        return output;
    }
}

```


Replace the contents of the `IntroToDistributedAI.WatcherService/Worker.cs`
```csharp
public class Worker : BackgroundService
{
    private readonly ILogger<Worker> _logger;
    private readonly FoundryLocalSupervisor _supervisor;

    public Worker(ILogger<Worker> logger, FoundryLocalSupervisor supervisor)
    {
        _logger = logger;
        _supervisor = supervisor;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        if (!await _supervisor.IsRunningAsync())
        {
            // Start Foundry Local when the Worker Service boots
            var foundryLocalUri = await _supervisor.StartAsync();
            _logger.LogInformation("[WatcherService]: Foundry Local URI is {FoundryLocalUri}", foundryLocalUri);
        }
        else
        {
            _logger.LogInformation("[WatcherService]: Foundry Local service is already running at {FoundryLocalUri}", _supervisor.ServiceUri);
        }


        // Keep Foundry Local alive
        while (!stoppingToken.IsCancellationRequested)
        {
            await _supervisor.EnsureRunningAsync();
            await Task.Delay(TimeSpan.FromSeconds(15), stoppingToken);
        }

        // Stop Foundry Local when the Worker Service shuts down
        await _supervisor.StopAsync();
    }
}

```

Replace the contents of the `IntroToDistributedAI.WatcherService/Program.cs` 

```csharp

var builder = Host.CreateApplicationBuilder(args);

// Add FoundryLocalSupervisor to the service collection so it can be injected into the Worker service
builder.Services.AddSingleton<FoundryLocalSupervisor>();

// Add the Worker service to the service collection so it runs as a background service
builder.Services.AddHostedService<Worker>();

var host = builder.Build();

// Run the host, which will start the Worker service and manage its lifetime
host.Run();

// The host will keep running, managing the lifetime of the Worker service and ensuring Foundry Local is started and monitored


```

Confirm that solution builds still
```
dotnet build
```

## 4. Test Watcher Service

Run the solution
```bash
aspire run
```

Open Dashboard and inspect the console logs for the Watcher Service

Manually shutdown Foundry Local to confirm that Watcher Service will restart it
```bash
foundry service stop
```

Confirm that it is restarted
```bash
foundry service status
```

## 5. Implement Tutor Agent

Replace the contents of the `IntroToDistributedAI.TutorAgent/Program.cs` 

```csharp
using System.ClientModel;
using Microsoft.Agents.AI;
using OpenAI;
using OpenAI.Chat;
using Microsoft.Extensions.AI;

var foundryLocalHost = Environment.GetEnvironmentVariable("FOUNDRY") ?? throw new InvalidOperationException("FOUNDRY environment variable is not set.");

var builder = WebApplication.CreateBuilder(args);

builder.AddServiceDefaults();

var model = builder.Configuration["FoundryLocal:Model"]; // <-- Need to provide model configuration

var client = new OpenAIClient(
    credential: new ApiKeyCredential("can be anything - foundry local"),
    options: new OpenAIClientOptions
    {
        // Use the foundry local host environment variable for the endpoint
        Endpoint = new Uri($"{foundryLocalHost}v1")
    });

var agent = client
    .GetChatClient(model)
    .AsAIAgent(
        name: "TutorAgent",
        instructions:
            "You are a precise teaching assistant. " +
            "Explain your reasoning step by step, use short paragraphs, " +
            "and give practical examples when useful.");

builder.Services.AddSingleton<AIAgent>(agent);

var app = builder.Build();

app.Use(async (context, next) =>
{
    Console.WriteLine($"Request: {context.Request.Method} {context.Request.Path}");
    await next();
    Console.WriteLine($"Response: {context.Request.Method} Status Code: {context.Response.StatusCode}");
});

app.MapPost("/chat", async (ChatRequest request, AIAgent agent) =>
{
    try
    {
        if (string.IsNullOrWhiteSpace(request.Message))
        {
            Console.WriteLine("Bad request: Message is required.");
            return Results.BadRequest(new { error = "Message is required." });
        }
        
        Console.WriteLine($"Message received: {request.Message}");
        
        var response = await agent.RunAsync(request.Message);

        return Results.Ok(new ChatResponse(response.ToString() ?? string.Empty));
    }
    catch (Exception ex)
    {
        Console.WriteLine($"Error processing request: {ex.Message}");
        return Results.StatusCode(500);
    }
});

app.MapDefaultEndpoints();

app.Run();

public sealed record ChatRequest(string Message);
public sealed record ChatResponse(string Reply);
```

About 9 lines of code from the top of the above code file, we are reading a model value from configuration.  We need to add this configuration; otherwise, we'll a null reference exception will be thrown and are application will crash.

Replace the content in the `IntroToDistributedAI\appsettings.json`
```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*",
  "FoundryLocal": {
    "Model": "qwen2.5-0.5b-instruct-openvino-npu:5"
  }
}

```
Replace the hardware specific model variant with a model available to your Foundry Local instance
```bash
foundry cache ls
```


## 6. Test Tutor Agent with .http file

Confirm solution builds
```bash
dotnet build
```

Run the solution
```bash
aspire run
```

Add a new file to TutorAgent project named `TutorAgent.http`

Replace the contents of `TutorAgent.http` with the following.
```http
@tutor_host = https://localhost:7086

# Test TutorAgent
POST {{tutor_host}}/chat
Content-Type: application/json

{
  "Message": "Hello, TutorAgent!"
}

###

# Test Foundry Local directly
POST http://localhost:53535/v1/chat/completions
Content-Type: application/json

{
  "model": "qwen2.5-0.5b-instruct-openvino-npu:5",
  "messages": [
    {
      "role": "user",
      "content": "Hello"
    }
  ]
}
```
There are two tests in the .http file.  
1. Test TutorAgent
2. Test Foundry

You only need to use Test Foundry if Test Tutor is failing

Test Tutor by clicking the link `Send Request` link directly above `POST {{tutor_host}}/chat`

> Note: If you do not have a Send Request link, then you are either using a profile that does not have HTTP Client extension installed or your file extension is not .http

You should get a response from the Tutor agent that looks something like this.
```http
HTTP/1.1 200 OK
Connection: close
Content-Type: application/json; charset=utf-8
Date: Sat, 11 Jul 2026 22:52:30 GMT
Server: Kestrel
Transfer-Encoding: chunked

{
  "reply": "Hello! How can I assist you today? I'm here to help you understand complex concepts clearly and provide the information you need efficiently. So, what is something interesting or challenging that you're trying to learn about, and could you tell me more about it?"
}
```

## 7. Implement Console Client

Replace the contents of the `IntroToDistributedAI.ConsoleClient/Program.cs` 

```csharp

using System.Net.Http.Json;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;


// Retrieve the tutor host URL from the environment variable
var tutorAgentHost = Environment.GetEnvironmentVariable("TUTOR_HTTPS") ?? throw new InvalidOperationException("TUTOR_HTTPS environment variable is not set.");

var builder = Host.CreateApplicationBuilder(args);

builder.AddServiceDefaults();

builder.Services.AddHttpClient("tutor", client =>
{
    client.BaseAddress = new Uri(tutorAgentHost);
});

using var host = builder.Build();

var httpClientFactory = host.Services.GetRequiredService<IHttpClientFactory>();
var agentServer = httpClientFactory.CreateClient("tutor");

Console.WriteLine($"Agent Server logical endpoint: {agentServer.BaseAddress}");

using var response = await agentServer.PostAsJsonAsync("/chat", new { Message = "Hello" });
var body = await response.Content.ReadAsStringAsync();

Console.WriteLine($"Agent Server responded with {(int)response.StatusCode} {response.ReasonPhrase}");
Console.WriteLine(body);

```

## 8. Test Console Client 

Confirm solution builds
```bash
dotnet build
```

Run the solution
```bash
aspire run
```

Inspect the console logs for the Console Client app.  You should see something similar to this.

```bash
Agent Server responded with 200 OK
{"reply":"Hello! How can I assist you today?"}
```


## Congrats! You finished the Hands-on.