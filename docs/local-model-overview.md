# Local Model Setup Overview (AMD AI 370 / 96GB RAM)

## Summary Recommendations

- Keep both DeepSeek models:
  - `deepseek-r1:32b` → reasoning / root-cause analysis
  - `deepseek-coder:33b` → code writing / fixing
- Use Qwen models for Dyad (tools/agent workflows)
- Avoid adding more overlapping models

---

## Model Comparison Table

| Model | Type | Best Use Case | Strengths | Weaknesses | Dyad Compatible | Keep? | Est. Speed (tok/s) |
|---|---|---|---|---|---|---|---|
| `qwen3-coder:30b` | Coding + agent | Dyad coding, debugging | Tool use, structured edits, good coding | Slightly weaker deep reasoning | Yes | Yes | ~2–5 |
| `qwen3:30b` | General + agent | Dyad fallback, reasoning + tools | Stable, good instruction following | Weaker coding precision | Yes | Yes | ~2–5 |
| `deepseek-r1:32b` | Reasoning | Root cause analysis, debugging logic | Deep reasoning, multi-step analysis | Weaker code output quality | No | Yes | ~1.5–4 |
| `deepseek-coder:33b` | Coding | Code fixes, generation | Strong code quality, syntax accuracy | Weak reasoning chains, no tools | No | Yes | ~2–4 |
| `mistral-small3.2` | General + tools | Backup agent model | Good tool/function calling | Not top-tier coding | Yes | Yes | ~3–7 |
| `phi4:14b` | Small reasoning | Fast debugging, lightweight coding | Efficient, good small-model reasoning | Limited depth | Partial | Yes | ~6–12 |
| `gemma3:12b` | General | Chat, summaries | Lightweight, fast | Weak coding | No | Optional | ~8–15 |
| `llama3.2-vision` | Vision | Screenshot/UI analysis | Multimodal | Slow, not for coding | Partial | Yes | ~4–10 |
| `nomic-embed-text` | Embedding | RAG / search | Fast embeddings | Not a chat model | N/A | Yes | Very fast |

---

## Role-Based Usage

### Dyad (agent / tools)
- Primary: `qwen3-coder:30b`
- Fallback: `qwen3:30b`
- Optional backup: `mistral-small3.2`

### Manual Debugging
- Root cause / analysis: `deepseek-r1:32b`
- Code fix / rewrite: `deepseek-coder:33b`

### Lightweight Tasks
- Fast responses: `phi4:14b`
- General chat: `gemma3:12b`

### Special Use
- Vision/screenshots: `llama3.2-vision`
- RAG/embeddings: `nomic-embed-text`

---

## Practical Workflow

1. Use Dyad (`qwen3-coder`) for implementation
2. If result is incorrect:
   - Send problem to `deepseek-r1` → analyze root cause
3. Then:
   - Use `deepseek-coder` → generate clean fix
4. Apply fix back via Dyad

---

## Notes

- DeepSeek models = manual only (no tools)
- Qwen models = agent-compatible (Dyad-ready)
- Setup balance:
  - Qwen = execution
  - DeepSeek = reasoning
