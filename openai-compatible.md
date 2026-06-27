


# OpenAI‑Compatible API Cheatsheet
A comprehensive, classroom‑ready reference for any runtime that implements the OpenAI API contract (v1).

---

# 1. What “OpenAI‑Compatible” Means
A server/runtime is *OpenAI‑compatible* when it:

- Implements the same **endpoints** as OpenAI’s API  
- Accepts the same **request schema**  
- Returns the same **response schema**  
- Supports the same **streaming format**  
- Works with the official **OpenAI SDKs** by only changing `base_url` and `api_key`

**Models do not provide compatibility. Runtimes do.**

Examples of compatible runtimes: vLLM, SGLang, Ollama, TGI, Foundry Local, LM Studio, Jan, OpenRouter, Groq, Cerebras, etc.

---

# 2. Core Endpoints Required for Compatibility

## 2.1 Chat Completions (Legacy)
`POST /v1/chat/completions`

### Request
```json
{
  "model": "llama3",
  "messages": [
    {"role": "system", "content": "You are an expert on LLMs"},
    {"role": "user", "content": "Explain vector databases."}
  ],
  "temperature": 0.7,
  "stream": false
}
```

### Response
```json
{
    "id": "chatcmpl-867",
    "object": "chat.completion",
    "created": 1782527417,
    "model": "llama3",
    "system_fingerprint": "fp_ollama",
    "choices": [
        {
            "index": 0,
            "message": {
                "role": "assistant",
                "content": "Vector databases! A fascinating topic in ..."
            },
            "finish_reason": "stop"
        }
    ],
    "usage": {
        "prompt_tokens": 28,
        "completion_tokens": 454,
        "total_tokens": 482
    }
}
```

---

## 2.2 Responses API (New Unified API)
`POST /v1/responses`

### Request
```json
{
  "model": "llama3",
  "input": "Summarize the role of attention in transformers.",
  "max_output_tokens": 200
}
```

### Response
```json
{
    "id": "resp_785215",
    "object": "response",
    "created_at": 1782526966,
    "completed_at": 1782526966,
    "status": "completed",
    "incomplete_details": null,
    "model": "llama3",
    "previous_response_id": null,
    "instructions": null,
    "output": [
        {
            "id": "msg_316879",
            "type": "message",
            "status": "completed",
            "role": "assistant",
            "content": [
                {
                    "type": "output_text",
                    "text": "In Transformers, attention plays a ...",
                    "annotations": [
                    ],
                    "logprobs": [
                    ]
                }
            ]
        }
    ],
    "error": null,
    "tools": [
    ],
    "tool_choice": "auto",
    "truncation": "disabled",
    "parallel_tool_calls": true,
    "text": {
        "format": {
            "type": "text"
        }
    },
    "top_p": 1,
    "presence_penalty": 0,
    "frequency_penalty": 0,
    "top_logprobs": 0,
    "temperature": 1,
    "reasoning": null,
    "usage": {
        "input_tokens": 20,
        "output_tokens": 200,
        "total_tokens": 220,
        "input_tokens_details": {
            "cached_tokens": 0
        },
        "output_tokens_details": {
            "reasoning_tokens": 0
        }
    },
    "max_output_tokens": 200,
    "max_tool_calls": null,
    "store": false,
    "background": false,
    "service_tier": "default",
    "metadata": {
    },
    "safety_identifier": null,
    "prompt_cache_key": null
}
```

---

## 2.3 Completions (Legacy Text)
`POST /v1/completions`

---

## 2.4 Embeddings
`POST /v1/embeddings`

### Request
```json
{
  "model": "nomic-embed-text",
  "input": "Neural networks are function approximators."
}
```

### Response
```json
{
  "data": [
    {
      "object": "embedding",
      "embedding": [0.0123, -0.9981, ...],
      "index": 0
    }
  ]
}
```

---

# 3. Streaming Format (SSE)
All OpenAI‑compatible runtimes must stream using **Server‑Sent Events**:

```
data: {"id":"...","object":"chat.completion.chunk","choices":[{"delta":{"content":"Hello"}}]}
data: {"id":"...","object":"chat.completion.chunk","choices":[{"delta":{"content":" world"}}]}
data: [DONE]
```

