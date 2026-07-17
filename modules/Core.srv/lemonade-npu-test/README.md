# lemonade-npu-test — Lemonade Server, FastFlowLM/NPU backend

> Part of epic #124, per [ADR-0024](../../../docs/adr/ADR-0024-vllm-serving-stack-n5pro.md)
> (LLM serving architecture). ADR-0024's dual-vLLM primary decision is
> unaffected by this document — see "Verdict" below.

## Status: CLOSED (2026-07-15) — reference stack, not deployed

**Gate #125 fails, and the answer is final, not "revisit later."** Lemonade
Server itself works fine on this host. The FastFlowLM NPU backend
specifically does not, for a host firmware/driver reason that isn't fixable
without an unsupported TrueNAS modification — and no investigated use case
actually needs the NPU even if it did work. This directory is kept as a
working, documented reference so this investigation doesn't have to be
repeated if the question comes up again. **It is not deployed, not routed to
any DNS/proxy, and not assigned an `llm-<letter>-<hardware>` name.**

## Verdict

- ✅ **Lemonade Server**: deploys, serves the OpenAI-compatible API, pulls
  models, runs inference on CPU/GPU without issue.
- ❌ **FastFlowLM NPU backend**: model load fails validation against the
  host's NPU firmware/driver — see "Root cause" below.
- ❌ **No use case needs the NPU** even setting the firmware issue aside —
  see "Use cases investigated" below.
