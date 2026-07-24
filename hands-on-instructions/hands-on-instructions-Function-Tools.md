# Hands-on Series: Agent Tools with C#, Ollama, and Microsoft Agent Framework


# Part 2: Function tools

## Concept

A function tool is ordinary application code that the model can request. Agent Framework:

1. describes the function to the model;
2. receives model-generated arguments;
3. invokes the C# function;
4. sends the function result back to the model;
5. lets the model produce the final answer.

The model chooses **whether** to call the tool and proposes arguments. Your C# code remains responsible for validation, authorization, side effects, and the result.

In MAF function tools are of type `AIFunction`
```csharp
AIFunction coursePolicyTool = AIFunctionFactory.Create(GetCoursePolicy);
```
AIFunction supports the schema required by OpenAI.  AIFunctionFactory generates the required schema from an existing method.
```json
{
    "type": "function",
    "function": {
    "name": "GetCoursePolicy",
    "description": "Look up an official course policy by topic.",
    "parameters": {
        "type": "object",
        "properties": {
        "query": { "type": "string" }
        },
        "required": ["topic"]
    }
    }
}
```

An AIFunction can be created from a named function or an anonymous / lambda function.  

`static` or `instance` function can be used.

`static` functions should only be used for stateless functions

`instance` functions should be used when there is state, dependency injection, or per agent configuration.

Meaningful function names and parameter names should be used to increase chance of model using your tool correctly.

The [Description("description text here")] attribute can be added to function name and each parameter to provide more detailed information to the mode.
```csharp
[Description("Look up an official course policy by topic.")]
static string GetCoursePolicy( [Description("Policy topic: deadlines, late work, attendance")] string topic)
{
    if (string.IsNullOrWhiteSpace(topic))
    {
        throw new ArgumentException("A policy topic is required.", nameof(topic));
    }

    // Models often use JSON-friendly underscores in arguments. Normalize the
    // value before comparing it with the human-readable policy names.
    string normalizedTopic = topic.Trim().Replace('_', ' ').ToLowerInvariant();

    return normalizedTopic switch
    {
        "deadlines" => "Assignments are due at 11:59 PM Eastern on the date shown.",
        "due dates" => "Assignments are due at 11:59 PM Eastern on the date shown.",
        "late work"  => "Late work is accepted for 48 hours with a 10 percent deduction.",
        _ => "No policy was found for that topic. Ask the instructor for clarification."
    };
}

```

> Note: MAF will execute a tool when the model requests, but he model provides the actual parameters. We do not have complete control of when model requests a tool, nor do we have control over what parameters are provided by the model.  

It is important that we provide clear and detailed instructions to the model to increase success.

Smaller models will be less reliable.
- less reliable in requesting tool when they should request the tool
- less reliable with the parameter values provided


The model requests to use a tool is made by model by responding with functionCall content type instead of the typical text content type.

There are many content types that are possible.  The most basic defined by OpenAI are `text`, `functionCall`, `functionResult`, and `usage`.

These content types are chunks of data returned in a streaming chat OpenAI scheme as a `ChatResponseUpdate`.

MAF wraps the `ChatResponseUpdate` in an `AgentResponseUpdate`.  We work with the `AgentResponseUpdate` in MAF.

The general content type of an `AgentResponseUpdate` is an `AIContent` abstraction.

Comparison of MAF AIContents to OpenAI content types
|AIContent | OpenAI content| 
|---|---|
|DataContent                        | data |
|ErrorContent                       | error |
|FunctionCallContent                | `functionCall` |
|FunctionResultContent              | `functionResult` |
|HostedFileContent                  | hostedFile |
|HostedVectorStoreContent           | hostedVectorStore |
|TextContent                        | `text` |
|TextReasoningContent               | reasoning |
|UriContent                         | uri |
|UsageContent                       | `usage` |
|ToolCallContent                    | toolCall |
|ToolResultContent                  | toolResult |
|InputRequestContent                | inputRequest |
|InputResponseContent               | inputResponse |
|ToolApprovalRequestContent         | toolApprovalRequest |
|ToolApprovalResponseContent        | toolApprovalResponse |
|McpServerToolCallContent           | mcpServerToolCall |
|McpServerToolResultContent         | mcpServerToolResult |
|ImageGenerationToolCallContent     | imageGenerationToolCall |
|ImageGenerationToolResultContent   | imageGenerationToolResult |
|CodeInterpreterToolCallContent     | codeInterpreterToolCall |
|CodeInterpreterToolResultContent   | codeInterpreterToolResult |
|WebSearchToolCallContent           | webSearchToolCall |
|WebSearchToolResultContent         | webSearchToolResult |


