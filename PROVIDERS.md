# 🔌 PROVIDERS.md — Model & API Configuration

> *How to connect your Scout and Specialist to inference backends — local, cloud, or hybrid.*

---

## ⚠️ Security Rule #1

**Never hardcode API keys. Ever.**

Not in your source files. Not in a comment. Not "temporarily". If you commit a key to a public repo, assume it's compromised within minutes — bots scrape GitHub continuously for exactly this.

Use a `.env` file or environment variables. The `.gitignore` in this repo already excludes `.env` — but that only works if you actually use `.env`.

---

## 🤖 Recommended Models

These are the tested, recommended models for this architecture. Both are available in GGUF format for llama.cpp-compatible runtimes.

### Scout Model
| Property | Value |
|---|---|
| **Model** | `Qwen2.5-0.5B-Instruct` |
| **Format** | GGUF |
| **Quantization** | Q4_K_M (recommended) |
| **RAM footprint** | ~400–600 MB |
| **HuggingFace** | [Qwen/Qwen2.5-0.5B-Instruct-GGUF](https://huggingface.co/Qwen/Qwen2.5-0.5B-Instruct-GGUF) |
| **Why this model?** | Extremely fast on CPU, good at structured output (JSON), small enough to keep warm permanently |

### Specialist Model
| Property | Value |
|---|---|
| **Model** | `Llama-3.2-3B-Instruct` |
| **Format** | GGUF |
| **Quantization** | Q4_K_M (recommended) |
| **RAM footprint** | ~2.2–2.8 GB |
| **HuggingFace** | [bartowski/Llama-3.2-3B-Instruct-GGUF](https://huggingface.co/bartowski/Llama-3.2-3B-Instruct-GGUF) |
| **Why this model?** | Strong reasoning and code generation at 3B scale, well-quantized community versions widely available |

> **Q4_K_M explained:** This quantization scheme compresses model weights from 16-bit floats to ~4-bit integers using a "K-means" grouping method. It hits the best tradeoff between RAM savings and quality loss — roughly 10–15% performance drop vs. the full model, with a 4x reduction in file size.

---

## ☁️ Cloud API Providers (Free Tiers Available)

If you're not running local inference, these providers support the recommended models with free-tier access — ideal for testing, development, or bridging when local hardware is insufficient.

### Groq Cloud
- **URL:** [console.groq.com](https://console.groq.com)
- **Why use it:** Absurdly fast inference — often 200–400 tokens/second. The best option for testing your swap logic without a local GPU.
- **Free tier:** Generous rate limits on most open models.
- **Best for:** Specialist model testing, latency benchmarking.
- **Supported models:** Llama 3.2 3B, Llama 3.1 8B, Mixtral, Gemma, and more.

```bash
# Example Groq API call (curl)
curl https://api.groq.com/openai/v1/chat/completions \
  -H "Authorization: Bearer $SPECIALIST_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama-3.2-3b-preview",
    "messages": [{"role": "user", "content": "Explain mmap in 2 sentences."}]
  }'
```

### Hugging Face Inference API
- **URL:** [huggingface.co/inference-api](https://huggingface.co/inference-api)
- **Why use it:** Direct access to nearly any open-source model on HF Hub without downloading it locally.
- **Free tier:** Shared inference (slower), or upgrade to serverless for priority.
- **Best for:** Scout model testing, rapid prototyping with unusual models.
- **Supported models:** Basically everything on the Hub — Qwen, Llama, Mistral, Phi, etc.

```bash
# Example HF Inference API call (curl)
curl https://api-inference.huggingface.co/models/Qwen/Qwen2.5-0.5B-Instruct \
  -H "Authorization: Bearer $SCOUT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"inputs": "Classify this query: complexity high or low? Query: What is 2+2"}'
```

### Local Node via Ollama (PC Bridge)
- **URL:** [ollama.com](https://ollama.com)
- **Why use it:** Run both models on your PC and expose them over local Wi-Fi. Your phone acts as a thin client — zero local inference, full model capability.
- **Free tier:** It's local. It's free forever.
- **Best for:** Development when your phone is on the same network as your dev machine. Also useful as a fallback when the phone overheats.
- **How it works:** Ollama runs an OpenAI-compatible REST server at `http://LOCAL_IP:11434`. Your mobile app calls it exactly like a cloud API.

```bash
# On your PC: pull and serve models
ollama pull qwen2.5:0.5b
ollama pull llama3.2:3b
ollama serve  # Starts server on 0.0.0.0:11434

# Test from your phone's browser or curl
curl http://192.168.x.x:11434/api/generate \
  -d '{"model": "llama3.2:3b", "prompt": "Hello"}'
```

> **Note:** For Ollama to be reachable from your phone, both devices must be on the same Wi-Fi network, and your PC firewall must allow port 11434.

---

## 🗝️ Environment Setup

Create a `.env` file in the root of your project. This file is already listed in `.gitignore` — **do not remove it from `.gitignore`**.

```env
# ==============================================
# Mobile LLM Orchestrator — Environment Config
# DO NOT COMMIT THIS FILE TO VERSION CONTROL
# ==============================================

# --- Scout Model ---
# Used for intent classification (0.5B–1B model)
# Get from: https://console.groq.com or https://huggingface.co/settings/tokens
SCOUT_PROVIDER=groq                          # Options: groq | huggingface | local
SCOUT_API_KEY=your_scout_api_key_here
SCOUT_MODEL_ID=llama-3.2-1b-preview         # Model ID as named by your provider

# --- Specialist Model ---
# Used for complex tasks (3B+ model)
# Get from: https://console.groq.com
SPECIALIST_PROVIDER=groq                     # Options: groq | huggingface | local
SPECIALIST_API_KEY=your_specialist_api_key_here
SPECIALIST_MODEL_ID=llama-3.2-3b-preview    # Model ID as named by your provider

# --- Local PC Bridge (Ollama) ---
# Only needed if SCOUT_PROVIDER or SPECIALIST_PROVIDER is set to "local"
LOCAL_IP=192.168.x.x                        # Your PC's local IP address
LOCAL_PORT=11434                            # Default Ollama port
```

### Loading the `.env` in your code

**Node.js / React Native:**
```javascript
import dotenv from 'dotenv';
dotenv.config();

const scoutKey = process.env.SCOUT_API_KEY;
const specialistKey = process.env.SPECIALIST_API_KEY;
```

**Python:**
```python
from dotenv import load_dotenv
import os

load_dotenv()
scout_key = os.getenv("SCOUT_API_KEY")
specialist_key = os.getenv("SPECIALIST_API_KEY")
```

**Android (Kotlin) — use BuildConfig instead of .env:**
```kotlin
// In local.properties (gitignored):
// scout_api_key=your_key_here

// In build.gradle:
// buildConfigField "String", "SCOUT_API_KEY", "\"${localProperties['scout_api_key']}\""

val scoutKey = BuildConfig.SCOUT_API_KEY
```

---

## 🔄 Provider Switching Logic

Your orchestrator should support runtime provider switching. Here's a minimal example:

```javascript
function getProviderConfig(role) {
  const provider = role === 'scout'
    ? process.env.SCOUT_PROVIDER
    : process.env.SPECIALIST_PROVIDER;

  const configs = {
    groq: {
      baseURL: 'https://api.groq.com/openai/v1',
      apiKey: role === 'scout' ? process.env.SCOUT_API_KEY : process.env.SPECIALIST_API_KEY,
    },
    huggingface: {
      baseURL: 'https://api-inference.huggingface.co/models',
      apiKey: role === 'scout' ? process.env.SCOUT_API_KEY : process.env.SPECIALIST_API_KEY,
    },
    local: {
      baseURL: `http://${process.env.LOCAL_IP}:${process.env.LOCAL_PORT}/v1`,
      apiKey: 'ollama', // Ollama accepts any non-empty string
    },
  };

  return configs[provider] || configs.groq;
}
```

---

## 📦 Downloading GGUF Model Files

For local/on-device inference, you need the actual `.gguf` files. Place them in the `/models` directory (gitignored).

```bash
# Using huggingface-cli (pip install huggingface_hub)
huggingface-cli download Qwen/Qwen2.5-0.5B-Instruct-GGUF \
  qwen2.5-0.5b-instruct-q4_k_m.gguf \
  --local-dir ./models

huggingface-cli download bartowski/Llama-3.2-3B-Instruct-GGUF \
  Llama-3.2-3B-Instruct-Q4_K_M.gguf \
  --local-dir ./models
```

> **On slow/mobile connections:** GGUF files are large (300MB–3GB). Download over Wi-Fi, not mobile data. Consider downloading on a PC and transferring via USB.