- **Decision**: dual-vLLM (`llm-a-n5` #126, `llm-b-b50` #127) remains
  primary per ADR-0024, unchanged.

## Investigation history

Two attempts, two different blockers:

1. **2026-07-12, TrueNAS 25.10.3 (kernel 6.12.33)**: failed before Lemonade
   was even involved. `amdxdna` merged upstream in Linux 6.14; this kernel
   predates it by two releases. `modinfo amdxdna` → module not found,
   `/dev/accel/` didn't exist, NPU showed in `lspci -nnk` with no driver
   bound.
2. **2026-07-15, TrueNAS 26.0.0-BETA.2 (kernel 6.18.23-production+truenas)**:
   the kernel blocker is genuinely cleared — `amdxdna` binds, `/dev/accel`
   passthrough works, confirmed via container log (`NPU hardware: Yes`).
   Deploying past that point surfaces a new, different, deeper blocker: the
   FastFlowLM NPU backend rejects the model load. See "Root cause" below.

## Root cause

**Firmware/driver protocol version-coupling gap — confirmed cross-distro,
not TrueNAS-specific.**

Exact versions from `flm validate` inside the container:

```
[Linux]  Kernel: 6.18.23-production+truenas
[ERROR]  NPU firmware version on /dev/accel/accel0 is incompatible.
[ERROR]  amdxdna version 0.1 is incompatible
[Linux]  NPU FW Version: 1.0.0.63   (FLM requires >= 1.1.0.0)
[Linux]  Memlock Limit: infinity
```

The in-tree `amdxdna` driver (v0.1) binds and is detected fine (hence "NPU
hardware: Yes" at startup), but it speaks an older firmware protocol than
FastFlowLM requires. Identical failure independently reported on Fedora 43
and Proxmox (same exact firmware version, `1.0.0.63`, and same error text —
this is evidently a common stock default, not something odd about this
specific TrueNAS build). Root-caused against `amd/xdna-driver#1219` (Ubuntu
25.10, same NPU generation): the fix is a matched driver+firmware pair via
`amdxdna-dkms`, either from AMD/Lemonade's own PPA
(`ppa:lemonade-team/stable`, Ubuntu 24.04/25.10 only) or building from
source.

**Why this isn't pursued on TrueNAS:**

- `amdxdna-dkms` needs to compile against the exact running kernel's
  headers. TrueNAS ships a custom-built kernel
  (`6.18.23-production+truenas`); no guarantee matching headers exist, and
  the PPA targets stock Ubuntu kernels.
- TrueNAS's read-only root means even a successful DKMS build wouldn't
  survive a reboot or update without extra unsupported scaffolding — the
  same constraint that already blocked the GTT-memory kernel-param
  workaround for `llm-a-n5` earlier in this epic.
- **Firmware-only update was explicitly ruled out as unsafe**, not just
  undesirable: `amd/xdna-driver#1219` documents that bumping firmware alone
  (without a matching driver) on an old in-tree driver can make the NPU
  disappear entirely (`firmware is not alive`) rather than just staying
  broken. A Proxmox thread independently confirms the same failure mode from
  a BIOS firmware bump. **Don't attempt a firmware-only fix.**
- **VM passthrough was evaluated as an alternative** (bypasses TrueNAS's
  read-only root via a real, mutable Ubuntu 25.10 VM using the official PPA
  path). IOMMU groups were checked and are clean — NPU (`cc:00.1`) and iGPU
  (`cb:00.0`/`cb:00.1`) each sit in their own isolated group, so passthrough
  is technically viable. **Not pursued**: the iGPU (`renderD129`) would have
  to leave the host to go to the VM, breaking `llm-a-n5`'s current
  container-on-host architecture unless `llm-a-n5` also moved into the VM —
  a real architecture change, combined with "no use case needs it" below,
  not worth the redesign.

## Use cases investigated — all ruled out

| Use case | Verdict | Why |
|---|---|---|
| Coding-agent model (replace/complement `llm-a-n5`) | ❌ Ruled out | FastFlowLM's actual NPU-tool-calling catalog tops out around `qwen3.5-9b-FLM` (9B). `llm-a-n5` already runs an 80B-total/3B-active MoE (Qwen3-Coder-Next) — no NPU-catalog model beats what's already deployed. |
| Immich (photo ML: face/CLIP search) | ❌ Not supported | Immich's ML service supports CUDA, ROCm, MIGraphX, OpenVINO, ARM NN, RKNN. No AMD XDNA/NPU backend exists at all. |
| Frigate (NVR object detection) | ❌ Not supported | Confirmed via Frigate maintainer (GitHub discussion #19181): "No, Frigate has no specific code to take advantage of AMD NPUs at this time." (discussion #20810: NPU object detection is slow and "unlikely to change given the focus on LLMs.") |
| Email archiving/classification (Paperless-ngx + MailArchiver + small LLM) | ❌ NPU not needed | Real use case, but the right-sized model (~2-4B, e.g. Qwen3-4B-Instruct-2507) runs fine on CPU alone at this task's actual volume — doesn't need dedicated accelerator silicon of any kind. |
| Time-sharing the B50 for background tasks | ❌ Not free capacity | `llm-b-b50` reserves VRAM statically (`--gpu-memory-utilization 0.90`) with no runtime CPU-offload fallback — its idle KV-cache headroom is throughput headroom for the loaded model, not free VRAM for a second model. |

## Real FastFlowLM capabilities — not pursued, worth remembering later

None of these map to a current need, but they're genuine, working FLM
capabilities (unlike the above) — listed here so this doesn't need
re-researching if a fitting problem shows up:

- **Whisper transcription** (`whisper-v3-turbo-FLM`) — voice memos, meeting
  recordings, dictation. Probably the best-shaped fit for this hardware's
  actual advantage (low-power, bursty, latency-tolerant) if a transcription
  need ever comes up.
- **Local embeddings** (`embed-gemma-300m-FLM`) — semantic search/RAG over
  local documents, without touching the production GPUs.
- **VLM image captioning** (Gemma3-vision, Qwen3-VL) — ad-hoc single-image
  description/Q&A. Not a Frigate replacement — no real-time multi-camera
  detection capability.
- **Translation** (`translategemma-4b-FLM`).

The NPU's real selling point (extreme power efficiency, ~80 tok/s under 2W
per FastFlowLM's own numbers) matters most on battery-powered hardware.
CORE-NAS is wall-powered with two already-installed, mostly-idle-outside-
bursts GPUs (890M, Arc B50). The constraint the NPU solves for isn't one
this box actually has — not "NPUs are useless," just not useful *here*, for
anything found so far.

## Revisit triggers

Only reopen this gate if one of these happens:

- TrueNAS ships a kernel/firmware pair that clears `flm validate` (i.e.
  `amdxdna` version and NPU firmware version both move to a matched, current
  pair) without requiring an unsupported host modification, **or**
- A concrete use case emerges that specifically needs Whisper transcription,
  local embeddings, or VLM captioning at NPU-appropriate scale (see "Real
  FastFlowLM capabilities" above).

## Hardware

| | |
|---|---|
| Host | CORE-NAS (Minisforum N5 Pro, Ryzen AI 9 HX PRO 370) |
| NPU | XDNA2, 50 TOPS — PCI `cc:00.1`, device `[1022:17f0]`, driver `amdxdna` (v0.1, confirmed binding on kernel 6.18.23), node `/dev/accel/accel0` |
| iGPU (needed for hybrid decode) | Radeon 890M — PCI `cb:00.0`, driver `amdgpu`, render node `/dev/dri/renderD129`. **Same device `llm-a-n5` (#126) uses exclusively in production** — see "Resource conflict with llm-a-n5" below |
| Arc B50 | PCI `c9:00.0`, driver `xe`, `renderD128` — not used by this stack or this epic's NPU path |

### Resource conflict with `llm-a-n5`

ADR-0024 states `llm-a-n5` uses the full 890M iGPU memory pool exclusively
and nothing else may run concurrently on `renderD129`. This stack also needs
`renderD129` (for the iGPU-decode half of the hybrid path it never reached
in practice), so **the two stacks cannot run at the same time**. If
re-deploying this for a future re-check, stop `llm-a-n5` first and expect
its Developer-role consumers (ADR-0026) to lose service for the duration.

## Pre-deployment checklist

This stack is not intended for routine deployment — it's kept for
re-validation if a "Revisit trigger" above occurs. If re-deploying:

1. **Re-run `flm validate` first**, before touching anything else. If
   `amdxdna` version and NPU firmware version both show as compatible, the
   root cause documented above may have actually been resolved — proceed to
   re-test model load. If either still shows "incompatible", stop here; the
   gate still fails for the same reason.
2. **Optionally set `LEMONADE_API_KEY`** in a git-ignored `.env` and
   uncomment its line in `compose.yaml`. Not required to start — this stack
   is normally spun up briefly just to re-run `flm validate`, not left
   running — but Lemonade's own startup log warns unauthenticated
   `/internal/*` control endpoints are exposed to the whole LAN on `0.0.0.0`
   without it, which is real for any deployment window, however short.
3. **Stop `llm-a-n5` first** — see "Resource conflict with llm-a-n5" above.

## Clean-host prerequisites

- TrueNAS 26.0.0-BETA.2 or later, kernel 6.18.23-production+truenas or
  later — confirmed sufficient for `amdxdna` to bind and `/dev/accel` to
  exist (does not by itself mean `flm validate` passes; see "Root cause").
- `renderD129` (890M, `amdgpu`) already working — same prerequisite
  `llm-a-n5` documents, already confirmed on this host.

## Stand-up procedure

```bash
cd modules/Core.srv/lemonade-npu-test
# optional: create .env (git-ignored repo-wide) with:
#   LEMONADE_API_KEY=<a real secret value>
# and uncomment the LEMONADE_API_KEY line in compose.yaml

# stop the production stack first — see "Resource conflict with llm-a-n5"
docker compose -f ../llm-a-n5/compose.yaml down

docker compose up -d
docker compose logs -f lemonade-npu-test

# pull and attempt to load the largest real FLM-catalog model with
# tool-calling support (see "Investigation history" for why this one)
docker exec lemonade-npu-test lemonade pull qwen3.5-9b-FLM
docker exec lemonade-npu-test flm validate
```

Deployment is manual via Portainer (per ADR-0017) — not wired into any
CI/automation path.

## Verification

The 2026-07-15 run confirmed:

1. **Driver/device present**: container log shows `NPU hardware: Yes`.
2. **Model pull**: succeeds cleanly (6/6 files, hash-verified, ~8.5GB for
   `qwen3.5-9b-FLM`).
3. **Model load**: **fails** —
   `FLM NPU validation failed: NPU firmware is incompatible. Kernel version
   is incompatible.` (the separate "Memlock limits are too low" portion of
   this error was cleared by this compose file's `ulimits: memlock: -1/-1`,
   isolating the remaining blocker to firmware/driver only — confirmed
   deterministic across two load attempts, not a transient race).
4. **Hybrid mode engagement / concurrent-request handling**: not reached —
   moot once model load fails.

If re-validating after a "Revisit trigger", re-run all four steps above; a
partial result (e.g. `flm validate` passes but hybrid mode silently falls
back to iGPU-only or CPU) is still a fail.

## Benchmark methodology + results

Not performed — model load never succeeded, so there was nothing to
benchmark. If a future re-check clears `flm validate` and model load
succeeds, compare TTFT under concurrency against `llm-a-n5`'s documented P99
TTFT of 44s at 3 concurrent requests (#174), using the same 3-concurrent
`curl` shape documented in that issue.

## Connections

Standalone stack (ADR-0027 — no shared docker network). No DNS/proxy
exposure.

| Service | URL |
|---|---|
| Lemonade OpenAI-compatible API (host, test only) | `http://<core-host>:30068/v1` |

## DIUN note

Not added to any DIUN watch config — no `diun.yml` exists in this repo
today (ADR-0009 is documented but the watch-list automation isn't
implemented yet; see `llm-b-b50/README.md`'s DIUN note for the same
caveat). Since this stack is not deployed, there is nothing to watch;
revisit if a future re-check leads to actual deployment.
