# ⚙️ SWAP_LOGIC.md — The Intelligent Swap Blueprint

> *How the orchestrator decides when to escalate, how it loads the Specialist cleanly, and how it cleans up after itself.*

---

## The Core Idea

The "magic" of this architecture isn't the models — it's the **Intent Router** sitting between them. Every user message passes through the Scout first. The Scout's job is not to answer the question; its job is to decide *who should answer the question*.

Think of it like a hospital triage system. The nurse at the front (Scout) assesses every patient. Most go to the GP. Only the serious cases go to the specialist. The specialist is never in the waiting room burning out — they're called in when actually needed.

---

## Step 1: The Classifier

The Scout model is prompted to return a structured JSON response classifying the incoming message.

### System Prompt (Scout)

```
You are an intent classifier. Your ONLY job is to analyze the user's message and return a JSON object.

Respond with ONLY valid JSON. No explanation, no markdown, no extra text.

Classification rules:
- "low": Greetings, simple questions, factual lookups, short summaries, basic math, yes/no answers
- "high": Code generation, debugging, creative writing, multi-step reasoning, analysis, comparison, anything requiring sustained logic

Examples:
User: "What's the capital of France?" → {"complexity": "low"}
User: "Write a Python function to parse nested JSON" → {"complexity": "high"}
User: "Set a reminder for 3pm" → {"complexity": "low"}
User: "Explain the pros and cons of React vs Vue for a mobile app" → {"complexity": "high"}
```

### Expected Output

```json
{"complexity": "high"}
```

or

```json
{"complexity": "low"}
```

### Why JSON output?

Because string matching is fragile. `"complexity": "high"` can be parsed reliably and extended later (e.g., adding `"confidence": 0.91` or `"category": "code"`). Don't parse free text from your router — it'll break on edge cases.

### Handling Malformed Output

The Scout will occasionally hallucinate extra text. Always parse defensively:

```javascript
function parseComplexity(raw) {
  try {
    // Attempt to extract JSON even if there's surrounding text
    const match = raw.match(/\{.*?"complexity"\s*:\s*"(high|low)".*?\}/s);
    if (match) {
      return JSON.parse(match[0]).complexity;
    }
  } catch (e) {
    console.warn('Scout output malformed, defaulting to low:', raw);
  }
  return 'low'; // Safe default: never escalate on parse failure
}
```

> **Why default to `"low"` on failure?** Because a false negative (treating a complex query as simple) is recoverable — the user sees a weak answer and can rephrase. A false positive (loading the Specialist unnecessarily) wastes RAM and thermal budget. Fail safe.

---

## Step 2: The Routing Decision

```javascript
async function route(userMessage) {
  const scoutResponse = await callScout(userMessage);
  const complexity = parseComplexity(scoutResponse);

  if (complexity === 'high') {
    return await executeSpecialist(userMessage);
  } else {
    return await executeScout(userMessage); // Scout answers directly
  }
}
```

For `"low"` complexity, the Scout handles the response entirely — no model swap, minimal latency, minimal battery draw.

---

## Step 3: The Cold-Swap Sequence

When `complexity === "high"`, the orchestrator executes a four-phase swap:

```
┌─────────────────────────────────────────────────────────┐
│                  COLD-SWAP SEQUENCE                      │
├──────────┬───────────────────────────────────────────────┤
│ Phase 1  │ PAUSE — Halt Scout inference                  │
│ Phase 2  │ FLUSH — Release Scout working memory          │
│ Phase 3  │ MAP   — Load Specialist via mmap              │
│ Phase 4  │ EXECUTE — Run complex prompt                  │
│ Phase 5  │ REVERT — Unload Specialist after idle timeout │
└──────────┴───────────────────────────────────────────────┘
```

### Phase 1: PAUSE
Stop any in-progress Scout inference. On mobile, don't leave threads dangling.

```kotlin
// Android (Kotlin)
scoutJob?.cancel()  // Cancel coroutine if mid-generation
```

```javascript
// React Native
scoutAbortController?.abort()
scoutAbortController = new AbortController()
```

### Phase 2: FLUSH
Free the Scout's KV cache and working buffers to reclaim RAM before loading the Specialist. Don't unload the Scout entirely — just flush its active state.

```kotlin
// Android via llama.cpp JNI
LlamaContext.clearKVCache(scoutContextPtr)
System.gc()  // Hint to GC — not guaranteed but worth calling
```

```javascript
// React Native / JS runtime
if (scoutContext?.dispose) {
  scoutContext.dispose()  // Dispose working tensors, not the model weights
}
// Force GC hint in Hermes (React Native's JS engine)
if (global.gc) global.gc()
```

