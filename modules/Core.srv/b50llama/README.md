# B50 GPU — LLM Inference via llama.cpp Vulkan

> Implemented: https://github.com/users/{{SUDO_USER}}/projects/4/views/1?pane=issue&itemId=177476037&issue={{SUDO_USER}}%7CHomelab-private%7C103

---

## Hardware
- **GPU**: Intel Arc Pro B50 (16GB GDDR6) — dedicated inference card
- **Host**: Minisforum N5 Pro (Ryzen AI 9 HX PRO 370, 96GB ECC DDR5)
- **iGPU**: Radeon 890M — separate, runs Ollama ROCm at `{{OLLAMA_AMD_HOSTNAME}}` — untouched
- **Endpoint**: `{{LLAMA_ON_B50_GPU}}.{{MY_DOMAIN}}` → TrueNAS port {{LLAMA_ON_B50_PORT}}

---

## TrueNAS 25.10

TrueNAS 25.10 (Fangtooth) ships with the Intel Xe driver and B50 firmware out of the box. No manual firmware install, no systemd workarounds needed — the GPU is ready to use immediately.

---

## Models

### Default — Qwen3-14B Q4_K_M (~9GB VRAM)
**TrueNAS app**: `b50llama-qwen3-14bq4km`  
**File**: `/mnt/Fast/Llama/qwen3-14b-q4_k_m.gguf`  
**Speed**: ~13 t/s  
**Use for**: n8n automation, general assistant, daily tasks, OpenWebUI chat  
**VRAM headroom**: ~6GB free — comfortable

### High Quality — Qwen3-14B Q8_0 (~15GB VRAM)
**TrueNAS app**: `b50llama-qwen3-14b-q8`  
**File**: `/mnt/Fast/Llama/qwen3-14b-q8_0.gguf`  
**Speed**: ~5 t/s  
**Use for**: when output quality matters more than speed  
**VRAM headroom**: ~500MB — tight, don't run anything else on B50

---

## Switching Models

Models cannot run simultaneously — both need the full 16GB.

1. Go to TrueNAS → Apps
2. **Stop** the currently running app
3. **Start** the desired app
4. Wait ~20s for model to load into VRAM
5. Same endpoint `http://truenas-ip:{{LLAMA_ON_B50_PORT}}/v1` — no config changes needed anywhere

**Default running**: `b50llama-qwen3-14bq4km` (Q4_K_M)

---

## Compose (Q4_K_M — default)

```yaml
services:
  llama-b50:
    image: ghcr.io/ggml-org/llama.cpp:server-vulkan
    restart: unless-stopped
    devices:
      - /dev/dri:/dev/dri
    volumes:
      - /mnt/Fast/Llama:/models
    ports:
      - "{{LLAMA_ON_B50_PORT}}:{{OLLAMA_B50_PORT}}"
    command: >
      --model /models/qwen3-14b-q4_k_m.gguf
      --host 0.0.0.0
      --port {{OLLAMA_B50_PORT}}
      --n-gpu-layers 999
      --ctx-size 8192
```

---

## Useful Commands

```bash
# Check VRAM usage (B50)
b50vram

# Add to ~/.zshrc — persists across reboots
alias b50vram='sudo grep -E "visible_avail|visible_size" /sys/kernel/debug/dri/0/vram0_mm'

# Check which model is loaded and on GPU
docker ps | grep llama
docker logs <container-name> 2>&1 | grep -i "vulkan"

# Test inference
curl http://localhost:{{LLAMA_ON_B50_PORT}}/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"qwen3-14b","messages":[{"role":"system","content":"/no_think"},{"role":"user","content":"say hi"}],"max_tokens":50}'

# Look for "predicted_per_second" in the timings field to check tokens/sec
```

---

## Notes

- **Thinking mode**: Qwen3 has built-in chain-of-thought reasoning that consumes tokens silently. Disable it by adding `/no_think` as system prompt, or `"thinking": false` in the request body. Leave it on for complex reasoning tasks.
- **Model stays in VRAM** as long as the container is running — no cold load per request
- **Driver**: Intel Xe (`xe` module) — loaded automatically by TrueNAS 25.10, no manual setup needed
- **Image**: `ghcr.io/ggml-org/llama.cpp:server-vulkan` — note ggml-org not ggerganov, repo moved
- **Q4_0 does not exist** in the official Qwen3 repo — Q4_K_M is the equivalent
- **VRAM sysfs**: `mem_info_vram_used` (AMD path) does not work on Xe — use `/sys/kernel/debug/dri/0/vram0_mm`

---

## Connections

| Service | URL |
|---|---|
| llama.cpp API | `http://truenas-ip:{{LLAMA_ON_B50_PORT}}/v1` |
| External endpoint | `http://{{OLLAMA_B50_HOSTNAME}}/v1` |
| OpenWebUI | port 31028 |
| n8n | port 30109 — OpenAI node, base URL = `http://{{OLLAMA_B50_HOSTNAME}}/v1`, key = `none` |
| Ollama ROCm (890M) | `{{OLLAMA_AMD_HOSTNAME}}` port 30068 — separate, untouched |
