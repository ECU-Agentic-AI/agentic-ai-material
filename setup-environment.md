
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
- [3. Ubuntu 24/Mint 22 Installation](#3-ubuntu-24mint-22-installation)
    - [3.0 Install curl](#30-install-curl)
    - [3.1 Install .NET SDK 10](#31-install-net-sdk-10)
    - [3.2 Install Foundry Local](#32-install-foundry-local-linux)
    - [3.3 Install Ollama](#33-install-ollama-linux)
    - [3.4 Configure .NET 10 App](#34-configure-net-10-app-to-use-both-backends)
- [4. Install and Configure VS Code](#4-install-and-configure-vs-code)
    - [4.1 Install VS Code](#41-install-vs-code)
    - [4.2 Create the agentic-ai Profile](#42-create-the-agentic-ai-profile)
    - [4.3 Turn On Auto Save](#43-turn-on-auto-save)
    - [4.4 Install Recommended C# Extensions](#44-install-recommended-c-extensions)
- [5. Using REST Client in VS Code](#5-using-rest-client-in-vs-code)


---

# 1. WINDOWS INSTALLATION (GPU‑Accelerated)

> Note: I recommend that you use WinGet for installing applications. `winget --version` to confirm it is installed.  If not installed, you can install from Microsoft Store  `ms-windows-store://pdp/?productid=9NBLGGH4NNS1`

> Note: I recommend that you install CoreUtils.  This will make the most common linux commands native in windows shells...including cUrl.  `winget install Microsoft.Coreutils`

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

### Install - Use WinGet

```powershell
winget install Foundry
```

### Verify installation:

```powershell
foundry --version
```

### Pull a model:

```powershell
foundry model download qwen2.5-0.5b
```
> Note: You can specify the model alias, foundry will inspect your hardware, and download the best hardware variant for your system.

### View downloaded models

```powershell
foundry cache list
```

### Test:

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

### Test REST API:

#### Check foundry service status

```powershell
foundry service status
```

#### Start the local server:

```powershell
foundry service start
```

Default endpoint:

```
No default endpoint.  Need to pay attention to console output for port or explicitly set the endpoint configuration if you want to use REST API.
```

Using Powershell
```powershell
curl http://127.0.0.1:54070/v1/chat/completions -d "{ ""model"": ""qwen2.5-0.5b-instruct-openvino-npu:5"", ""messages"": [ { ""role"": ""system"", ""content"": ""You are a NFL football analyst"" }, { ""role"": ""user"", ""content"": ""Who will win the super bowl in 2027"" } ], ""response_format"": { ""type"": ""json_object"" } }"
```
Using REST Client in VS Code
```
POST http://127.0.0.1:54070/v1/chat/completions
Content-Type: application/json

{
    "model": "qwen2.5-0.5b-instruct-openvino-npu:5",
    "messages": [
        {"role": "user", "content": "Hello!"}
    ]
}
```

> Note: You must specify the specific model variant when using Foundry Local REST API.

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
- You can download using the provided PowerShell command or download the installer
- Install → restart terminal.

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

```powershell
curl http://localhost:11434/v1/chat/completions `
  -H "Content-Type: application/json" `
  -d "{""model"":""llama3"",""messages"":[{""role"":""user"",""content"":""Explain vector databases.""}]}"
```
Using REST Client in VS Code
```
POST http://127.0.0.1:11434/v1/chat/completions
Content-Type: application/json

{
    "model": "llama3",
    "messages": [
        {"role": "user", "content": "Hello!"}
    ]
}
```

> Note: You can use the model alias with Ollama REST API.

---

## 1.4 Configure .NET 10 App to Use Both Backends

Create project:

Decide where you want to place Agentic AI source code on your file system, then create your first project.

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

You should see:

```
10.x.x
```

---
---

## 2.2 Install Foundry Local (macOS)

> Note: Foundry Local is a native AI solution that is embedded into your application.  The following install is to provide Ollama-like REST functionality, but it is not required for native AI embedding.

### Install - Use WinGet

```bash
brew tap microsoft/foundrylocal

brew trust --formula microsoft/foundrylocal/foundrylocal

brew install foundrylocal

```
> Note: You get an error when installing foundry local saying you must install or update your xcode command line tools.  I believe foundry requires the latest version of xcode.  Below are the commands if you received the error message.

Only run these commands if you received the xcode commandline error message
```bash
sudo rm -rf /Library/Developer/CommandLineTools

sudo xcode-select --install

# Accept license agreement

# Try installing foundry local again
brew install foundrylocal
```

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

Using REST Client in VS Code
```
POST http://127.0.0.1:54070/v1/chat/completions
Content-Type: application/json

{
    "model": "qwen2.5-0.5b-instruct-openvino-npu:5",
    "messages": [
        {"role": "user", "content": "Hello!"}
    ]
}
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
- You can download either using curl or download dmg package
- Install → macOS will prompt for security approval.
- Install → restart terminal.

Verify:

```bash
ollama --version
```

Start service:

```bash
ollama serve
```

Default endpoint:

```
http://localhost:11434/v1
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

Using REST Client in VS Code
```
POST http://127.0.0.1:54070/v1/chat/completions
Content-Type: application/json

{
    "model": "llama3",
    "messages": [
        {"role": "user", "content": "Hello!"}
    ]
}
```

> Note: You can use the model alias with Ollama REST API.

---

## 2.4 Configure .NET 10 App (Same Code as Windows)

Decide where you want to place Agentic AI source code on your file system, then create your first project.

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


# 3. UBUNTU 24/MINT 22 INSTALLATION

## 3.0 Install curl

Install using apt package manager

update packages first
```bash
sudo apt update
```
install curl
```bash
sudo apt-get install -y curl
```

Confirm installation

```bash
curl --version
```

## 3.1 Install .NET SDK 10

if using something other than Ubuntu/Mint 24 check for other installation options:
- https://dotnet.microsoft.com/en-us/download/dotnet/10.0

Download and install:

```bash
sudo apt-get install -y dotnet-sdk-10.0
```

Verify:

```bash
dotnet --version
```

You should see:

```bash
10.x.x
```

## 3.2 Install Foundry Local (Linux)

> Note: Foundry Local is a native AI solution that is embedded into your application. The following install is to provide Ollama-like REST functionality, but it is not required for native AI embedding.

> Note: Microsoft does not currently distribute Foundry Local through apt or an official Debian package. Linux users install by extracting a tarball from the GitHub releases page and adding the binary to PATH.

> Note: The standalone Foundry Local CLI for Linux is currently distributed as a preview release (separate track from the stable v1.2.x SDK). The "preview" label reflects distribution maturity, not runtime instability.

> Note: The Linux CLI preview 0.10.1 has a slightly different command surface than the Windows and macOS 1.2.x CLIs. The commands in this section use the Linux syntax. Notable divergences:
> - `foundry run <model>` (Linux) vs `foundry model run <model>` (Windows/macOS)
> - `foundry server start/stop/status` (Linux) vs `foundry service start/stop/status` (Windows/macOS)
> - `foundry model list` does not support the `--filter` flag on the Linux preview CLI

### Install - Download from GitHub Releases

Latest Linux x64 CLI preview at time of writing: `cli-preview-0.10.1`.  Check the releases page for newer versions:
- https://github.com/microsoft/Foundry-Local/releases

The tarball ships with an `install.sh` script that copies the binaries to `/usr/local/lib/foundry-cli/` and symlinks the CLI at `/usr/local/bin/foundry`.  `/usr/local/bin/` is already in the default PATH on Ubuntu/Mint, so no shell configuration is needed.

Download, extract, and run the installer:

```bash
# Download the Linux x64 CLI preview tarball
curl -L -o /tmp/foundry-linux-x64.tar.gz \
  https://github.com/microsoft/Foundry-Local/releases/download/cli-preview-0.10.1/foundry-0.10.1-linux-x64.tar.gz

# Extract to a temporary staging directory
mkdir -p /tmp/foundry-install
tar -xzf /tmp/foundry-linux-x64.tar.gz -C /tmp/foundry-install --strip-components=1

# Run Microsoft's install script (installs to /usr/local/, needs sudo)
sudo /tmp/foundry-install/install.sh

# Clean up staged files
rm -rf /tmp/foundry-install /tmp/foundry-linux-x64.tar.gz
```

### Verify installation:

```bash
foundry --version
```

### Pull a model:

```bash
foundry model download qwen2.5-0.5b
```
> Note: You can specify the model alias, foundry will inspect your hardware, and download the best hardware variant for your system.

### View downloaded models

```bash
foundry cache list
```

> Note: On Linux the downloaded variant differs from Windows.  As of the 0.10.1 preview CLI, Linux downloads the `generic-cpu` variant of the model.  Take note of the exact variant name from `foundry cache list` — you will need it for the REST API calls below.

> Note: The Foundry Local CLI preview 0.10.1 for Linux does not currently ship a CUDA execution provider, even though the Windows and macOS CLIs do.  Foundry Local's `status` command detects your NVIDIA GPU (shown under System > GPU), but the execution provider field is empty and model downloads pull the `generic-cpu` variant regardless of CUDA installation state.  Installing the NVIDIA CUDA Toolkit does not change this behavior for the current preview.  The CPU variant of `qwen2.5-0.5b` responds quickly enough for the class demos.  Track this gap at https://github.com/microsoft/Foundry-Local/issues.

### Test:

```bash
foundry run qwen2.5-0.5b
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

### Test REST API:

#### Check foundry server status

```bash
foundry server status
```

#### Start the local server:

```bash
foundry server start
```

Default endpoint:

The Foundry Local daemon assigns a dynamic port on startup.  To find the current port, use:

```bash
foundry server status
```

This prints the daemon state and the `Web URLs` field (e.g. `http://127.0.0.1:38633`).  Substitute your actual port in the `curl` examples below — the `54070` shown is illustrative only.

```bash
curl http://127.0.0.1:54070/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen2.5-0.5b-instruct-generic-cpu:4",
    "messages": [
      { "role": "system", "content": "You are a NFL football analyst" },
      { "role": "user", "content": "Who will win the super bowl in 2027" }
    ],
    "response_format": { "type": "json_object" }
  }'
```

> Note: You must specify the specific model variant when using Foundry Local REST API.  Use the variant name shown by `foundry cache list` — it will differ from the Windows OpenVINO NPU variant.  On Linux, `bash` + `curl` is sufficient for API testing; the VS Code REST Client extension shown for Windows/macOS is unnecessary here (the REST Client extension exists primarily to work around PowerShell's `curl` escaping).

Example response:
```json
{
    "model": "qwen2.5-0.5b-instruct-generic-cpu:4",
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

## 3.3 Install Ollama (Linux)

Install using the official one-line installer script.  It downloads Ollama to `/usr/local/bin/ollama`, creates a dedicated `ollama` system user, writes `/etc/systemd/system/ollama.service`, and starts the service automatically.

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

> Note: On Linux, Ollama runs as a **systemd service** (`ollama.service`), not as a foreground process.  The Windows and macOS sections above tell you to run `ollama serve` in a terminal — do not do that on Linux; the installer already started the service.  Manage the service with `systemctl` instead.

> Note: Unlike Foundry Local's Linux CLI preview (which is CPU-only), Ollama's Linux installer detects and configures NVIDIA CUDA GPU acceleration automatically during install.  If you see `>>> NVIDIA GPU installed.` at the end of the install script output, GPU inference is active and no additional configuration is required.  You can verify by running an inference and checking `nvidia-smi` — look for a compute-type (`C`) process named `llama-server` holding GPU memory in the Processes table.

Verify:

```bash
ollama --version
```

Confirm the systemd service is running:

```bash
systemctl status ollama
```

Look for `Active: active (running)` in the output.  If it is not running, start it with `sudo systemctl start ollama`.

Common systemd commands for managing the service:

```bash
sudo systemctl start ollama       # Start the service
sudo systemctl stop ollama        # Stop the service
sudo systemctl restart ollama     # Restart after configuration changes
sudo systemctl enable ollama      # Enable on boot (the installer usually does this already)
sudo journalctl -u ollama -f      # Tail the service logs (Ctrl+C to exit)
```

Default endpoint:

```
http://localhost:11434/v1
```

Pull a model:

```bash
ollama pull llama3
```

Test:

```bash
curl http://localhost:11434/api/generate -d '{ "model": "llama3", "prompt": "Hello" }'
```

The REST call above is not OpenAI compliant.  You must use OpenAI compliant calls when using Microsoft Agent Framework.

Here is an example of the same prompt using OpenAI compliant schema:

```bash
curl http://localhost:11434/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"llama3","messages":[{"role":"user","content":"Explain vector databases."}]}'
```

> Note: You can use the model alias with Ollama REST API (unlike Foundry Local, which requires the exact variant name from `foundry cache list`).

---

## 3.4 Configure .NET 10 App to Use Both Backends

Create project:

Decide where you want to place Agentic AI source code on your file system, then create your first project.

```bash
dotnet new console -n LocalAI
cd LocalAI
```

Add OpenAI client:

```bash
dotnet add package Microsoft.Extensions.AI.OpenAI
```

Open project in VS Code:

```bash
code .
```

Replace contents of `Program.cs` with the code below.  Two Linux-specific adjustments compared to the Windows/macOS versions above:

- **Foundry Local port** — replace `<your-foundry-port>` with the dynamic port shown by `foundry server status` (e.g. `38633`).  The Windows/macOS examples hardcode `22334`, but the Linux CLI preview assigns ports dynamically at daemon startup.
- **Foundry Local model variant** — Linux uses `qwen2.5-0.5b-instruct-generic-cpu:4` rather than the Windows OpenVINO NPU variant.

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
        Endpoint = new Uri("http://localhost:<your-foundry-port>/v1")
    });

Console.WriteLine("Ollama:");
var ollamaResponse = await ollama
                            .GetChatClient("llama3")
                            .CompleteChatAsync("Say hello from Ollama.");

Console.WriteLine(ollamaResponse.Value.Content[0].Text);

Console.WriteLine("Foundry Local:");

var foundryResponse = await foundry
                            .GetChatClient("qwen2.5-0.5b-instruct-generic-cpu:4")
                            .CompleteChatAsync("Say hello from Foundry Local.");
                            
Console.WriteLine(foundryResponse.Value.Content[0].Text);
```

Run:

```bash
dotnet run
```

---




# 4. INSTALL AND CONFIGURE VS CODE


Visual Studio Code is the editor used for this course. Install it before working through the local AI and .NET examples.

## 4.1 Install VS Code

Download and install VS Code:

- https://code.visualstudio.com/download

Windows students can also install it with winget:

```powershell
winget install Microsoft.VisualStudioCode
```

macOS students can also install it with Homebrew:

```bash
brew install --cask visual-studio-code
```

After installation, open a new terminal and confirm that the `code` command is available:

```powershell
code --version
```

If the command is not found on macOS, open VS Code, press `Cmd+Shift+P`, run **Shell Command: Install 'code' command in PATH**, then restart the terminal.

## 4.2 Create the agentic-ai Profile

Use a separate VS Code profile for this course so extensions and settings stay organized.

1. Open VS Code.
2. Select the gear icon in the lower-left corner.
3. Select **Profiles**.
4. Select **Create Profile**.
5. Name the profile `agentic-ai`.
6. Create it from the default profile, then select **Create**.
7. Click the checkmark next to the new agentic-ai profile in the list of profiles to activate it.
8. Make sure the active profile shown in VS Code is `agentic-ai`.

## 4.3 Turn On Auto Save

Turn on Auto Save while the `agentic-ai` profile is active so the setting is saved to that profile.

1. Select **File**.
2. Select **Auto Save**.

## 4.4 Install Recommended C# Extensions

Install these extensions while the `agentic-ai` profile is active:

> Note: **IntelliCode for C# Dev Kit** (`ms-dotnettools.vscodeintellicode-csharp`) was deprecated by Microsoft on 12 November 2025 as part of a shift away from free local AI toward subscription GitHub Copilot.  Microsoft's official guidance is to uninstall the IntelliCode extension and rely on the built-in Roslyn language server (bundled with C# Dev Kit) for base language features, and optionally install **GitHub Copilot Chat** (`github.copilot-chat`) for AI-assisted completions.  The deprecated extension may still install but no longer receives updates.  Note this deprecation applies to VS Code only — Visual Studio 2026 retains IntelliCode.  See [Microsoft's deprecation announcement](https://github.com/MicrosoftDocs/intellicode/issues/614).

| Extension | Marketplace ID |
| --- | --- |
| C# Dev Kit | `ms-dotnettools.csdevkit` |
| C# | `ms-dotnettools.csharp` |
| ~~IntelliCode for C# Dev Kit~~ *(deprecated Nov 2025 — see note above)* | ~~`ms-dotnettools.vscodeintellicode-csharp`~~ |
| .NET Install Tool | `ms-dotnettools.vscode-dotnet-runtime` |
| GitHub Copilot Chat *(recommended replacement — optional)* | `github.copilot-chat` |

You can install them from the Extensions view, or use the command line:

```powershell
code --profile agentic-ai --install-extension ms-dotnettools.csdevkit
code --profile agentic-ai --install-extension humao.rest-client
code --profile agentic-ai --install-extension bierner.markdown-mermaid
code --profile agentic-ai --install-extension ms-vscode.live-server
code --profile agentic-ai --install-extension ms-windows-ai-studio.windows-ai-studio
code --profile agentic-ai --install-extension ms-azuretools.vscode-containers
code --profile agentic-ai --install-extension microsoft-aspire.aspire-vscode
```

---


# 5. USING REST CLIENT IN VS CODE


This guide includes REST API checks for Foundry Local and Ollama. You can run those requests from the command line with `curl`, or directly inside VS Code with the REST Client extension.

If you have not installed already, install the extension: 
```bash
code --profile agentic-ai --install-extension humao.rest-client
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
POST {{ollama-host}}/v1/chat/completions
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