Key rules:
- Each chunk is a JSON object prefixed with `data: `
- Final message is `data: [DONE]`
- `delta` contains incremental tokens

---

# 4. Required Request Fields

| Field | Type | Description |
|-------|------|-------------|
| `model` | string | Model identifier (runtime‑specific) |
| `messages` | array | Chat messages (role + content) |
| `input` | string/array | For `/v1/responses` or embeddings |
| `temperature` | number | 0–2 |
| `max_tokens` / `max_output_tokens` | number | Output limit |
| `stream` | boolean | Enable SSE streaming |
| `stop` | string/array | Stop sequences |

---

# 5. Required Response Fields

| Field | Description |
|-------|-------------|
| `id` | Unique request ID |
| `object` | `chat.completion`, `response`, etc. |
| `choices` or `output` | Model output |
| `finish_reason` | `stop`, `length`, `content_filter`, etc. |
| `usage` | Token counts |

---

# 6. Error Format (Must Match OpenAI)

### Example Error
```json
{
  "error": {
    "message": "Model not found",
    "type": "invalid_request_error",
    "param": "model",
    "code": "model_not_found"
  }
}
```

---

# 7. Authentication
OpenAI‑compatible servers accept:

```
Authorization: Bearer <api_key>
```

Some runtimes ignore the key but still require the header.

---

# 8. Using the Official OpenAI SDK with Any Runtime

## Python
```python
from openai import OpenAI
client = OpenAI(base_url="http://localhost:8000/v1", api_key="none")

resp = client.chat.completions.create(
    model="llama-3.1-70b",
    messages=[{"role": "user", "content": "Hello"}]
)
print(resp.choices[0].message.content)
```

## JavaScript
```javascript
import OpenAI from "openai";
const client = new OpenAI({
  baseURL: "http://localhost:8000/v1",
  apiKey: "none"
});

const resp = await client.chat.completions.create({
  model: "llama-3.1-70b",
  messages: [{ role: "user", content: "Hello" }]
});
console.log(resp.choices[0].message.content);
```

---

# 9. Compatibility Checklist (For Runtimes)

## Must‑Have
- [ ] `/v1/chat/completions`
- [ ] `/v1/completions`
- [ ] `/v1/embeddings`
- [ ] OpenAI request schema
- [ ] OpenAI response schema
- [ ] SSE streaming format
- [ ] OpenAI‑style error objects
- [ ] Accept `Authorization: Bearer`
- [ ] Support `model` field
- [ ] Support `messages` array

## Should‑Have
- [ ] `/v1/responses`
- [ ] `/v1/audio/*`
- [ ] `/v1/images/*`
- [ ] `/v1/fine_tuning/*`

---

# 10. How Runtimes Implement Compatibility
Runtimes act as **translators**:

```
OpenAI SDK → OpenAI JSON → Runtime → Model → Runtime → OpenAI JSON → Client
```

The model never sees the OpenAI schema.  
The runtime handles:
- Tokenization  
- Sampling  
- Streaming  
- JSON shaping  
- Error shaping  

---

# 11. Common Pitfalls

| Issue | Cause | Fix |
|-------|-------|-----|
| `model_not_found` | Wrong model name | Use runtime’s model list |
| No streaming | Runtime missing SSE | Enable SSE in server |
| Wrong JSON shape | Runtime not fully compatible | Validate against OpenAI spec |
| SDK errors | Missing headers | Ensure `Authorization` + `Content-Type` |

---

# 12. Minimal Test Script (Universal)
```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer none" \
  -d '{
        "model": "test-model",
        "messages": [{"role": "user", "content": "ping"}]
      }'
```

If this returns a valid OpenAI‑shaped JSON → the runtime is compatible.

---

# 13. Mental Model for Students
**OpenAI API = USB port**  
**Runtime = computer**  
**Model = device you plug in**  

If the port shape matches, everything works.

---

# 14. Quick Reference Summary

- Compatibility = API contract, not model behavior  
- Runtimes implement the contract  
- Clients can swap backends by changing only `base_url`  
- Streaming must use SSE  
- Errors must match OpenAI format  
- `/v1/responses` is the future‑proof endpoint  

---

