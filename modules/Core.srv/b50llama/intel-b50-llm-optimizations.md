# Intel Arc Pro B50 — LLM Inference Optimizations
**Hardware:** Minisforum N5 Pro · Intel Arc Pro B50 (PCIe) · 16GB GDDR6 VRAM · 224 GB/s bandwidth  
**Stack:** llama.cpp `server-vulkan` via Docker on TrueNAS

---

## Final Optimized docker-compose.yml

```yaml
services:
  llama-b50:
    command: >
      --model /models/openai_gpt-oss-20b-Q4_K_M.gguf --host 0.0.0.0 --port {{OLLAMA_B50_PORT}}
      --n-gpu-layers 999 --parallel 1 --ctx-size 32768
    devices:
      - /dev/dri:/dev/dri
    image: ghcr.io/ggml-org/llama.cpp:server-vulkan
    ports:
      - '{{LLAMA_ON_B50_PORT}}:{{OLLAMA_B50_PORT}}'
    restart: unless-stopped
    volumes:
      - /mnt/Fast/Llama:/models
```

---

## Optimizations Explained

### `--n-gpu-layers 999`
Forces all model layers onto the GPU. Without this, llama.cpp will offload layers to CPU, causing massive slowdowns. Setting to 999 is a safe way to say "use GPU for everything" — it just clamps to however many layers the model actually has.

### `--parallel 1`
By default llama.cpp creates 4 parallel slots (for 4 simultaneous users). Each slot reserves its own KV cache. As a solo user this wastes ~3x VRAM on KV cache you'll never use. Setting to 1 frees that memory for larger context instead.

### `--ctx-size 32768`
Increases the context window from the default 8192 to 32768 tokens (~24,000 words). This is the amount of conversation history the model can "see" at once. Made possible by the VRAM freed from `--parallel 1`. GPT-OSS 20B supports up to 131,072 tokens natively.

### `--host 0.0.0.0 --port {{OLLAMA_B50_PORT}}`
Binds the server to all network interfaces so it's reachable from other machines on the network, not just localhost.

### Model Choice: GPT-OSS 20B Q4_K_M
Q4_K_M is a 4-bit quantization format with medium quality — it uses ~10.8GB of the 16GB VRAM, leaving enough room for KV cache and compute buffers. Q8_0 of the same model would be too large to fit comfortably with any meaningful context window.

---

## VRAM Budget (at 32768 context, parallel 1)

| Component | Memory |
|---|---|
| Model weights (GPU) | 10,740 MiB |
| KV cache | 786 MiB |
| Compute buffer | 398 MiB |
| **Total used** | **~11,924 MiB** |
| **Free headroom** | **~2,748 MiB** |

---

## Benchmarks

All tests run with the same prompt: *"Write a long detailed essay about the history of computers"*

| Model | Quant | Context | t/s | Notes |
|---|---|---|---|---|
| GPT-OSS 20B | Q4_K_M | 32768 | **20–22** | ✅ Winner — fully in VRAM |
| Qwen3 14B | Q4_K_M | 32768 | 12–15 | KV cache eats 5GB at this context |
| Qwen3 14B | Q8_0 | 4096 | 5–6 | Too large — barely fits, tiny context |

**GPT-OSS 20B Q4_K_M is the clear winner** — fastest tokens/sec, largest context, most VRAM headroom.

---

## What Didn't Work

- **SYCL backend** (`server-sycl` image) — failed to start on TrueNAS. SYCL requires Intel oneAPI runtime baked into the container environment with specific device passthrough beyond `/dev/dri`. Not worth pursuing on TrueNAS without bare Ubuntu.
- **Q8_0 quants on 16GB** — any 14B+ model at Q8 leaves under 1GB headroom, limiting context to 4096 and causing instability risk.

---

## Backend Note: Vulkan vs SYCL

The B50 uses Vulkan for inference here. Intel's native SYCL backend would offer ~2x faster token generation by using the XMX matrix engines directly, but requires complex setup not supported cleanly in TrueNAS containers. Vulkan is stable and delivers solid results at ~20 t/s on a 20B model.
