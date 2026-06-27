# Foundry Local CLI Reference 

---

## 1. Overview
The Foundry Local CLI provides commands for:
- Managing AI models
- Controlling the Foundry Local service
- Managing the local model cache
- Inspecting execution providers
- Running models interactively

The CLI is organized into three major command groups:
- `foundry model`
- `foundry service`
- `foundry cache`

---

## 2. Installation & Verification

### Windows
```
winget install Microsoft.FoundryLocal
```

### macOS
```
brew tap microsoft/foundrylocal
brew install foundrylocal
```

### Verify installation
```
foundry --version
```

### If service connection errors occur
```
foundry service restart
```

---

## 3. Quick Help
```
foundry --help
```

---

# 4. MODEL COMMANDS

## 4.1 List all model commands
```
foundry model --help
```

---

## 4.2 Run a model interactively
```
foundry model run <model>
```
- Downloads the model if not cached
- Starts an interactive chat or text session

Example:
```
foundry model run qwen2.5-0.5b
```

---

## 4.3 List available models
```
foundry model list
```

On first run, Foundry Local downloads execution providers for your hardware.

---

## 4.4 Filter models
```
foundry model list --filter <key>=<value>
```

### Supported filter keys (authoritative list)
- `device` — CPU, GPU, NPU  
- `provider` — CPUExecutionProvider, CUDAExecutionProvider, WebGpuExecutionProvider, QNNExecutionProvider, OpenVINOExecutionProvider, NvTensorRTRTXExecutionProvider, VitisAIExecutionProvider  
- `task` — chat-completion, text-generation  
- `alias` — supports wildcard matching with `*`

### Negation
```
foundry model list --filter device=!GPU
```

### Wildcard alias matching
```
foundry model list --filter alias=qwen*
```

### Examples
```
foundry model list --filter device=GPU
foundry model list --filter task=chat-completion
foundry model list --filter provider=CUDAExecutionProvider
```

---

## 4.5 Show model info
```
foundry model info <model>
```

### Show license
```
foundry model info <model> --license
```

---

## 4.6 Download a model without running it
```
foundry model download <model>
```

---

## 4.7 Load a model into the service
```
foundry model load <model>
```

---

## 4.8 Unload a model
```
foundry model unload <model>
```

---

# 5. SERVICE COMMANDS

## 5.1 List service commands
```
foundry service --help
```

---

## 5.2 Start service
```
foundry service start
```

---

## 5.3 Stop service
```
foundry service stop
```

---

## 5.4 Restart service
```
foundry service restart
```

---

## 5.5 Show service status
```
foundry service status
```
Displays:
- Whether the service is running  
- Local endpoint URL  
- Dynamic port assignment  

---

## 5.6 List loaded models
```
foundry service ps
```

---

## 5.7 Show service logs
```
foundry service diag
```

---

## 5.8 Set service configuration
```
foundry service set <options>
```

---

# 6. CACHE COMMANDS

## 6.1 List cache commands
```
foundry cache --help
```

---

## 6.2 Show cache directory
```
foundry cache location
```

---

## 6.3 List cached models
```
foundry cache list
```

---

## 6.4 Change cache directory
```
foundry cache cd <path>
```

---

## 6.5 Remove a cached model
```
foundry cache remove <model>
```

---

# 7. EXECUTION PROVIDERS

Execution providers accelerate model inference on specific hardware.

## 7.1 Built-in providers
- CPUExecutionProvider  
- WebGpuExecutionProvider  
- CUDAExecutionProvider  

## 7.2 Plugin providers (auto-downloaded on Windows)
- NvTensorRTRTXExecutionProvider  
- OpenVINOExecutionProvider  
- QNNExecutionProvider  
- VitisAIExecutionProvider  

Each has specific hardware and driver requirements.

---

# 8. OPEN WEBUI INTEGRATION

## 8.1 Start a model
```
foundry model run <model>
```

## 8.2 Get service endpoint
```
foundry service status
```

## 8.3 Connect Open WebUI
- Open WebUI → Settings → Admin Settings → Connections → Direct Connections  
- Add new connection:
  - URL: `http://localhost:<PORT>/v1`
  - Auth: None

If no models appear:
```
foundry model run <model>
```

---

# 9. UPGRADE FOUNDRY LOCAL

## Windows
```
winget upgrade --id Microsoft.FoundryLocal
```

## macOS
```
brew upgrade foundrylocal
```

---

# 10. UNINSTALL FOUNDRY LOCAL

## Windows
```
winget uninstall Microsoft.FoundryLocal
```

## macOS
```
brew rm foundrylocal
brew untap microsoft/foundrylocal
brew cleanup --scrub
```

---

# 11. TROUBLESHOOTING

## Service connection error
Example error:
```
Exception: Request to local service failed. Uri: http://127.0.0.1:0/foundry/list
```

Fix:
```
foundry service status
foundry service restart
```

---

# 12. MODEL TASK TYPES (Authoritative List)
These are the **only** valid `task` filter values:
- `chat-completion`
- `text-generation`

---

# 13. Notes
- All filter comparisons are case-insensitive.
- Only one filter may be used per command.
- Unrecognized filter keys produce an error.
- Aliases automatically select the best hardware variant.

---

# 14. Reference
Source: Foundry Local CLI Reference – Microsoft Learn  
Reference ID: turn0browsertab1

