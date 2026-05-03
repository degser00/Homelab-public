# AI Node Hardware Considerations

## Overview
This document summarizes practical findings for running local LLM workloads (coding, agents, reasoning) on homelab hardware.

Focus areas:
- GPU vs iGPU performance
- Model sizing and quantization
- Multi-GPU architecture
- Ollama behavior and constraints

---

## 1. Key Insight

Performance is primarily constrained by:
- Memory bandwidth (not just capacity)
- VRAM availability (for full model residency)
- KV cache growth (context-dependent)

Not by:
- Total system RAM alone

---

## 2. iGPU (AMD AI 370) vs Dedicated GPU (R9700)

### iGPU Characteristics
- Shared system RAM (DDR5)
- Limited bandwidth
- Partial GPU offload
- Frequent RAM spill

### R9700 Characteristics
- 32GB dedicated VRAM
- High bandwidth (GDDR6)
- Full model residency possible
- Stable performance under load

---

## 3. Estimated Performance (tokens/sec)

| Model | iGPU (baseline) | R9700 (1 GPU) | R9700 (2 GPUs) |
|------|----------------|--------------|----------------|
| Qwen3-Coder 30B | 3–6 tok/s | 25–45 tok/s | n/a |
| DeepSeek-Coder 33B | 2.5–5 tok/s | 22–40 tok/s | n/a |
| Qwen3 14B Q4 | 12–22 tok/s | 65–100 tok/s | n/a |
| Phi4 14B | 10–18 tok/s | 55–90 tok/s | n/a |
| 70B Q4 | 1–3 tok/s | 8–15 tok/s | 18–35 tok/s |

### Interpretation
- 30B models: ~6–10x improvement
- 14B models: ~5–7x improvement
- 70B models: only viable with multi-GPU

---

## 4. Model Strategy

### Recommended roles

| Role | Model |
|-----|------|
| Coding | Qwen3-Coder 30B |
| Debugging | DeepSeek-Coder 33B |
| Planning | 70B model (DeepSeek / Qwen / Llama) |
| Agents / orchestration | Phi4 14B |

### Key principle
Use multiple specialized models instead of one large model.

---

## 5. Multi-GPU Architecture

### Example (4 GPUs)

| GPUs | Purpose |
|-----|--------|
| GPU 0–1 | 70B planner |
| GPU 2 | Coding model |
| GPU 3 | Debug/review model |

### Important
- Do NOT expose all GPUs to one Ollama instance
- Partition GPUs per service

---

## 6. Ollama Behavior

### Model placement
- Fits in one GPU → uses one GPU
- Does not fit → spreads across ALL visible GPUs

### Implication
- Must isolate GPUs per Ollama instance

---

## 7. Concurrency (Ollama)

### Supported
Yes, multiple prompts can be processed simultaneously

### Tradeoffs
- Each request consumes KV cache memory
- Performance drops per request
- Context usage multiplies

### Recommendation

| Workload | Parallel setting |
|--------|-----------------|
| 70B planner | 1 |
| 30B coding | 1 |
| 30B debugging | 1 |
| 14B agents | 1–2 |

---

## 8. Quantization Strategy

### Model weights
- Q4_K_M → best balance
- Q5_K_M → higher quality, slightly slower
- Q8 → rarely worth it

### KV cache
- q8_0 → default
- q4_0 → for large context

---

## 9. Hardware Scaling

### Threadripper considerations
- Threadripper PRO required for high PCIe lanes
- Up to 128 lanes (PCIe 5.0)
- Typical boards support 4–7 GPUs

### Power considerations
- ~300W per GPU
- 4 GPUs ≈ 1200W
- Full system ≈ 1500W+

---

## 10. Practical Limits

### iGPU
- Works for experimentation
- Not ideal for sustained 30B usage

### 16GB GPU
- Ideal for 14B models
- Limited for 30B

### 32GB GPU
- Ideal for 30B models
- Enables real workflows

### Multi-GPU
- Required for 70B
- Enables agent-based architectures

---

## 11. Final Recommendations

### Minimal effective setup
- 1× 32GB GPU
- 30B model

### Advanced setup
- 4× 32GB GPUs
- 70B planner + 2× 30B agents

### Design principle
- Prioritize bandwidth over RAM
- Prefer multiple specialized models
- Avoid single large model for all tasks

---

## Summary

The key upgrade path is not bigger models, but:
- Running mid-size models fully on GPU
- Separating workloads across GPUs
- Using the right model for each task

This enables a scalable local AI system rather than a single-node inference setup.

