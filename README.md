# 📱 Mobile LLM Orchestrator: The "Scout & Specialist" Specification

> *A thin-client architecture for running large language models on memory-constrained mobile devices — without killing your RAM, battery, or sanity.*

---

## 🚀 The Vision

Most mobile AI implementations take one of two bad paths:

- **Tiny model, always on** → Fast but dumb. Can't handle anything nuanced.
- **Big model, always loaded** → Capable but the phone becomes a hand warmer and your RAM screams.

This project proposes a third path: **intelligent model swapping**.

The idea is simple — not every query needs a PhD. Most of what users ask ("set a timer", "summarize this", "what does this word mean") is shallow. Only a fraction of real-world queries genuinely require deep reasoning. So why pay the full RAM and thermal cost for a 3B+ model sitting loaded 100% of the time?

This spec outlines the **Scout & Specialist** pattern: a dual-model orchestration system that keeps a lightweight model warm at all times, and only pulls in the heavy lifter when the task actually demands it.

---

## 🧠 The Architecture

```
User Input
    │
    ▼
┌─────────────────────────────┐
│        INTENT ROUTER         │   ← Always running
│   Scout Model (0.5B–1B)     │
│   Classifies: Simple/Complex │
└──────────────┬──────────────┘
               │
     ┌─────────┴──────────┐
     │                    │
  Simple              Complex
     │                    │
     ▼                    ▼
Scout handles      Cold-Swap Triggered
directly           Specialist loaded
(fast, cheap)      (3B+, on-demand)
```

### The Scout (0.5B – 1B Model)

- **Always warm in memory.** Loads at app start, never unloads.
- **Role:** Intent classification + handling simple queries directly.
- **Latency target:** < 200ms first token.
- **Recommended model:** `Qwen2.5-0.5B-Instruct` (GGUF, Q4_K_M quantization)

The Scout is your front desk. It reads every incoming message, decides what it's dealing with, and either handles it itself or escalates to the Specialist. It should be fast and cheap enough to run on the CPU without significant thermal or battery impact.

### The Specialist (3B+ Model)

- **Cold by default.** Not loaded until the Scout flags `"complexity": "high"`.
- **Role:** Code generation, creative writing, deep analysis, multi-step reasoning.
- **Load time target:** < 3 seconds (mmap-based, see `SWAP_LOGIC.md`)
- **Recommended model:** `Llama-3.2-3B-Instruct` (GGUF, Q4_K_M quantization)

The Specialist is your on-call expert. They don't sit in the office burning electricity — they get called in when the problem actually needs them.

---

## 📊 RAM Budget (8GB Target Device)

| Component | Status | RAM Usage (Est.) |
|---|---|---|
| Android OS + System | Always On | ~2.5 GB |
| Background Apps | Always On | ~0.5 GB |
| Scout (Q4_K_M) | Warm | 400 – 600 MB |
| Specialist (Q4_K_M) | Cold (Swappable) | 2.2 – 2.8 GB |
| App Runtime + Buffers | Active | ~300 MB |
| **Total Headroom** | **Safe** | **~1.4 – 2.0 GB** |

> ⚠️ **Note:** These are estimates based on GGUF Q4_K_M quantization. Actual usage varies by device, OS version, and what else is running. Always profile on your target hardware.

---

## 🗂️ Repository Structure

```
mobile-llm-orchestrator/
├── README.md           ← You are here (The Manifesto)
├── PROVIDERS.md        ← Model sources & API key setup
├── SWAP_LOGIC.md       ← The technical blueprint for swapping
├── CHALLENGE.md        ← Open problems for contributors
├── .gitignore          ← What NOT to commit (important!)
├── models/             ← Place your .gguf files here (gitignored)
└── src/                ← Implementation (yours to build)
```

---

## ⚙️ Key Design Principles

### 1. The Device Never Chokes
The Scout's RAM footprint is small enough that the OS can always breathe. We maintain a minimum ~1.5GB headroom at all times to prevent aggressive paging and jank.

### 2. Cold Loads Over Hot Swaps
The Specialist is never "hot-swapped" — it's cold-loaded via `mmap` when needed and evicted after idle timeout. This is safer and more predictable than trying to keep both models warm simultaneously.

### 3. Thermal Awareness is Not Optional
On mobile, thermal throttling is a first-class problem. The swap logic includes temperature and battery gates — see `CHALLENGE.md` for the open problems here.

### 4. Open-Source & API-Key-Safe
No keys are ever hardcoded. The project supports both local inference (via Ollama bridging) and cloud API providers (Groq, Hugging Face). See `PROVIDERS.md`.

---

## 🏁 Getting Started

1. **Clone the repo**
   ```bash
   git clone https://github.com/your-username/mobile-llm-orchestrator
   cd mobile-llm-orchestrator
   ```

2. **Set up your environment**
   Copy the `.env` template from `PROVIDERS.md` and fill in your keys.

3. **Download your models**
   Pull the recommended GGUF models from Hugging Face and drop them in `/models`. They are gitignored by default — see `.gitignore`.

4. **Read the blueprint**
   Start with `SWAP_LOGIC.md` to understand the cold-swap sequence before writing any implementation code.

5. **Pick a challenge**
   Check `CHALLENGE.md` for open problems. This project was built in a resource-constrained environment — contributions that solve real-world constraints are the most valued.

---

## 🤝 Contributing

This spec is intentionally implementation-agnostic. It can be built on:
- **Android (Kotlin/Java)** using llama.cpp via JNI
- **React Native** using a native module bridge
- **Flutter** via FFI bindings to llama.cpp

PRs that add implementation examples for any of these are very welcome.

---

## 📜 License

MIT — use it, fork it, build on it. Attribution appreciated but not required.
