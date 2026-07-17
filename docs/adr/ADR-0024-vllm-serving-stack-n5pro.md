# ADR-0024: LLM Serving Architecture on CORE-NAS (N5 Pro iGPU/NPU + B50)

**Status:** Accepted
**Date:** 2026-07-11
**Last updated:** 2026-07-17

## Context

CORE-NAS (Minisforum N5 Pro, Ryzen AI 9 HX PRO 370) has three inference-capable devices:

- **890M iGPU** — currently running Ollama (TrueNAS native app, ROCm), ~49GB unified memory from 96GB system RAM. Silicon is **gfx1150 (Strix Point)** — despite "Ryzen AI" branding shared with Strix Halo (gfx1151), these are distinct dies with different vLLM/ROCm support (see supporting factors below)
- **XDNA2 NPU (50 TOPS)** — currently unused
- **Arc Pro B50 (16GB)** — currently llama.cpp server (Vulkan backend), GPT-OSS 20B Q4_K_M at 20–22 tok/s

The serving layer must feed three consumer types **simultaneously**:

1. n8n automation workflows (ADR-0020 pipeline)
2. Autonomous agents — dev, QA, task manager (Hermes Agent or similar), in a three-stage human-gated pipeline
3. Direct interactive chat via Open WebUI

Ollama serializes requests by default, which disqualifies it as a shared backend. This is the core limitation driving the change.

Supporting factors:

- ROCm 7.2.1 added official full vLLM support for Ryzen APUs, but that support targets the **gfx1151 family (Strix Halo) specifically**. The N5 Pro's 890M iGPU is **gfx1150 (Strix Point)** — a different die under the same "Ryzen AI" marketing family — and has **no official ROCm vLLM support**. The `llm-a-n5` deployment adapts the community gfx1151 build pattern to gfx1150 (`HSA_OVERRIDE_GFX_VERSION` overrides, falling back to a custom `PYTORCH_ROCM_ARCH` build if overrides don't work — see `modules/Core.srv/llm-a-n5/README.md`)
- Lemonade Server (`ghcr.io/lemonade-sdk/lemonade-server`) ships an official Docker image with an OpenAI-compatible API, documented concurrent multi-app integrations (Continue, Open WebUI, n8n), and a FastFlowLM XDNA2 NPU path enabling hybrid NPU-prefill + iGPU-decode
- AMD AI Playbooks document iGPU memory tuning (Adrenalin/BIOS: minimize dedicated VRAM to maximize the shared pool)

## Decision

### Hard requirement (applies to every path)

**Every model endpoint must expose an OpenAI-compatible API and serve multiple consumers concurrently without serializing.** No single-user/single-session serving path is acceptable anywhere in the architecture. Continuous batching (vLLM) or documented concurrent-client support (Lemonade) is the qualifying bar.

### Lemonade+NPU hybrid — attempted, failed (gate #125)

The Lemonade+NPU hybrid path was the originally-planned primary approach: deploy Lemonade Server in Docker on the N5 Pro and attempt NPU passthrough (`/dev/accel`, IOMMU passthrough mode, matched kernel/firmware versions) to enable FastFlowLM's XDNA2 path alongside the 890M iGPU, with the NPU handling prefill and the iGPU handling decode. NPU passthrough was scoped as a stretch goal, not a committed dependency, from the outset.

**Gate #125 failed, in two stages.** A 2026-07-12 attempt failed at the TrueNAS host kernel level (`amdxdna` requires Linux ≥6.14; the host was on 6.12.33). A 2026-07-15 re-test on TrueNAS 26.0.0-BETA.2 (kernel 6.18.23) cleared that blocker — `amdxdna` binds, `/dev/accel` passthrough works — but surfaced a deeper, final blocker: the FastFlowLM NPU backend rejects model load with a firmware/driver protocol mismatch (`amdxdna` driver v0.1 paired with NPU firmware 1.0.0.63; FastFlowLM requires firmware ≥1.1.0.0). The fix is a matched `amdxdna-dkms` build, which needs a writable, persistent kernel-module build environment that TrueNAS's read-only root doesn't offer. Because it was scoped as a stretch goal, this cost only the experiment time — the architecture falls through to the dual-vLLM path below, which is the **implemented primary path**, not a fallback. The XDNA2 NPU remains idle; no investigated use case needs it regardless. Full investigation record, root cause, and revisit triggers: `modules/Core.srv/lemonade-npu-test/README.md`.