MAF automatically processes tool requests and sends response back to the model. 
```csharp

await foreach (AgentResponseUpdate update in tutorAgent.RunStreamingAsync(
        prompt,
        options: runOptions,
        cancellationToken: cancellation.Token))
{
    if (!string.IsNullOrEmpty(update.Text))
    {
        Console.Write(update.Text);
    }
}
```



This example gives the Tutor Agent access to official course policies. A deterministic lookup is better than asking the model to remember or invent policy.

## Create the project

```bash
dotnet new sln -n AiFunctionTools
dotnet new console -n TutorWithTools -f net10.0
dotnet sln AiFunctionTools.slnx add TutorWithTools/TutorWithTools.csproj     
cd TutorWithTools

dotnet add package Microsoft.Agents.AI --version 1.13.0
dotnet add package Microsoft.Agents.AI.OpenAI --version 1.13.0
dotnet add package Microsoft.Extensions.AI --version 10.8.0
dotnet add package OpenAI --version 2.12.0
```

### Packages

| Package | Purpose |
| --- | --- |
| `Microsoft.Agents.AI` | Supplies the agent and its streaming APIs. |
| `Microsoft.Agents.AI.OpenAI` | Connects an OpenAI `ChatClient` to Agent Framework. |
| `Microsoft.Extensions.AI` | Supplies `AIFunctionFactory`, `AIFunction`, `AIContent`, and function call/result content types. |
| `OpenAI` | Calls Foundry Local through its OpenAI-compatible endpoint. |

### Required `using` directives

| Namespace | Why it is needed |
| --- | --- |
| `System.ClientModel` | Local placeholder API credential. |
| `System.ComponentModel` | `DescriptionAttribute` for tool and parameter metadata. |
| `Microsoft.Agents.AI` | Agent types and streaming APIs. |
| `Microsoft.Extensions.AI` | Tool abstractions and function call/result content. |
| `OpenAI` | OpenAI-compatible client configuration. |
| `OpenAI.Chat` | Chat client agent adapter. |


## Create an agent helper file

Right-click on project folder and select new file
Name file `TutorAgentInfo.cs`
The `TutorAgentInfo.cs` file should be right next to the `Program.cs` file in the file explorer.
Replace the contents of the `TutorAgentInfo.cs` with the following:
```csharp
using System;

namespace TutorWithTools;

public class TutorAgentInfo
{
    public readonly static string Model = "qwen2.5:0.5b"; //"gpt-oss:latest";  //"llama3.2"; //"gemma4:latest"; //"qwen2.5:0.5b";
    public readonly static string CoursePolicyToolName = "GetCoursePolicy";
    public readonly static string TutorAgentName = "AgenticAiTutor";

    public readonly static string Instructions = 
"""
You are a tutor agent with expert‑level knowledge of the Microsoft Agent Framework and the ability to answer student questions about agents, skills, tools, orchestration, planning, and best practices.  
When a student asks a **class policy question**, the agent must invoke the **GetCoursePolicy** tool instead of answering from memory.

---

## **Behavioral Contract**
You must follow these rules:

### **1. Expert Domain Knowledge**
You must:
- Provide authoritative explanations of the Microsoft Agent Framework  
- Break down concepts such as skills, tools, planners, agent orchestration, memory, and local inference  
- Use structured, pedagogically sound explanations suitable for students

### **2. Policy‑Sensitive Behavior**
If a student asks anything related to:
- deadlines
- late work
- attendance
- grading
- office hours
- missed quizzes
- makeup exams
- resubmissions
- late work window  

You **must** call the `GetCoursePolicy` tool and return its result to the student.

In order to call `GetCoursePolicy`, your response must be content type=functionCall. 

If user asks about late work, the agent should respond with a functionCall specifying the `GetCoursePolicy` tool and the relevant query.
- *Correct Response Example:*{"$type":"functionCall","Name":"GetCoursePolicy","Arguments":{"topic":"late work"},"FinishReason":"tool_calls"}
- *Incorrect Response Example:*{"$type":"text","Name":"GetCoursePolicy","Arguments":{"topic":"late work"}}

You **must not** invent or guess policy information.

### **3. Intent Detection**
You must classify student queries into one of the following categories:

- **Framework Question**  
  Questions about agents, skills, tools, planners, memory, or architecture.  
  → Respond with expert explanation.

- **Class Policy Question**  
  Questions about course rules or expectations.  
  → Invoke `GetCoursePolicy`.

- **Mixed Question**  
  Contains both framework and policy elements.  
  → Answer the framework portion normally.  
  → Invoke `GetCoursePolicy` for the policy portion.

### **4. Tone & Pedagogy**
You must:
- Use clear, structured explanations  
- Provide examples when helpful  
- Avoid jargon unless defined  
- Encourage student understanding  
- Never shame or belittle a student for not knowing something


### **Policy Detection Rules**
A question is a **Class Policy Question** if it contains terms such as:

- “late work”, “late policy”
- “attendance”, “absence”
- “grading”, “grade weights”
- “make‑up exam”, “test policy”
- “office hours”
- “participation requirements”
- “course rules”
- “syllabus says”
- “how does the class handle…”

If any of these appear, the agent must call `GetCoursePolicy`.

---

## **Examples**

### **Example 1 — Framework Question**
**Student:** “What’s the difference between a skill and a tool?”  
→ Agent uses `ExplainConcept`.

### **Example 2 — Class Policy Question**
**Student:** “What’s the late assignment policy?”  
→ Agent calls `GetCoursePolicy`.

### **Example 3 — Mixed Question**
**Student:** “How does planning work, and how many points is the planning assignment worth?”  
→ Agent explains planning.  
→ Agent calls `GetCoursePolicy` for assignment points.

---

## **Guarantees**
The Tutor Agent:
- Never fabricates policy information  
- Always routes policy questions through `GetCoursePolicy`  
- Always provides expert‑level guidance on Microsoft Agent Framework topics
- Only use .Net 10 SDK references. Providing a response based on an older framework version is strictly prohibited.
- Maintains a supportive, instructional tone

""";


}

```

