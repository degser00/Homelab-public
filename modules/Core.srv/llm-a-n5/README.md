# llm-a-n5 — Qwen3-Coder-Next (AWQ 4-bit) on Radeon 890M (gfx1150/ROCm)

**Status: deployed and confirmed working.** The original full-bf16 model
(`Qwen/Qwen3-Coder-Next`, ~159GB) could never fit on this iGPU's shared-memory
pool — see "Model" below for what changed. Root-cause detail for every fix in
this document is in issue #174; the initial build spike is in #126/#154/#156.

Part of [ADR-0024](../../../docs/adr/ADR-0024-vllm-serving-stack-n5pro.md)
(LLM serving architecture). Maps to the Developer role endpoint in
[ADR-0026](../../../docs/adr/ADR-0026-development-pipeline.md).
Primary implementation per issue #126 — gate #125 (NPU passthrough) failed at
the TrueNAS host kernel level, so this is the committed path, not a fallback.

---

## Hardware

| | |
|---|---|
| Host | CORE-NAS (Minisforum N5 Pro, Ryzen AI 9 HX PRO 370) |
| GPU | Radeon 890M iGPU — **gfx1150** (Strix Point), NOT gfx1151/Strix Halo |
| PCI address | `cb:00.0` |
| Render node | `/dev/dri/renderD129` (confirmed via #125 device audit) |
| Driver | `amdgpu` |
| Memory | Unified DDR5, shared with system RAM and ZFS ARC/L2ARC on the same host — no fixed VRAM carve-out |

The Intel Arc B50 at `renderD128` is a **different GPU and backend** (Vulkan via
Xe driver, own `llm-b-b50` stack, #127). Nothing in this stack should reference
`/dev/dri` broadly — only `renderD129`.

## Image

`rocm/vllm:rocm7.13.0_gfx1150_ubuntu24.04_py3.13_pytorch_2.10.0_vllm_0.19.1` — an
official AMD-pushed tag on Docker Hub, built for **gfx1150**. Confirmed working
on-host, but **`HSA_OVERRIDE_GFX_VERSION=11.5.0` is required** — this stack
previously assumed the official image needed no gfx override; that assumption
was wrong in practice and has been corrected (see `docker-compose.yml`).

## Model

`bullpoint/Qwen3-Coder-Next-AWQ-4bit` — a 4-bit AWQ quant of the same 80B-total
/ 3B-active MoE architecture as `Qwen/Qwen3-Coder-Next`, ~40GB at 4-bit vs.
~159GB at the original bf16 (confirmed via the HF model card: 80B params,
BF16). The bf16 checkpoint is not a memory-tuning problem — no viable GTT
ceiling on a 96GB host fits it once ZFS ARC headroom is accounted for (tried up
to a 64GB GTT ceiling; still OOM'd deeper into layer construction). The AWQ
quant fits comfortably within the 64GB GTT ceiling this host is configured
for, with ~14 GiB left over for KV cache (33x concurrency at 16,384
tokens/request, per engine logs).

Loads via `CompressedTensorsWNA16MoEMethod`; confirmed working on
gfx1150/ROCm7.13 — this combination wasn't previously confirmed anywhere
findable at authoring time, noted here for future reference.

No revision/commit SHA is currently pinned for this HF repo (unlike the image
tag, which is pinned). Not currently blocking — the checkpoint is small and
stable — but if reproducibility becomes a concern, pin an explicit commit SHA
the same way the image tag is pinned.

## Out of scope

DNS/proxy (#128), the `llm-b-b50` stack (#127), changes to existing stacks.

---

## Pre-deployment checklist

- [ ] **Confirm numeric GIDs for `render`/`video`** on the actual host:
  `getent group render` and `getent group video`. `docker-compose.yml` is
  pinned to `107`/`44` respectively (confirmed on CORE-NAS at authoring time)
  — GIDs are **not guaranteed stable across hosts or reinstalls**; update
  `group_add` if they differ.
- [ ] **Confirm the `ttm.pages_limit`/`ttm.page_pool_size` kernel parameters are
  applied** (see "Clean-host prerequisites" below) — `dmesg | grep -i "GTT
  memory ready"` after reboot, not just the sysfs param file (sysfs echoes the
  *requested* value even if the driver didn't apply it).
- [ ] Confirm actual free-memory headroom on CORE-NAS before starting — the
  96GB pool is shared with ZFS ARC/L2ARC, which grows to fill available RAM by
  default. Check `arc_summary` (or equivalent ZFS ARC stats) alongside
  `free -h` immediately before first start.

## Clean-host prerequisites

1. **GTT memory ceiling**, set via TrueNAS SCALE's `kernel_extra_options` (NOT
   `/etc/modprobe.d` + `update-initramfs` — TrueNAS root is read-only, that
   path fails outright). `amdgpu.gttsize=<N>` alone is deprecated on current
   kernels and silently has no effect; `ttm.pages_limit`/`ttm.page_pool_size`
   (in 4KB pages — GiB × 262144) are the parameters that actually apply:

   ```bash
   midclt call system.advanced.update '{"kernel_extra_options": "amdgpu.gttsize=65536 ttm.pages_limit=16777216 ttm.page_pool_size=16777216"}'
   # reboot required
   ```

   This reserves 64GB of GTT out of the same 96GB pool ZFS ARC uses — a
   deliberate middle ground. The AWQ model only needs ~40GB weights, so there's
   room to shrink to `ttm.pages_limit=12582912 ttm.page_pool_size=12582912`
   (48GB) if ARC/pool performance degrades noticeably at 64GB.

2. Numeric GIDs for the `render`/`video` groups — see "Pre-deployment
   checklist" above.

## Stand-up procedure

1. Complete "Clean-host prerequisites" above (kernel params + reboot, GID
   confirmation).
2. Deploy via Portainer per ADR-0017 (stack from this repo path) — do not
   deploy manually outside Portainer's tracked stacks.
3. **Budget 10-12 minutes for cold start.** Weight load takes ~71s; engine init
   (KV cache creation + warmup) clocked at 250.58s in engine logs, before
   `Application startup complete`. Curl/health-checks during this window will
   hang or connect-then-idle — this is expected, not a hang. `restart:
   unless-stopped` means a crash during this window will retry automatically;
   watch the container restart count if startup seems to never complete.

## Verification

```bash
# GPU should show as detected and in use — 890M only, not the B50
docker exec llm-a-n5 rocm-smi --showuse

# vLLM's own health/model listing. Use 127.0.0.1 or the host's real IPv4
# address, not `localhost` — if `localhost` resolves IPv6-first on the client,
# you'll hit Docker's [::]  listener even though nothing listens there
# internally, and get an intermittent "Connection reset by peer" (fixed for
# the container side by the IPv4-only port publish, but client-side DNS
# resolution order can still bite depending on client config).
curl http://127.0.0.1:30068/v1/models

# Minimal completion test
curl http://127.0.0.1:30068/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llm-a-n5",
    "messages": [{"role": "user", "content": "Reply with exactly: ok"}],
    "max_tokens": 10
  }'
```

Check engine logs for two known-untuned paths on this GPU/quant combo — both
are expected, not errors, but explain the throughput ceiling in the benchmark
section below:

- `Using default MoE config... Config file not found at
  .../E=512,N=512,device_name=AMD_Radeon_890M_Graphics,dtype=int4_w4a16.json`
  — no pre-tuned Triton MoE kernel config exists for this GPU/quant combo yet.
- `Cannot use ROCm custom paged attention kernel, falling back to Triton
  implementation` — no optimized ROCm attention path is available here either.

A single `EngineCore died unexpectedly` crash was observed once in the wild
(restart count 0→1 via `restart: unless-stopped`), not reproducible on a clean
rerun immediately after, and hasn't recurred since. Best guess: transient
jitter shortly after cold start (TunableOp GEMM autotuning possibly still
settling). Worth knowing about if it happens again, not currently chased
further.

## Benchmark methodology + results

Tool: `vllm bench serve`, random dataset, 512 input / 256 output tokens.
**Must pass `--tokenizer` explicitly** — `--model` alone resolves to the
served alias (`llm-a-n5`), not a real HF repo, and 401s trying to fetch it
from the Hub:

```bash
vllm bench serve \
  --base-url http://127.0.0.1:30068 \
  --model llm-a-n5 \
  --tokenizer bullpoint/Qwen3-Coder-Next-AWQ-4bit \
  --dataset-name random \
  --random-input-len 512 \
  --random-output-len 256 \
  --num-prompts 10 \
  --max-concurrency 1   # rerun with --num-prompts 30 --max-concurrency 3 for the second row
```

| Metric | Concurrency 1 | Concurrency 3 | Change |
|---|---|---|---|
| Successful / Failed | 10 / 0 | 30 / 0 | — |
| Output throughput | 9.97 tok/s | 15.62 tok/s | **1.57x**, not 3x |
| Total throughput (in+out) | 29.92 tok/s | 46.87 tok/s | 1.57x |
| Mean TTFT | 1297 ms | 6950 ms | 5.4x worse |
| Median TTFT | 1337 ms | 3482 ms | 2.6x worse |
| **P99 TTFT** | 1363 ms | **44,348 ms** | **32x worse** |
| Mean TPOT | 95.6 ms | 165.5 ms | 1.7x worse |
| Mean ITL | 95.2 ms | 164.9 ms | 1.7x worse |
| GPU KV cache usage | ~0.4% | ~1.3% | still trivial |

### Conclusion: memory-bandwidth-bound, not capacity-bound

KV cache usage stayed under 1.5% even at concurrency 3 (pool is 14 GiB /
152,864 tokens — nowhere close to exhausted), so this isn't a capacity
ceiling. Throughput scaling is nowhere near linear with concurrency (1.57x for
3x the load), pointing at the 890M's GTT-backed shared memory bandwidth as the
bottleneck. Consistent with the untuned-kernel-path log lines in
"Verification" above — every request pays the generic-kernel cost, and
concurrent requests compound that by contending for the same bandwidth rather
than getting real parallelism.

### Concurrency behavior — single-consumer only

**This deployment is not intended to serve multiple concurrent coding
agents.** A P99 TTFT of 44s at concurrency 3 is not workable for interactive
agent tool-calling loops — one agent could stall nearly a minute waiting on
first token while two others are in flight.

- Treat `llm-a-n5` as a **single-agent/session-at-a-time endpoint** for
  anything latency-sensitive.
- If concurrent multi-agent serving is needed, route the extra load to
  `llm-b-b50` (dedicated VRAM, no shared-bandwidth contention with this GPU)
  rather than expecting `llm-a-n5` to absorb it.
- Background/batch-tolerant agent work that can tolerate degraded latency is
  the only case where sharing this endpoint across concurrent callers is
  reasonable.
- `llm-b-b50` has not yet been benchmarked for its own concurrency ceiling —
  worth doing before routing multi-agent load there under the assumption it
  scales better; tracked as a possible follow-up, not covered by this
  document.

A future win, if AMD/vLLM tooling supports it: generate a tuned Triton MoE
kernel config for this specific GPU+quant combo (see the untuned-path log
lines in "Verification") — would likely raise the throughput ceiling without
touching the bandwidth-bound conclusion above.

---

## Environment variables reference

Set in `docker-compose.yml`:

| Variable | Value | Why |
|---|---|---|
| `HIP_VISIBLE_DEVICES` / `ROCR_VISIBLE_DEVICES` | `0` | Scopes the container to the single AMD GPU visible to it (only `renderD129` is passed through, so this is belt-and-suspenders, not strictly load-bearing). |
| `HSA_OVERRIDE_GFX_VERSION` | `11.5.0` | Required in practice for this image to correctly target the 890M — see "Image" above. |
| `PYTORCH_TUNABLEOP_ENABLED` | `1` | Autotunes ROCm GEMM kernels on first load — slow first start, faster steady-state throughput. |
| `PYTORCH_ALLOC_CONF` | `expandable_segments:True` | Reduces allocator fragmentation loading the large MoE checkpoint. |

---

## Connections

| Consumer | URL | Notes |
|---|---|---|
| Host-published | `http://127.0.0.1:30068/v1` (or CORE-NAS's real IPv4) | IPv4-only publish — see "Verification" for why `localhost` can be unreliable client-side. Standalone stack, no shared docker network (ADR-0027) — this published port is the only way consumers reach it. |
| External | `{{OLLAMA_AMD_HOSTNAME}}` (Caddy/CoreDNS, `modules/HP800a.srv/DNS.box/caddy/Caddyfile`) | Host port `30068` deliberately reused from the previous Ollama stack — the 890M can only run one container at a time regardless of port, so Ollama and `llm-a-n5` were already mutually exclusive. Reusing the port makes stopping this stack and starting the old Ollama stack a no-touch fallback for URL consumers. |

**What the port/URL contract does *not* guarantee** — confirm before treating
a swap between backends as no-touch:

1. **`model` field in the request must match what's actually being served.**
   Ollama answers to its own pulled tag; `llm-a-n5` serves `llm-a-n5`
   (`served-model-name`). A client with a hardcoded model string breaks on
   swap in either direction.
2. **Tool-calling response shape/reliability differs between backends** even
   though both claim OpenAI-compatible tool support — a known rough edge
   between Ollama and vLLM specifically. This is load-bearing for agent
   traffic (ADR-0026, Hermes-class agents) hitting this endpoint — verify
   before relying on a silent swap under agent traffic, not just interactive
   chat.

## Update review

This stack sits on fast-moving ground: official `rocm/vllm` gfx1150 support is
recent, and AMD is pushing new ROCm/vLLM version combinations for this chip
roughly monthly or faster. There is no repo-wide update-detection process
actually running (DIUN is documented in ADR-0009 but not implemented for this
stack yet — see "DIUN" below), so nothing prompts a recheck automatically —
run this checklist manually, periodically.

**Last reviewed: 2026-07-15** (working AWQ config landed, #174; supersedes the
2026-07-13 image-tag-pin-only review from #154/#156).

1. **Image tag.** Check the currently pinned tag
   (`rocm/vllm:rocm7.13.0_gfx1150_ubuntu24.04_py3.13_pytorch_2.10.0_vllm_0.19.1`,
   see "Image" above) against `rocm/vllm`'s tags on Docker Hub for a newer
   `gfx1150`-specific release. Update the "Last reviewed" date above
   regardless of outcome.
2. **What "worth upgrading" means here.** A newer tag existing is not by
   itself a reason to move. Take a new tag only if it fixes a known issue with
   the current one, measurably improves throughput or TTFT, or the current
   tag is deprecated/pulled from Docker Hub — not automatically on every push,
   given how fast this image moves.
3. **Tuned MoE kernel config availability.** Re-check whether vLLM/AMD tooling
   has published a tuned Triton MoE kernel config for this GPU+quant combo
   (see "Verification" and "Benchmark methodology + results" above) — would
   raise the throughput ceiling without any compose change.
4. **TrueNAS 26 / NPU path (#125).** Check whether TrueNAS 26 (or a point
   release carrying a kernel ≥6.14) has shipped. Gate #125 failed specifically
   because TrueNAS 25.10.3's kernel (6.12.33) is below the `amdxdna` driver's
   ≥6.14 requirement — not because the NPU hybrid approach itself was flawed;
   NPU-prefill + iGPU-decode hybrid mode (faster TTFT) was the original
   preferred path before it was blocked at the kernel level. If a qualifying
   kernel has shipped, re-attempt the NPU passthrough validation from #125. If
   it succeeds, that reopens the primary-vs-fallback question this epic
   settled in dual-vLLM's favor by necessity — not something dual-vLLM won on
   merit — so it's worth revisiting rather than treating as closed.

No automation, cron job, or DIUN dependency for this section — this is a
documented manual procedure, consistent with ADR-0017/CLAUDE.md's
manual-deployment stance. If/when repo-wide DIUN automation (ADR-0009) is
eventually built and reconciled, item 1 above can be simplified to point at it
instead; item 4 (TrueNAS GA / NPU recheck) has no DIUN equivalent and stays
manual regardless.

## DIUN

`rocm/vllm:rocm7.13.0_gfx1150_ubuntu24.04_py3.13_pytorch_2.10.0_vllm_0.19.1` needs
a DIUN watch entry per ADR-0009 (no `diun.yml` currently exists in this repo to
add it to yet — flagged in the PR description, unchanged from the prior
review).
