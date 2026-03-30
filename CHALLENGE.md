# 🌍 CHALLENGE.md — The "Resourceful Dev" Open Problems

> *This project was conceived and spec'd from Kathmandu, Nepal — on a phone, on mobile data, in a country where the median dev environment is not a MacBook Pro with gigabit fiber.*
> 
> *These challenges aren't hypothetical. They're the actual constraints we build around. Solve one and you've solved it for millions of developers in similar conditions.*

---

## Why This Document Exists

Most open-source AI projects are built by people with:
- Fast, stable internet
- A powerful local GPU
- No thermal or power constraints
- Unlimited data plans

That's not most of the world.

The Scout & Specialist architecture was designed specifically for **memory-constrained, thermally-limited, bandwidth-scarce** environments. These open challenges represent the hardest unsolved problems in that context — and we think they're worth solving.

If you're building from a similar environment, you already understand these constraints better than most contributors will. That's a superpower, not a disadvantage.

---

## 🟡 Challenge 1: Offline-First Scout

**The Problem:**
The Scout model currently supports cloud API fallback (Groq, HuggingFace). But in areas with intermittent connectivity — or when you want zero latency and zero data usage — the Scout needs to run fully offline, on-device, always.

**The Constraint:**
The app's install footprint needs to stay reasonable. A 400MB bundled model is acceptable. 2GB is not. The candidate model is **TinyLlama-1.1B** (Q4_K_M ≈ ~670MB) or ideally a sub-200MB option.

**The Challenge:**
> Can we build a Scout classifier that runs 100% offline, fits under 200MB as a GGUF, and achieves >90% accuracy on the `high/low` classification task?

**Approaches to explore:**
- Fine-tune a 0.5B model exclusively on the classification task (not general chat) — this could dramatically shrink the required parameter count.
- Use a non-LLM classifier: a small BERT-style model (~50MB) fine-tuned on intent classification might outperform a 0.5B generative model on this specific task.
- Quantize further: Q2_K or Q3_K_S quantization would shrink the model but needs quality evaluation.
- Keyword/heuristic pre-filter: before calling the LLM Scout at all, a simple rule-based system catches obvious `"low"` cases (single words, greetings, pure math) and skips the model entirely.

**Success criteria:** <200MB model file, >90% classification accuracy on a held-out test set, runs on CPU-only Android device.

---

## 🔴 Challenge 2: Thermal Throttling Governor

**The Problem:**
On mobile devices, sustained LLM inference generates significant heat. When the SoC (System-on-Chip) temperature exceeds ~40–45°C, Android's thermal governor starts throttling the CPU — reducing clock speeds to protect the hardware. This causes:
- Token generation rate to drop unpredictably mid-response
- Increased latency and jitter in output
- In severe cases: the OS kills the app entirely

**The Challenge:**
> Build a token generation rate limiter that monitors device temperature in real-time and dynamically adjusts inference speed to keep the device below 40°C.

**Approaches to explore:**

```
Thermal Governor Loop:
  Every N tokens generated:
    1. Read device temperature (see below)
    2. IF temp > 38°C: reduce n_threads by 1, add 50ms delay between tokens
    3. IF temp > 42°C: pause generation entirely for 2s
    4. IF temp > 45°C: abort Specialist, fall back to Scout, warn user
    5. IF temp < 35°C: restore thread count to default
```

**Reading device temperature on Android:**

```kotlin
// Android Thermal API (API level 29+)
val thermalManager = getSystemService(Context.THERMAL_SERVICE) as ThermalManager

thermalManager.addThermalStatusListener(executor) { status ->
    when (status) {
        ThermalManager.THERMAL_STATUS_NONE -> setThreads(fullThreads)
        ThermalManager.THERMAL_STATUS_LIGHT -> setThreads(fullThreads - 1)
        ThermalManager.THERMAL_STATUS_MODERATE -> setThreads(fullThreads - 2)
        ThermalManager.THERMAL_STATUS_SEVERE -> pauseInference()
        ThermalManager.THERMAL_STATUS_CRITICAL -> abortAndEvict()
    }
}
```

**For older Android or React Native:**
Direct temperature file reads (varies by device):
```bash
# Common paths (not guaranteed on all devices):
cat /sys/class/thermal/thermal_zone0/temp  # CPU temp in millidegrees
cat /sys/class/power_supply/battery/temp   # Battery temp
```

**Success criteria:** Device temperature stays under 40°C during a 10-minute sustained inference session, with graceful degradation rather than hard crashes.

---

## 🟠 Challenge 3: Battery SIP Mode