## Replace `Program.cs`

```csharp
using System.ClientModel;
using System.ComponentModel;
using System.Text.Json;
using System.Text.Json.Serialization.Metadata;
using Microsoft.Agents.AI;
using Microsoft.Extensions.AI;
using OpenAI;
using OpenAI.Chat;
using TutorWithTools;

// Ollama OpenAI Compatible REST service
string endpoint =  "http://127.0.0.1:11434/v1";

// Use a tools-capable model.
string model = TutorAgentInfo.Model; 

var openAIClient = new OpenAIClient(
    new ApiKeyCredential("ollama-local"),
    new OpenAIClientOptions
    {
        Endpoint = new Uri(endpoint)
    });

[Description("Look up an official course policy by topic.")]
static string GetCoursePolicy(
    [Description("Policy topic: deadlines, late work, attendance, grading, office hours, missed quizzes, makeup exams, resubmissions, late work window")]
    string topic)
{
    if (string.IsNullOrWhiteSpace(topic))
    {
        throw new ArgumentException("A policy topic is required.", nameof(topic));
    }

    // Models often use JSON-friendly underscores in arguments. Normalize the
    // value before comparing it with the human-readable policy names.
    string normalizedTopic = topic.Trim().Replace('_', ' ').ToLowerInvariant();

    return normalizedTopic switch
    {
        "deadlines" => "Assignments are due at 11:59 PM Eastern on the date shown.",
        "due dates" => "Assignments are due at 11:59 PM Eastern on the date shown.",
        "late work"  => "Late work is accepted for 48 hours with a 10 percent deduction.",
        "attendance" => "Students are expected to attend all classes and participate actively.",
        "class participation" => "Students are expected to attend all classes and participate actively.",
        "grading" => "Final grade is weighted with the following weights: assignments 40%, exams 50%, participation 10%.",
        "final grade" => "Final grade is weighted with the following weights: assignments 40%, exams 50%, participation 10%.",
        "office hours" => "Instructor's office hours are posted on the syllabus.",
        "instructor availability" => "Instructor's office hours are posted on the syllabus.",
        "missed quizzes" => "Missed quizzes can be made up within one week with prior approval.",
        "makeup quizzes" => "Missed quizzes can be made up within one week with prior approval.",
        "missed exams" => "Makeup exams are allowed with documented justification.",
        "makeup exams" => "Makeup exams are allowed with documented justification.",
        "resubmissions" => "One resubmission is allowed before the late-work window closes.",
        "late work window" => "The late work window is 48 hours after the original deadline.",
        _ => "No policy was found for that topic. Ask the instructor for clarification."
    };
}


AIFunctionFactoryOptions coursePolicyToolOptions = new AIFunctionFactoryOptions
{
  Name = "GetCoursePolicy",
  Description = "Look up an official course policy by topic.",
  SerializerOptions = new JsonSerializerOptions
  {
      PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
      WriteIndented = true,
      TypeInfoResolver = new DefaultJsonTypeInfoResolver()
  }
};

AIFunction coursePolicyTool = AIFunctionFactory.Create(GetCoursePolicy,coursePolicyToolOptions);

// AIFunctionFactory reads the method signature and Description attributes to
// build the JSON schema that the model sees.
/*
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "search",
        "description": "Search the web for information.",
        "parameters": {
          "type": "object",
          "properties": {
            "query": { "type": "string" }
          },
          "required": ["query"]
        }
      }
    }
  ],
*/

AIAgent tutorAgent = openAIClient
    .GetChatClient(model)
    .AsAIAgent(
        name: TutorAgentInfo.TutorAgentName,
        instructions:TutorAgentInfo.Instructions,
        tools: [coursePolicyTool]);

using var cancellation = new CancellationTokenSource(TimeSpan.FromMinutes(1));

var runOptions = new ChatClientAgentRunOptions(
    new ChatOptions 
    { 
        ToolMode = ChatToolMode.RequireAny,         // must use at least one tool  (demo only)
        // specific instructions for this request
        Instructions = "If a course policy question arises, you must call the GetCoursePolicy tool. Your audience is a student.",
        AllowMultipleToolCalls = true,
        Tools = [coursePolicyTool],                 // ability to change tools for this specific request
        Temperature = 0.2f,                         // controls the randomness of the model's output, higher values make output more random
              
        // ModelId = "gemma4:latest",                         // ability to change model for this specific request
        // ResponseFormat = ChatResponseFormat.Json // ability to return structured JSON responses
        // Reasoning = new ReasoningOptions         // requires a model with thinking capabilities
        // {
        //     Effort = ReasoningEffort.Medium,
        //     Output = ReasoningOutput.Full
        // }
    });


// e.g.   dotnet run -- "Am I allowed to make up a missed quiz?"
string prompt = args.Length > 0
    ? string.Join(' ', args)
    : "What is the course policy on late work?";

Console.WriteLine($"Student: {prompt}");
Console.Write("Tutor: ");

var stream = tutorAgent.RunStreamingAsync(
    prompt,
    options: runOptions,
    cancellationToken: cancellation.Token);

await foreach (AgentResponseUpdate update in stream.WithCancellation(cancellation.Token))
{
    // Console.Write(JsonSerializer.Serialize(update));

    // foreach (AIContent content in update.Contents)
    // {
    //     // Tool events are operational information, so send them to stderr.
    //     // The student-facing answer continues to stream on stdout.
    //     if (content is FunctionCallContent)
    //     {
    //         Console.Error.WriteLine("\n[The tutor requested a function tool.]");
    //     }
    //     else if (content is FunctionResultContent)
    //     {
    //         Console.Error.WriteLine("[The function tool returned a result.]");
    //     }
    // }

    if (!string.IsNullOrEmpty(update.Text))
    {
        Console.Write(update.Text);
    }
}

Console.WriteLine();


```

