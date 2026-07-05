# Simple AI Chatbot using Foundry Local SDK

Below is a well documented C# example of simple chatbot using foundry local sdk.

```csharp
// Add required packages
// - dotnet add package Microsoft.AI.Foundry.Local
// - or dotnet add package Microsoft.AI.Foundry.Local.WinML

// Add using statements
// Using statements makes it so that you do not have to fully qualify a types name.
// For instance, if we did not specify:  using Betalgo.Ranul.OpenAI.ObjectModels.RequestModels;
// Instead of just referring to ChatMessage, we would have to refer to using fully qualified name
// Betalgo.Ranul.OpenAI.ObjectModels.RequestModels.ChatMessage
// Otherwise; dotnet does not know where to find the type definition for ChatMessage.
using Betalgo.Ranul.OpenAI.ObjectModels.RequestModels;
using Microsoft.AI.Foundry.Local;
using Microsoft.Extensions.Logging.Abstractions;

// Create a Cancellation Token

// It is best practice to provide cancellation tokens where they are supported
// to allow long running processes to be interruped and clean up resources

// You can reuse this Cancellation Token for all methods that take a CancellationToken in this class.
CancellationToken ct = new CancellationToken();


// Create foundry local configuration
// Be cautious about specifying a ModelCacheDir.  Sharing the same model with multiple Foundry applications is 
// valuable characteristic of Foundry Local.  If you specify a different model cache directory, then only Foundry applications 
// that specify the same directly will use the same models.  All others will download a duplicate model.
var config = new Configuration
{
    AppName = "Simple Chatbot",
    LogLevel = LogLevel.Information
};


// Initialize FoundryLocalManager - singleton instance
// You only have to initialize it once in your application.
// Every where else you just need retrieve the instance.
// Trying to call CreateAsync a second time will throw an error.
await FoundryLocalManager.CreateAsync(config, NullLogger.Instance);

var mgr = FoundryLocalManager.Instance;

// Retrieve a list of Execution Providers (EP) that are available for your system.
// This will be different for everyone's system because Execution Providers
// are specific the hardware, firmware, drivers, OS on your system.
var eps = mgr.DiscoverEps();

foreach (var ep in eps)
{
    Console.WriteLine($"EP Name: {ep.Name}");
}


// Download and register EPs with Foundry.  Registration in this scenario refers to
// Foundry Local specifically.  Registered EPs are used to identify the models available
// to you in the catalog.  There is a more familiar/general concept of execution providers 
// being registered, but this is referring an EP being registered with ONNX.  This only 
// happens when ONNX actually uses an EP in an inference session.  These two registrations
// are not related.

// If you hover over DownloadAndRegisterEpsAsync below, you will see that it takes an Action<string,double> 
// as a parameter. An Action is just an anonymous method that does not return a result.  The <string,double>
// in Action<string,double> says that your anonymous method will receive two inputs. The first is a string
// and the second is a double.  In the documentation, it tells you that the string is the name of the EP,
// and the double is the percent downloaded. DownloadAndRegisterEpsAsync will periodically call your
// anonymous function and provide the EP name and the Percent downloaded. It is up to you what you do 
// inside your anonymous method.  

// The Action is the first parameter, the second parameter is an optional CancellationToken.  

// Skips download of EP if it has previously been downloaded and new version is not available.
await mgr.DownloadAndRegisterEpsAsync((epname,percent) =>
{
    Console.WriteLine($"Percent Downloaded: {percent:F2} EP Name: {epname}"); // :F2 = format is 2 decimal places
}, ct);

// Retrieve the model catalogue
// Provided the optional CancellationToken
var catalog = await mgr.GetCatalogAsync(ct);

// show list of available models
var models = await catalog.ListModelsAsync(ct);

foreach(var m in models)
{
    Console.WriteLine($"Model Name: {m.Alias}");
}

// Retrieve model info for a specific model from the catalog
var model = await catalog.GetModelAsync("qwen2.5-0.5b", ct);

if( model is null)
{
    Console.WriteLine("Model did not exist in catalog.");
    return;
}

// Download the model

// DownloadAsync takes an Action<float> and optional Cancellation token
// for parameters.  Action<float> just means that you need to provide an
// anonymous method with one input parameter that is a float. The documentation
// tells us that this float represents the download progress.

// Skips download if model has already been downloaded
await model.DownloadAsync((progress) =>
{
    Console.WriteLine($"Downloading Model: {progress:0.00}%"); // The 
});

// Load the model
await model.LoadAsync(ct);

// Retrieve chat client from the model
var chatClient = await model.GetChatClientAsync(ct);

// Create list of chat messages  - role / content
List<ChatMessage> messages = new()
{
    new ChatMessage 
    {
        Role = "user", 
        Content = "Why does the earth look blue from outer space?"
    }
};

// Complete the chat - receive inference response from LLM. 

// Waits for the whole message to be returned
// as opposed to streaming which responses with each token as it is predicted. 
var response = await chatClient.CompleteChatAsync(messages, ct);

// This is the standard shape for OpenAI chat completion response model
// ChatCompletionCreateResponse
// - List of ChatChoiceResponse
//   - ChatMessage
//     - String
Console.WriteLine(response.Choices[0].Message.Content);

// unload the model

// When process is done, unload model to free up memory

// This was an example of a simple chatbot, but a more sophisticated
// chatbot or AI agent may run for an extended period of time before 
// being shutdown.  It is important that you unload the model whenever
// you shutdown the application.
await model.UnloadAsync(ct);


```