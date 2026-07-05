# Foundry Local SDK Cheatsheet

A quick, classroom-ready reference for using the Foundry Local SDK from Python, JavaScript, C#, and Rust.

---

## 1. What the SDK Does

The Foundry Local SDK lets an application run local AI models without requiring students or end users to manually manage the Foundry CLI.

It handles:
- Hardware detection for CPU, GPU, and NPU
- Execution provider download and registration
- Model catalog lookup
- Model download and caching
- Model load and unload
- Optional local REST server startup
- OpenAI-compatible `/v1` API access

Use the SDK when you want local AI inside an application. Use the CLI when you want to explore models, test quickly, inspect cache state, or troubleshoot the local service.

---

## 2. Package Install Commands

### Python
Windows with WinML acceleration:

```bash
pip install foundry-local-sdk-winml openai
```

Cross-platform:

```bash
pip install foundry-local-sdk openai
```

### JavaScript / Node.js
Windows with WinML acceleration:

```bash
npm install foundry-local-sdk-winml openai
```

Cross-platform:

```bash
npm install foundry-local-sdk openai
```

### C#
Windows with WinML acceleration:

```bash
dotnet add package Microsoft.AI.Foundry.Local.WinML
dotnet add package OpenAI
```

Cross-platform:

```bash
dotnet add package Microsoft.AI.Foundry.Local
dotnet add package OpenAI
```

### Rust
Windows with WinML acceleration:

```bash
cargo add foundry-local-sdk --features winml
cargo add tokio --features full
cargo add tokio-stream anyhow
```

Cross-platform:

```bash
cargo add foundry-local-sdk
cargo add tokio --features full
cargo add tokio-stream anyhow
```

---

## 3. Core Workflow

Most Foundry Local SDK apps follow this sequence:

1. Initialize the SDK manager
2. Download and register execution providers
3. Find a model in the catalog
4. Download the model if needed
5. Load the model
6. Use either the native SDK client or the OpenAI-compatible REST server
7. Unload the model when finished
8. Stop the web service if you started it

Important distinction:

| Term | Meaning |
| --- | --- |
| Alias | Friendly model name, such as `qwen2.5-0.5b` |
| Model ID | Exact hardware-specific model variant selected by Foundry Local |
| Catalog | Available models for the current machine |
| Cache | Models already downloaded locally |
| Loaded model | Model currently active in the local runtime |
| REST server | Optional OpenAI-compatible local HTTP service |

Use aliases for selection. Use the resolved model ID when sending requests to the OpenAI-compatible API.

---

## 4. Python Quickstart

### List available models

```python
import asyncio
from foundry_local_sdk import Configuration, FoundryLocalManager

async def main():
	config = Configuration(app_name="foundry_local_class")
	FoundryLocalManager.initialize(config)
	manager = FoundryLocalManager.instance

	models = manager.catalog.list_models()
	print(f"Models available: {len(models)}")
	for model in models:
		print(model.alias)

if __name__ == "__main__":
	asyncio.run(main())
```

### Load a model and call it with the OpenAI SDK

```python
import openai
from foundry_local_sdk import Configuration, FoundryLocalManager

config = Configuration(app_name="foundry_local_class")
FoundryLocalManager.initialize(config)
manager = FoundryLocalManager.instance

manager.download_and_register_eps()

model = manager.catalog.get_model("qwen2.5-0.5b")
model.download()
model.load()

manager.start_web_service()
base_url = f"{manager.urls[0]}/v1"

client = openai.OpenAI(
	base_url=base_url,
	api_key="none",
)

response = client.chat.completions.create(
	model=model.id,
	messages=[
		{"role": "system", "content": "You are a helpful assistant."},
		{"role": "user", "content": "Explain the golden ratio in one paragraph."},
	],
)

print(response.choices[0].message.content)

model.unload()
manager.stop_web_service()
```

### Stream a response

```python
response = client.chat.completions.create(
	model=model.id,
	messages=[{"role": "user", "content": "Give me three study tips."}],
	stream=True,
)

for chunk in response:
	if chunk.choices and chunk.choices[0].delta.content is not None:
		print(chunk.choices[0].delta.content, end="", flush=True)
print()
```

### Use the native chat client

```python
chat_client = model.get_chat_client()

result = chat_client.complete_chat([
	{"role": "system", "content": "You are concise."},
	{"role": "user", "content": "What is local inference?"},
])

print(result)
```