> **Why not unload the Scout model weights?** Reloading model weights from disk on every swap is slow (500ms–2s). We keep the weights mapped in memory but flush the KV cache (the "working memory" of the current conversation). KV cache is cheap to rebuild; model weights are not.

### Phase 3: MAP (The Key Step)

Use `mmap` (memory-mapped file I/O) to load the Specialist's GGUF weights. This is fundamentally different from a standard `fread()` load:

| Method | How it works | RAM behavior |
|---|---|---|
| `fread()` | Copies file into heap | Full allocation upfront |
| `mmap` | Maps file addresses into virtual memory | OS pages in only what's actually accessed |

With mmap, the OS loads Specialist weights **lazily** — only the pages that the model actually touches during inference are pulled into physical RAM. This makes the initial load faster and the peak RAM usage lower.

```c
// llama.cpp handles this internally when you use llama_load_model_from_file()
// Just ensure the GGUF file is on fast storage (internal flash, not SD card)

struct llama_model_params params = llama_model_default_params();
params.use_mmap = true;   // ← This is the flag. It's true by default in llama.cpp.
params.use_mlock = false; // Don't lock pages — let OS manage eviction

llama_model * specialist = llama_load_model_from_file(model_path, params);
```

```javascript
// In React Native with a llama.cpp native module (e.g., llama.rn)
const specialist = await LlamaContext.create({
  model: `${modelDir}/Llama-3.2-3B-Instruct-Q4_K_M.gguf`,
  use_mmap: true,
  use_mlock: false,
  n_ctx: 2048,     // Context window size — keep this reasonable on mobile
  n_threads: 4,    // Match to your device's efficiency core count
})
```

### Phase 4: EXECUTE
Run the complex prompt through the Specialist. Pass the full conversation context, not just the latest message.

```javascript
const response = await specialist.completion({
  messages: conversationHistory,  // Full context
  max_tokens: 1024,
  temperature: 0.7,
  stop: ['</s>', '<|eot_id|>'],   // Llama 3.2 stop tokens
})
```

### Phase 5: REVERT (Idle Eviction)

The Specialist should not stay loaded indefinitely. Start an idle timer when the Specialist finishes generating. If no new complex query arrives within 60 seconds, evict it.

```javascript
let specialistIdleTimer = null
const IDLE_TIMEOUT_MS = 60_000  // 60 seconds

function onSpecialistFinished() {
  clearTimeout(specialistIdleTimer)
  specialistIdleTimer = setTimeout(async () => {
    await specialist.dispose()
    specialist = null
    console.log('[Orchestrator] Specialist evicted — idle timeout reached')
  }, IDLE_TIMEOUT_MS)
}

// Cancel the timer if a new complex query comes in before timeout
function onNewComplexQuery() {
  clearTimeout(specialistIdleTimer)
}
```

---

## Full Orchestrator Flow (Pseudocode)

```
FUNCTION handleMessage(userMessage):

  1. Call Scout.classify(userMessage)
     → Returns {"complexity": "low"} or {"complexity": "high"}

  2. IF complexity == "low":
       → Call Scout.generate(userMessage)
       → Return response

  3. IF complexity == "high":
       a. CHECK thermal gate: if device_temp > 40°C → fallback to Scout + warning
       b. CHECK battery gate: if battery_level < 15% → fallback to Scout + warning
       c. PAUSE Scout inference
       d. FLUSH Scout KV cache
       e. IF Specialist not loaded:
            → MAP Specialist GGUF via mmap
       f. EXECUTE Specialist.generate(userMessage)
       g. START idle eviction timer (60s)
       h. Return response
```

---

## Performance Targets

| Metric | Target |
|---|---|
| Scout classification latency | < 200ms |
| Specialist cold-load time (mmap) | < 3 seconds |
| Specialist first token (after load) | < 500ms |
| Idle eviction trigger | 60 seconds |
| RAM headroom maintained | ≥ 1.5 GB |
| Device temp abort threshold | 40°C |
| Battery abort threshold | 15% |

---

## Common Failure Modes

| Problem | Cause | Fix |
|---|---|---|
| Scout always returns `"high"` | System prompt too vague | Tighten the classifier prompt with more examples |
| OOM crash during Specialist load | Not enough headroom | Add pre-load RAM check; abort if < 1.5GB free |
| Specialist load takes 10+ seconds | GGUF file on SD card or external storage | Move model to internal flash storage |
| Thermal throttling mid-generation | CPU sustained load | Implement token generation rate limiting (see `CHALLENGE.md`) |
| KV cache blows up context | Conversation too long | Implement context trimming — keep only last N tokens of history |