Run it:

```bash
dotnet run
```

Try a second request:

```bash
dotnet run -- "Use GetCoursePolicy to check resubmissions."
```

## The pieces required for a function tool

| Piece | Where it appears |
| --- | --- |
| Tool implementation | `GetCoursePolicy` |
| Tool description | `[Description(...)]` on the method |
| Parameter description | `[Description(...)]` on `topic` |
| Tool schema creation | `AIFunctionFactory.Create(GetCoursePolicy)` |
| Agent access | `tools: [coursePolicyTool]` |
| Guidance for tool selection | The agent's `instructions` |
| Required demo call | `ChatToolMode.RequireAny` |
| Streaming call observation | `FunctionCallContent` |
| Streaming result observation | `FunctionResultContent` |

Descriptions matter because the model sees a schema, not your source-code intent. Clear names, descriptions, parameter types, and allowed values improve tool selection.

This lesson uses `ChatToolMode.RequireAny` because exactly one safe lookup tool is available and the purpose of the run is to observe a tool call. In a normal conversation, automatic selection is usually more appropriate. Do not require a tool on every request when the agent might answer directly, and do not force repeated calls to a tool with side effects.

## Benefits

- Runs trusted, testable, strongly typed C# code.
- Keeps deterministic facts and calculations outside the model.
- Has low overhead because it runs in the same application process.
- Can use dependency injection and existing application services.
- Is usually the simplest tool option.

## Limitations and risks

- Model-generated arguments are untrusted input and must be validated.
- A tools-capable model can still choose the wrong tool or no tool when tool mode is automatic.
- Small local models may follow tool instructions less reliably than larger models.
- A function with side effects can modify data more than once if retry behavior is not designed carefully.
- The function runs with the application's permissions.
- Tool descriptions can accidentally reveal sensitive implementation details.

Use approval or an application-controlled confirmation step before destructive, costly, or externally visible actions.

---