---

## 5. JavaScript Quickstart

### List available models

```javascript
import { FoundryLocalManager } from 'foundry-local-sdk';

const manager = FoundryLocalManager.create({
	appName: 'foundry_local_class',
	logLevel: 'info'
});

const models = await manager.catalog.getModels();
console.log(`Found ${models.length} models:`);

for (const model of models) {
	console.log(model.alias);
}
```

### Load a model and call it with the OpenAI SDK

```javascript
import { FoundryLocalManager } from 'foundry-local-sdk';
import { OpenAI } from 'openai';

const endpointUrl = 'http://127.0.0.1:5764';

const manager = FoundryLocalManager.create({
	appName: 'foundry_local_class',
	logLevel: 'info',
	webServiceUrls: endpointUrl
});

await manager.downloadAndRegisterEps();

const model = await manager.catalog.getModel('qwen2.5-0.5b');
await model.download();
await model.load();

manager.startWebService();

const openai = new OpenAI({
	baseURL: `${endpointUrl}/v1`,
	apiKey: 'notneeded',
});

const response = await openai.chat.completions.create({
	model: model.id,
	messages: [
		{ role: 'user', content: 'Explain local inference in one paragraph.' }
	],
});

console.log(response.choices[0].message.content);

await model.unload();
manager.stopWebService();
```

---

## 6. C# Quickstart

### List available models

```csharp
using Microsoft.AI.Foundry.Local;
using Microsoft.Extensions.Logging;

var config = new Configuration
{
	AppName = "foundry_local_class",
	LogLevel = Microsoft.AI.Foundry.Local.LogLevel.Information,
};

using var loggerFactory = LoggerFactory.Create(builder =>
{
	builder.SetMinimumLevel(Microsoft.Extensions.Logging.LogLevel.Information);
});

await FoundryLocalManager.CreateAsync(config, loggerFactory.CreateLogger<Program>());
var manager = FoundryLocalManager.Instance;

var catalog = await manager.GetCatalogAsync();
var models = await catalog.ListModelsAsync();

foreach (var model in models)
{
	Console.WriteLine(model.Alias);
}
```

### Load a model and call it with the OpenAI SDK

```csharp
using Microsoft.AI.Foundry.Local;
using Microsoft.Extensions.Logging;
using OpenAI;
using System.ClientModel;

var config = new Configuration
{
	AppName = "foundry_local_class",
	LogLevel = Microsoft.AI.Foundry.Local.LogLevel.Information,
	Web = new Configuration.WebService
	{
		Urls = "http://127.0.0.1:52495"
	}
};

using var loggerFactory = LoggerFactory.Create(builder =>
{
	builder.SetMinimumLevel(Microsoft.Extensions.Logging.LogLevel.Information);
});

await FoundryLocalManager.CreateAsync(config, loggerFactory.CreateLogger<Program>());
var manager = FoundryLocalManager.Instance;

await manager.DownloadAndRegisterEpsAsync();

var catalog = await manager.GetCatalogAsync();
var model = await catalog.GetModelAsync("qwen2.5-0.5b")
	?? throw new Exception("Model not found");

await model.DownloadAsync();
await model.LoadAsync();

await manager.StartWebServiceAsync();

var client = new OpenAIClient(
	new ApiKeyCredential("notneeded"),
	new OpenAIClientOptions
	{
		Endpoint = new Uri(config.Web.Urls + "/v1"),
	});

var chatClient = client.GetChatClient(model.Id);
var completion = chatClient.CompleteChat("What is local inference?");

Console.WriteLine(completion.Value.Content[0].Text);

await manager.StopWebServiceAsync();
await model.UnloadAsync();
```

---

## 7. Rust Quickstart

### List available models

```rust
use foundry_local_sdk::{FoundryLocalConfig, FoundryLocalManager};

#[tokio::main]
async fn main() -> anyhow::Result<()> {
	let manager = FoundryLocalManager::create(
		FoundryLocalConfig::new("foundry_local_class")
	)?;

	let models = manager.catalog().get_models().await?;
	println!("Models available: {}", models.len());

	Ok(())
}
```

### Native chat client pattern