### Implemented — dual vLLM instances, one per GPU

Two independent vLLM instances, each on its own GPU (issue #126 / #127, under epic #124):

| Instance | GPU | Model | Consumers |
|---|---|---|---|
| **`llm-a-n5`** | 890M iGPU — gfx1150, `/dev/dri/renderD129`, ROCm | Qwen3-Coder-Next (80B total / 3B active MoE, ~45GB @ Q4, 256K native context, YaRN-extendable to 1M) | Developer/coding agent (ADR-0026) — dedicated capacity, full iGPU memory pool |
| **`llm-b-b50`** | Arc Pro B50, 16GB dedicated VRAM, `/dev/dri/renderD128`, Intel XPU/Level Zero backend | Qwen3-14B AWQ (~9GB weights, 128K native context via YaRN, served at 16K) | Planner, Reviewer, Test Engineer roles (ADR-0026) — shared concurrent endpoint, also serves n8n and Open WebUI |

Neither GPU is split between two vLLM instances — the originally-considered two-instances-sharing-one-iGPU split was superseded once the B50 was confirmed as the second endpoint instead.

`llm-b-b50` replaces the previous `b50llama` (llama.cpp/Vulkan) deployment on the same GPU. `b50llama` is left in place and marked deprecated pending a separate human-executed decommission step (epic #124 / issue #129) — the two stacks cannot run concurrently, since both need the full 16GB of B50 VRAM.

Both stacks were authored without access to the physical host or image registries; unresolved values (image tags, model revisions) are guarded by required env vars (`${VAR:?...}`) per this repo's confirm-before-deploy convention, and must be resolved per each stack's README pre-deployment checklist before first deploy.

### Deployment

Custom Docker containers managed through the existing Docker/Portainer layer (per ADR-0017) — full image version pinning and flag control. Not TrueNAS native apps.

## Alternatives Considered

**Keep Ollama** — Rejected. Default request serialization fails the concurrency requirement. Fine for the single-consumer workloads it served historically; the multi-consumer architecture changes the bar.

**TrueNAS native apps (Ollama or vLLM catalog app)** — Rejected. ADR-0017 already rejected TrueNAS native apps platform-wide (hidden, non-deterministic internal networking; brittle service-to-service connectivity); the current Ollama native app predates that decision, and this migration brings the AI lane into compliance. Additionally, the vLLM catalog app does not support custom image tags, blocking ROCm-tuned builds and the AITER environment tuning gfx1150 needs.

**Single vLLM instance serving all N5 Pro consumers** — Rejected. One instance means one model: either the dev agent's large coding model starves lighter consumers of capacity, or a smaller shared model shortchanges the dev agent. The two-GPU split keeps the heaviest single-stream task (`llm-a-n5`) isolated from the concurrent light traffic (`llm-b-b50`).

**Keep llama.cpp (Vulkan) on the B50** — Superseded. It was retained in an earlier draft of this decision as a dedicated single-consumer lane, but the unified requirement made every endpoint a shared concurrent one, and standardizing on vLLM simplified the consumer-facing contract. Implemented as `llm-b-b50`; the llama.cpp/Vulkan `b50llama` stack is deprecated in place pending decommission (issue #129).

## Consequences

**Positive**
- Concurrent multi-consumer service on both endpoints; no head-of-line blocking between agents, n8n, and interactive chat
- No new hardware required — same OpenAI-compatible contract for all consumers regardless of which GPU serves them
- Full control over image versions, ROCm/XPU builds, and tuning flags
- vLLM exposes native Prometheus metrics (improvement over the Ollama scraping dead end)

**Negative / risks**
- **NPU passthrough failed** (gate #125) — the initial kernel-level blocker was later cleared (TrueNAS 26.0.0-BETA.2, kernel 6.18.23), but a firmware/driver protocol mismatch in the FastFlowLM NPU backend replaced it, unfixable without an unsupported TrueNAS host modification (read-only root blocks the required `amdxdna-dkms` build). The XDNA2 NPU (50 TOPS) sits idle; see `modules/Core.srv/lemonade-npu-test/README.md` for the full investigation record and revisit triggers — out of scope here
- **No official ROCm vLLM support for gfx1150** (Strix Point) — `llm-a-n5` runs on a community-adapted build pattern (HSA_OVERRIDE_GFX_VERSION overrides against the gfx1151/Strix Halo precedent, or a custom PYTORCH_ROCM_ARCH build if overrides don't work); validate correctness and throughput, not just that the container starts — a silent fallback to the eager/reference kernel path would still produce correct output at a fraction of expected throughput
- **`llm-a-n5` uses the full 890M iGPU memory pool exclusively** — nothing else may run concurrently on `renderD129`. CORE-NAS's 96GB pool is shared with ZFS ARC/L2ARC; check `arc_summary` alongside nominal free RAM before assuming headroom
- **`llm-b-b50` is not yet validated against physical hardware** — the B50's history is SYCL failing on TrueNAS and settling on Vulkan (`b50llama`); vLLM on Arc means the Intel XPU/Level Zero backend, a different (and less-trodden) path than Vulkan llama.cpp. Validate before decommissioning `b50llama`; if it fails, `b50llama` remains the fallback until then
- Both stacks were authored without host/registry access — image tags and model revisions are unresolved pending the pre-deployment checklist in each stack's README; do not deploy on assumed values
- Custom containers mean self-maintained upgrade lifecycle (DIUN watch per ADR-0009 mitigates detection)

**Neutral**
- All consumers move to OpenAI-compatible endpoints; n8n, Hermes, and Open WebUI configs must be repointed and re-tested
- DNS: `llm-a-n5` and `llm-b-b50` deliberately reuse the host ports of the stacks they replace (30068 for `ollama`, {{LLAMA_ON_B50_PORT}} for `b50llama`) rather than getting new port assignments, so the existing `{{OLLAMA_AMD_HOSTNAME}}` / `{{LLAMA_ON_B50_GPU}}.{{MY_DOMAIN}}` Caddy/CoreDNS routes (`modules/HP800a.srv/DNS.box/caddy/Caddyfile`) already reach the new stacks with no proxy/DNS change — see #139 and each stack's README ("Port reuse" section) for the fallback rationale and the two things that reuse does *not* guarantee (hardcoded `model` fields, tool-calling response shape across backends). Broader DNS/reverse-proxy exposure beyond these two pre-existing routes is tracked separately (issue #128)
- BIOS/driver VRAM allocation change on the N5 Pro (minimize dedicated VRAM to maximize the shared pool available to `llm-a-n5`)

## Out of Scope

**Dedicated multi-GPU inference node** (R9700, Phase 1 ~Oct 2026 / Phase 2 ~Jan 2027) — separate decision, own ADR when committed. Note for that record: re-verify vLLM issue #40980 (TP=2 deadlock on dual R9700 / gfx1201) before buying the second card. Topology precedent set: ADR-0015 (revised 2026-07-11) defines the AI compute node as bare metal Ubuntu Server, not Proxmox.

**Compliance logging of agent traffic** — ADR-0002 defines a Proxy-based compliance logging layer between consumers and LLM endpoints. That ADR is on hold and not implemented; all consumers call endpoints directly, and this ADR makes no provision for compliance capture. If ADR-0002 is revived, the Proxy would sit in front of the endpoints defined here without changing this decision.

## References

- AMD ROCm 7.2.1 release notes — vLLM Ryzen APU (gfx1151/Strix Halo) support; does not cover gfx1150/Strix Point, the N5 Pro's actual iGPU
- AMD AI Playbooks — vLLM on Ryzen AI Halo
- Lemonade Server — `ghcr.io/lemonade-sdk/lemonade-server`, FastFlowLM XDNA2 hybrid mode (NPU path abandoned — see Decision, gate #125)
- vllm-project/vllm issue #40980 — TP=2 deadlock on gfx1201 (future GPU-node gate)
- Epic #124, issues #125 (NPU passthrough gate, failed), #126 (`llm-a-n5`), #127 (`llm-b-b50`), #128 (DNS/proxy, out of scope here), #129 (`b50llama` decommission)
- `modules/Core.srv/llm-a-n5/README.md`, `modules/Core.srv/llm-b-b50/README.md` — stack-level deployment detail
- ADR-0002 — AI compliance logging (on hold, not implemented)
- ADR-0009 — Container image update detection (DIUN)
- ADR-0015 — Platform topology (AI compute node = bare metal, revised 2026-07-11)
- ADR-0017 — Container management
- ADR-0020 — Agentic news intelligence pipeline
- ADR-0026 — Development pipeline (role-to-endpoint mapping: Developer → `llm-a-n5`; Planner/Reviewer/Test Engineer → `llm-b-b50`)
