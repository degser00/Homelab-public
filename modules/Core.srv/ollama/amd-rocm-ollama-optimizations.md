# AMD ROCm Ollama — LLM Inference Optimizations
**Hardware:** Minisforum N5 Pro · AMD Ryzen AI 370 · AMD Radeon 890M iGPU · 96GB DDR5 unified memory (49GB allocated to iGPU)  
**Stack:** Ollama ROCm image via TrueNAS native app

---

## Final Environment Variables

| Variable | Value |
|---|---|
| `HSA_OVERRIDE_GFX_VERSION` | `11.0.0` |
| `OLLAMA_FLASH_ATTENTION` | `true` |
| `OLLAMA_KV_CACHE_TYPE` | `q8_0` |
| `OLLAMA_MAX_LOADED_MODELS` | `1` |
| `OLLAMA_NUM_PARALLEL` | `1` |
| `OLLAMA_KEEP_ALIVE` | `30m` |

---

## Optimizations Explained

### `HSA_OVERRIDE_GFX_VERSION = 11.0.0`
The AMD AI 370's Radeon 890M is a relatively new iGPU that ROCm doesn't always recognize correctly out of the box. This variable overrides the GPU architecture version to `gfx1100` (RDNA3), which the 890M is based on. Without this, ROCm may fall back to CPU inference entirely. This is the single most critical variable for this hardware.

### `OLLAMA_FLASH_ATTENTION = true`
Enables Flash Attention, an optimized attention computation algorithm that processes the KV cache more efficiently. Benefits: reduced memory usage during inference and faster processing of long contexts. Particularly impactful on the iGPU where the 49GB is shared and precious. Off by default in Ollama.

### `OLLAMA_KV_CACHE_TYPE = q8_0`
Compresses the KV cache (attention state storage) to 8-bit instead of 16-bit. This halves the memory used by the KV cache without meaningfully affecting output quality — the KV cache stores intermediate attention computations, not model weights. On a 30B model with a long context this can save several GB of iGPU allocation, allowing larger contexts or bigger models to stay fully GPU-resident.

### `OLLAMA_MAX_LOADED_MODELS = 1`
Prevents Ollama from keeping multiple models loaded simultaneously. By default Ollama may keep a previously used model warm in memory while loading a new one. With 49GB of iGPU memory shared across everything, allowing two large models to co-exist would cause one to spill to system RAM and slow to a crawl.

### `OLLAMA_NUM_PARALLEL = 1`
Limits concurrent request processing to 1. Each parallel slot requires its own KV cache allocation. As a solo user there is no benefit to parallel slots — they only consume memory that could otherwise extend context length.

### `OLLAMA_KEEP_ALIVE = 30m`
Keeps the loaded model in iGPU memory for 30 minutes after the last request. Without this, Ollama unloads the model after a short idle period, causing a slow reload (several seconds) on the next request. 30 minutes is a good balance between responsiveness and not wasting memory overnight.

---

## Memory Layout

The AMD AI 370's 890M iGPU uses unified DDR5 — there is no separate VRAM. The OS allocates a portion of system RAM to the iGPU:

| Pool | Size |
|---|---|
| Total system RAM | 96 GB |
| iGPU allocation | 49 GB |
| Remaining for CPU/OS | ~47 GB |
| Default context window | 262,144 tokens |

The 262K default context is enormous — a result of Ollama calculating available iGPU memory and maximizing context accordingly.

---

## Model Library

Models available on this system, all running GPU-accelerated via ROCm:

| Model | Size | Fits in 49GB iGPU? |
|---|---|---|
| qwen3.6:27b | 17 GB | ✅ Comfortably |
| qwen3.5:27b-q8_0 | 29 GB | ✅ With room |
| laguna-xs.2 | 23 GB | ✅ |
| devstral-small-2 | 15 GB | ✅ |
| devstral | 14 GB | ✅ |
| deepseek-coder:33b | 18 GB | ✅ |
| deepseek-r1:32b | 19 GB | ✅ |
| qwen3-coder:30b | 18 GB | ✅ |
| qwen3:30b | 18 GB | ✅ |
| mistral-small3.2 | 15 GB | ✅ |
| gemma3:12b | 8.1 GB | ✅ |
| phi4:14b | 9.1 GB | ✅ |
| llama3.2-vision | 7.8 GB | ✅ |
| nomic-embed-text | 274 MB | ✅ |

---

## Architecture Note

Unlike a discrete GPU, the 890M iGPU shares the DDR5 memory bus with the CPU. This means memory bandwidth (~100 GB/s effective for iGPU) is lower than a discrete GDDR6 card, but the advantage is capacity — 49GB of effectively "free" VRAM that can run models a discrete 16GB card cannot touch at all.