**The Problem:**
Loading the Specialist costs battery — both from the CPU intensive inference and from the increased RAM pressure (which keeps more DIMMs active). On a low-battery device, the last thing a user wants is the app chewing through their remaining 12% to answer a coding question.

**The Challenge:**
> Implement a Battery SIP Mode that prevents the Specialist from loading when the battery falls below a configurable threshold (default: 15%), and degrades gracefully with a user-friendly explanation.

**Implementation:**

```javascript
// React Native
import { NativeModules } from 'react-native'
import DeviceInfo from 'react-native-device-info'

const BATTERY_THRESHOLD = 0.15  // 15%

async function checkBatteryGate() {
  const level = await DeviceInfo.getBatteryLevel()  // Returns 0.0 – 1.0
  const isCharging = await DeviceInfo.isBatteryCharging()

  if (isCharging) return { allowed: true }  // Always allow when plugged in

  if (level < BATTERY_THRESHOLD) {
    return {
      allowed: false,
      reason: `Battery at ${Math.round(level * 100)}%. Specialist disabled to preserve charge. Plug in or use Scout mode.`
    }
  }

  return { allowed: true }
}
```

```kotlin
// Android (Kotlin)
fun getBatteryGate(context: Context): BatteryGateResult {
    val bm = context.getSystemService(Context.BATTERY_SERVICE) as BatteryManager
    val level = bm.getIntProperty(BatteryManager.BATTERY_PROPERTY_CAPACITY)
    val isCharging = bm.isCharging

    return when {
        isCharging -> BatteryGateResult(allowed = true)
        level < 15 -> BatteryGateResult(
            allowed = false,
            reason = "Battery at $level%. Connect charger to enable Specialist mode."
        )
        else -> BatteryGateResult(allowed = true)
    }
}
```

**Extended ideas:**
- **Adaptive threshold:** Set the floor higher (25%) when device is in airplane mode (no charging opportunity nearby).
- **Queuing:** Instead of refusing the task, queue the complex query and execute it when the device is next plugged in.
- **User override:** Let users override the gate with a single tap, but show a projected battery cost ("This will use ~2% battery").

**Success criteria:** Specialist never loads below 15% battery (unless charging), user sees an informative message not a blank error.

---

## 🟢 Challenge 4: Bandwidth-Aware Model Selection

**The Problem:**
When using cloud providers (Groq, HuggingFace), the round-trip latency varies dramatically based on connection quality. On a 2G connection, even a "fast" API call can take 5–10 seconds. On spotty mobile data, streamed responses stall unpredictably.

**The Challenge:**
> Build a connection quality detector that switches between cloud and local inference depending on measured latency and packet loss.

**Approaches to explore:**
- Ping the provider endpoint before each request; if RTT > 500ms, prefer local Ollama bridge or cached Scout response.
- Measure the time between streaming tokens; if a gap > 3s occurs, abort and retry locally.
- Maintain a rolling average of recent API response times and adapt provider selection accordingly.

---

## 🔵 Challenge 5: Context Length Management on Mobile

**The Problem:**
LLMs have a finite context window (typically 2048–8192 tokens for mobile-optimized models). Long conversations fill this window, causing either: silent truncation (model loses early context), or OOM errors (KV cache grows too large for RAM).

**The Challenge:**
> Implement a smart context trimming strategy that preserves the most relevant conversation history within the available token budget.

**Approaches to explore:**
- **Sliding window:** Keep only the last N tokens of conversation history.
- **Summarization:** When context exceeds 80% of the limit, ask the Scout to summarize earlier conversation turns and replace them with the summary.
- **Importance scoring:** Tag messages by type (user instruction, code block, factual answer) and preferentially retain high-importance turns when trimming.

---

## 📬 How to Contribute

1. Pick a challenge.
2. Build a solution — even a partial one or a proof-of-concept is valuable.
3. Open a PR with:
   - Your implementation
   - A brief explanation of your approach and tradeoffs
   - Test results (even informal ones — "ran on Snapdragon 695, 8GB, stable at 38°C for 8 minutes")

**You don't need a high-end device to contribute.** In fact, if you're testing on a mid-range or older device, your results are *more* valuable to this project than someone running it on flagship hardware.

---

## 📍 Context: Why Nepal?

This project was built by a developer in Kathmandu, Nepal — working on a mobile phone with limited data and no dedicated GPU.

That context matters because it shaped every architectural decision:
- The offline-first priority
- The thermal sensitivity
- The obsession with RAM budget
- The cloud-API-as-bridge-not-dependency approach

The best solutions to these challenges will come from developers who live inside the constraints, not around them.