```rust
let model = manager.catalog().get_model("qwen2.5-0.5b").await?;
model.download(None).await?;
model.load().await?;

let client = model.create_chat_client()
	.temperature(0.7)
	.max_tokens(256);

// Build messages with the SDK chat message types, then call:
// client.complete_chat(&messages, tools).await?;
// client.complete_streaming_chat(&messages, tools).await?;

model.unload().await?;
```

---

## 8. OpenAI-Compatible REST Pattern

Foundry Local can expose a local OpenAI-compatible API. This lets standard OpenAI SDKs and HTTP clients talk to a local model.

### Base URL

```text
http://127.0.0.1:<port>/v1
```

When the SDK starts the web service, use the URL from the SDK manager when available. When the CLI starts the service, get the current dynamic port with:

```bash
foundry service status
```

### API key

Local Foundry does not need a real cloud API key. Use a placeholder value such as:

```text
none
notneeded
```

### Model value

For OpenAI-compatible requests, pass the loaded model's resolved model ID:

```python
model=model.id
```

Do not assume the alias and the model ID are always the same. The alias can resolve to a hardware-specific variant.

---

## 9. Native SDK vs OpenAI-Compatible API

| Approach | Best for |
| --- | --- |
| Native SDK APIs | Tightly integrated local apps, direct model lifecycle control |
| OpenAI-compatible REST server | Reusing OpenAI SDK code, LangChain-style tools, REST harnesses, demos |
| CLI | Manual testing, model discovery, service troubleshooting |

Recommended classroom pattern:

1. Use CLI to discover a model alias
2. Use SDK to load that model in code
3. Use OpenAI SDK to call the local `/v1` endpoint
4. Compare the same prompt against cloud or other local runtimes

---

## 10. Model Lifecycle Cheatsheet

| Task | Python | JavaScript | C# | Rust |
| --- | --- | --- | --- | --- |
| Initialize manager | `FoundryLocalManager.initialize(config)` | `FoundryLocalManager.create(config)` | `FoundryLocalManager.CreateAsync(config, logger)` | `FoundryLocalManager::create(config)` |
| List models | `manager.catalog.list_models()` | `manager.catalog.getModels()` | `catalog.ListModelsAsync()` | `manager.catalog().get_models().await` |
| Get model | `manager.catalog.get_model(alias)` | `manager.catalog.getModel(alias)` | `catalog.GetModelAsync(alias)` | `manager.catalog().get_model(alias).await` |
| Download model | `model.download()` | `model.download()` | `model.DownloadAsync()` | `model.download(None).await` |
| Load model | `model.load()` | `model.load()` | `model.LoadAsync()` | `model.load().await` |
| Start REST server | `manager.start_web_service()` | `manager.startWebService()` | `manager.StartWebServiceAsync()` | `manager.start_web_service().await` |
| Stop REST server | `manager.stop_web_service()` | `manager.stopWebService()` | `manager.StopWebServiceAsync()` | `manager.stop_web_service().await` |
| Unload model | `model.unload()` | `model.unload()` | `model.UnloadAsync()` | `model.unload().await` |

---

## 11. Common CLI Commands That Help SDK Work

```bash
foundry --version
foundry model list
foundry model list --filter task=chat-completion
foundry model info qwen2.5-0.5b
foundry model load qwen2.5-0.5b
foundry service status
foundry service ps
foundry service restart
foundry cache list
```

Use these when you need to verify that Foundry Local can see models, load models, or expose a reachable service.

---

## 12. Troubleshooting

### Service connection error

If a command or SDK call cannot connect to the local service, restart the service:

```bash
foundry service restart
```

### No models appear

Try:

```bash
foundry model list
```

On first run, Foundry Local may download execution providers before listing models.

### OpenAI SDK cannot connect

Check:
- The REST server is started
- The base URL includes `/v1`
- The port is current
- The model is loaded
- The request uses `model.id`, not just the alias

### First run is slow

The first run can download execution providers and model files. Later runs are faster because the model is cached.

### Wrong hardware selected

Use a specific model ID instead of an alias when you need to force a particular CPU, GPU, or NPU variant.

---

## 13. Related Course Files

- [Foundry Local CLI Reference](foundry-cli-reference.md)
- [OpenAI-Compatible API Cheatsheet](openai-compatible.md)
- [Local Inference Comparison](local-inference-comparison.md)
- [Resource Links](resource-links.md)
