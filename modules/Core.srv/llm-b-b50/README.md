# llm-b-b50 — Qwen3-14B on Intel Arc Pro B50 (XPU)

> Part of epic #124, per ADR-0024 (LLM serving architecture) and ADR-0026
> (role-based development pipeline). Primary implementation — issue #127.

## Overview

- **GPU**: Intel Arc Pro B50, 16GB dedicated VRAM, PCI `c9:00.0`, driver `xe`,
  render node `/dev/dri/renderD128`. **Not** the 890M iGPU shared pool used by
  `llm-a-n5` (#126) — this card is exclusive to this stack.
- **Model**: Qwen3-14B (AWQ, ~9GB weights), 128K native context via YaRN,
  served at 16K here (see "VRAM sizing" below).
- **Backend**: vLLM, Intel XPU (Level Zero), not ROCm and not the previous
  Vulkan/llama.cpp path.
- **Role**: shared concurrent endpoint for Planner / Reviewer / Test
  Engineer-class consumers (task manager + QA agents, n8n, Open WebUI) —
  ADR-0026's Phase 2 endpoint mapping for those three roles.
- **Replaces** `../b50llama/` (llama.cpp/Vulkan) on this same GPU. That stack
  is left in place and marked deprecated (see its README) — decommissioning
  it is a separate, human-executed step (epic #124 invariants / #129).
  **The two stacks cannot run at the same time**: they contend for the same
  16GB of VRAM. Stop `b50llama` before starting `llm-b-b50`, and vice versa.

## Hard constraint: no runtime CPU-offload fallback

vLLM's XPU backend cannot spill weights or KV cache to host RAM at runtime if
VRAM is exhausted — there is a `VLLM_OFFLOAD_WEIGHTS_BEFORE_QUANT` flag, but
that only applies to staging during on-the-fly quantization, not a runtime
overflow path. If model + KV cache exceeds 16GB, the process OOMs. Treat the
`--max-model-len` / `--gpu-memory-utilization` values in `docker-compose.yml` as
load-bearing, not defaults to casually raise — see "VRAM sizing" and re-run
the benchmark in this README before changing them.

## Pre-deployment checklist

1. ~~**`VLLM_IMAGE_TAG`**~~ — **resolved** (issue #155): `docker-compose.yml`
   now pins `intel/vllm:0.10.2-xpu` directly. See "Why 0.10.2-xpu, not the
   newer 0.21.0 tag" below before considering an upgrade.
2. **`VLLM_MODEL_REVISION`** — must be resolved at deploy time. This stack
   was authored without registry/network access, so the exact commit SHA for
   `Qwen/Qwen3-14B-AWQ` could not be pinned at authoring time. It is a
   **required** env var — `docker compose` refuses to start without it (see
   `docker-compose.yml`'s `${VLLM_MODEL_REVISION:?...}` guard). Create a
   local `.env` next to `docker-compose.yml` (git-ignored repo-wide, same as
   every other module here — it is never committed) with:
   ```
   VLLM_MODEL_REVISION=<resolved Qwen3-14B-AWQ commit SHA>
   ```
   See **"Pinning the model revision"** below for how to resolve the SHA.

Do not deploy on an assumed value — an unpinned model revision fails silently
by drifting to whatever `main` happens to point to later.

### Pinning the model revision

Hugging Face model repos are plain git remotes, so the current `main` commit
SHA can be resolved directly, with no HF tooling or API involved:

```bash
git ls-remote https://huggingface.co/Qwen/Qwen3-14B-AWQ HEAD
```

This prints `<sha>\tHEAD` — the SHA is the value for `VLLM_MODEL_REVISION`.
Alternative, if `huggingface_hub` is available:

```bash
python3 -c "from huggingface_hub import HfApi; print(HfApi().model_info('Qwen/Qwen3-14B-AWQ').sha)"
```

> **Do not use** the previously documented
> `curl -s https://huggingface.co/api/models/Qwen/Qwen3-14B-AWQ | jq -r .sha`
> — verified not working in practice (2026-07-14). `git ls-remote` is the
> reliable path.

For reference only (re-resolve at deploy time, do not blindly reuse): as of
**2026-07-14**, `main` resolved to
`31c69efc29464b6bb0aee1398b5a7b50a99340c3` — a verified README-only commit,
so the pinned tree still carries the original weight files.

If `Qwen/Qwen3-14B-AWQ` does not exist under that exact repo id at deploy
time, find the Qwen team's current AWQ (or equivalent ~Q4) quantization of
Qwen3-14B and update both the repo id in `docker-compose.yml`'s `--model`
flag and this revision accordingly — the model *choice* (Qwen3-14B) is
fixed by issue #127's decision, the exact artifact id/revision is not.

### Why `0.10.2-xpu`, not the newer `0.21.0` tag

`intel/vllm:0.21.0-ubuntu24.04-20260625` is a newer tag on the registry and is
likely a legitimate upgrade path, but no source found during research (issue
#155) explicitly confirms Arc Pro B-series (Battlemage) validation for it.
`0.10.2-xpu` is the tag explicitly named in the joint vLLM/Intel Arc Pro
B-series announcement (which also covers MoE model support, e.g. gpt-oss,
since that release), and independent hands-on testing confirmed
`torch.xpu.is_available()` returns `True` with this tag on B-series hardware.
Pin `0.10.2-xpu` for now. Upgrading to `0.21.0` (or later) is a reasonable
follow-up once someone validates it against the real B50 hardware — see
"Verifying the model loaded on XPU" above — not something to guess into the
pin without that validation.

**Dev-snapshot observation (issue #170, 2026-07-14)**: the `0.10.2-xpu` tag
actually contains a vLLM *dev snapshot*, not a clean 0.10.2 release — startup
reports `vLLM API server version 0.1.dev9453+g1babc91fe.d20251028`. This
explains the CLI drift found during deployment (`--device` is not a valid
argument on this build; `--gpu-memory-util` requires the full
`--gpu-memory-utilization` spelling — see `docker-compose.yml`). Relevant
context for any future tag re-evaluation: the pin is not just a version
number, it's a specific dev commit's CLI surface.

## Update review

A periodic manual check for whether the `0.10.2-xpu` pin above should be
revisited — distinct from the one-time tag research done in #155 that
justified it. No automation, no cron job, no DIUN dependency (there is no
repo-wide DIUN automation actually running today — ADR-0009 is documented but
unimplemented; see the "DIUN" note below). Someone has to do this by hand
periodically.

**Last reviewed:** not yet performed — section added 2026-07-13. The next
review should check `intel/vllm` tags published after that date.

**What to check:**

1. Browse `intel/vllm` tags on Docker Hub for anything newer than the
   currently pinned tag (see `docker-compose.yml`).
2. For each newer tag, look for evidence that it has *explicit* Arc Pro
   B-series (Battlemage) validation — a higher version number alone is not
   sufficient. Specifically look for:
   - Intel or vLLM blog posts / release notes that name Battlemage or Arc Pro
     B-series support directly — the same category of evidence #155 used to
     justify `0.10.2-xpu` over the newer-but-unconfirmed `0.21.0` tag at the
     time.
   - Independent hands-on reports of `torch.xpu.is_available()` returning
     `True` on B-series hardware with that tag.
3. If no such evidence exists for a newer tag, leave the pin as-is and update
   the "Last reviewed" date above.

**If a newer tag with confirmed B-series validation is found:**

- Do not swap the pin as a drive-by change. Re-run "Verifying the model
  loaded on XPU" and the full "Benchmark" methodology below against real B50
  hardware first — a version bump on the XPU backend can change VRAM/KV-cache
  behavior (see "Hard constraint: no runtime CPU-offload fallback" above), so
  the pre-deployment checklist and VRAM-sizing numbers in this README need
  re-validation, not just the tag string.
- Update the "Why `0.10.2-xpu`..." section above with the new pin and the
  evidence that justified it, and update `docker-compose.yml` accordingly.
- Update the "Last reviewed" date above regardless of outcome.

**Retiring this process:** if/when ADR-0009's DIUN automation is actually
implemented and reconciled with this stack, simplify this section to point at
DIUN instead of a manual check.

## Prerequisites (clean host)

The GPU currently sits on TrueNAS Core (temporary coupling per ADR-0015,
pending the future dedicated bare-metal AI compute node). TrueNAS 25.10+
ships the Intel `xe` kernel driver and B50 firmware out of the box — no
manual firmware install or systemd workarounds needed, matching the existing
`b50llama` deployment on the same host.

1. **Kernel driver**: confirm `xe` is loaded and the render node exists:
   ```bash
   lsmod | grep xe
   ls -l /dev/dri/renderD128
   ```
2. **oneAPI / Level Zero runtime**: not required on the host — the
   `intel/vllm` XPU image bundles the oneAPI runtime and Level Zero loader
   internally. The host only needs to expose the render node (`/dev/dri`),
   same passthrough pattern as `b50llama`.
3. **No other container holds the GPU**: stop `b50llama` (or any other
   consumer of `/dev/dri/renderD128`) first — see "Overview" above.
4. **Model cache location**: `/mnt/Fast/Models/llm-b-b50` must exist on the
   host (same fast-storage volume `b50llama` uses for weights) — vLLM will
   populate `HF_HOME` under it on first pull.

## Stand-up procedure

```bash
cd modules/Core.srv/llm-b-b50
# create .env with VLLM_MODEL_REVISION (see checklist above)

docker compose up -d
docker compose logs -f llm-b-b50
```

Deployment itself is manual via Portainer (per ADR-0017) — the `docker
compose` invocation above is what Portainer's stack editor should run under
the hood; this is not wired into any CI/automation path.

## Verifying the model loaded on XPU (not a silent CPU fallback)

vLLM logs its device selection and memory profiling on startup. Confirm both
of the following before trusting the endpoint:

```bash
# 1. Startup logs should show the XPU device, not "cpu"
docker logs llm-b-b50 2>&1 | grep -iE "device|xpu|level.zero|cpu"

# 2. GPU is actually active during a request — B50 VRAM/engine usage should
#    move off idle. TrueNAS Xe debugfs path (same as b50llama uses):
sudo grep -E "visible_avail|visible_size" /sys/kernel/debug/dri/0/vram0_mm

# 3. A real completion, confirming the OpenAI-compatible surface works end to end
curl http://localhost:{{LLAMA_ON_B50_PORT}}/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"llm-b-b50","messages":[{"role":"user","content":"say hi"}],"max_tokens":50}'
```

If step 1 shows no XPU/Level-Zero device reference, or step 3 succeeds but is
implausibly slow (single-digit tok/s on a 14B AWQ model), vLLM has silently
fallen back to CPU — stop the container and check the `ONEAPI_DEVICE_SELECTOR`
env var and `/dev/dri` passthrough before re-attempting.

## VRAM sizing

| Component | Approx. size |
|---|---|
| Model weights (Qwen3-14B, AWQ) | ~9GB |
| `--gpu-memory-utilization 0.90` budget (90% of 16GB) | ~14.4GB |
| Remaining for KV cache + activations | ~5.4GB |
| KV cache per token (40 layers, 8 KV heads, head_dim 128, fp16) | ~0.16MB |
| Approx. total live KV tokens the ~5.4GB budget covers | ~32K tokens (paged across all concurrent sequences, not a fixed per-request slice) |

`--max-model-len` is set to 16384 — the ADR-0020 amendment floor (minimum 16K
for agent tool schemas) — rather than pushed toward the model's 128K YaRN
ceiling, precisely because there's no runtime fallback if concurrent KV usage
from multiple pipeline roles exceeds the ~5.4GB budget above. **Validated
2026-07-14 (issue #170)**: at 3 concurrent 512-token requests, actual GPU KV
cache usage peaked at 4.9% — far below this table's ~5.4GB/32K-token
estimate. The table above is a conservative worst-case budget, not the
observed working set; revisit only if real pipeline traffic (longer prompts,
more concurrent roles) approaches it — see "Benchmark" below.

## Benchmark

**Validated on physical B50 hardware 2026-07-14** (issue #170): deploy → XPU
inference → external route (`{{LLAMA_ON_B50_GPU}}.{{MY_DOMAIN}}`) → single + concurrent-load
benchmarks, all confirmed working. Results below are the eager-mode baseline
(see `--enforce-eager` note in `docker-compose.yml` — torch.compile crashes
engine-core startup on this image/hardware combo) and the number to beat if a
future image resolves that compile crash.

### Method

Single-request baseline:
```bash
curl -w '\n%{time_total}s total\n' http://localhost:{{LLAMA_ON_B50_PORT}}/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"llm-b-b50","messages":[{"role":"user","content":"Write a long detailed essay about the history of computers"}],"max_tokens":512}'
```

Concurrent load, simulating Planner + Reviewer + Test Engineer hitting the
endpoint at once (three concurrent streams, representative prompt lengths —
adjust payload sizes to real pipeline traffic once available):
```bash
for role in planner reviewer test-engineer; do
  curl -s -o "/tmp/${role}.json" -w "${role}: %{time_total}s\n" \
    http://localhost:{{LLAMA_ON_B50_PORT}}/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{"model":"llm-b-b50","messages":[{"role":"user","content":"..."}],"max_tokens":512}' &
done
wait
```

Record tokens/sec from each response's `usage` field (or vLLM's own
`/metrics` Prometheus endpoint) alongside wall-clock latency. The concurrent
run below disabled Qwen3 thinking mode per-request (see "Notes") to keep the
comparison to the solo run apples-to-apples on output token count.

### Results (validated 2026-07-14)

| Scenario | Throughput | Wall time | GPU KV cache usage |
|---|---|---|---|
| Single request, 512 tok | 21.8 tok/s client-side (23.5 engine-reported) | 23.5s | 1.6% |
| 3 concurrent × 512 tok (thinking disabled) | **68.1 tok/s aggregate** (peak) | ~23.4s each (≈ same as solo) | 4.9% peak |

Continuous batching confirmed working: ~3x aggregate throughput at
unchanged per-request latency (`Running: 3 reqs` in engine logs) — this is
the core reason for the llama.cpp → vLLM move per ADR-0024. KV cache
headroom at this load is far above what the "VRAM sizing" table above
estimates; re-run this benchmark with longer prompts / more concurrent roles
before relying on that headroom for heavier real pipeline traffic.

## Connections

This stack is **standalone**: it does not join any shared docker network
(ADR-0027). Consumers (n8n, Open WebUI, future agents) reach this endpoint
via the regular network through the published host port only.

| Service | URL |
|---|---|
| vLLM OpenAI-compatible API (host) | `http://<core-host>:{{LLAMA_ON_B50_PORT}}/v1` — same port `b50llama` used, reused deliberately (see "Port reuse" below); this is also how all consumers (internal or external) reach the endpoint |
| Prometheus metrics (native vLLM) | `http://<core-host>:{{LLAMA_ON_B50_PORT}}/metrics` |
| External (Caddy/CoreDNS) | `{{LLAMA_ON_B50_GPU}}.{{MY_DOMAIN}}` — `modules/HP800a.srv/DNS.box/caddy/Caddyfile` already routes this to port {{LLAMA_ON_B50_PORT}} on `{{TS_NAS_NAME}}`; confirmed working end-to-end (`GET /v1/models`) 2026-07-14 |

DNS/reverse-proxy exposure beyond the existing `{{LLAMA_ON_B50_GPU}}.{{MY_DOMAIN}}` route is
out of scope for this stack (#128). **Note**: this route is currently
unauthenticated and publicly reachable — see "Notes" below.

## Port reuse ({{LLAMA_ON_B50_PORT}}) as a deliberate fallback strategy

Host port `{{LLAMA_ON_B50_PORT}}` is the `b50llama` stack's port, reused here on purpose, not
a naming collision to fix later. The B50 can only run one container at a time
regardless of port — `b50llama` and `llm-b-b50` were already mutually
exclusive before this stack existed (both need the full 16GB VRAM). Reusing
the exact host port means:

- **Trivial fallback**: stop `llm-b-b50`, start `b50llama` — no downstream
  reconfiguration, since the URL consumers hit (`{{LLAMA_ON_B50_GPU}}.{{MY_DOMAIN}}` and the
  raw `{{LLAMA_ON_B50_PORT}}` host port) never changes.
- **Safe incremental testing**: this stack can be brought up, tested, and torn
  down with the known-good `b50llama` stack one command away the whole time.
- Both backends (llama.cpp / vLLM) are OpenAI-compatible at the transport
  level (`/v1/chat/completions`), which is what makes this swap-in-place
  viable at all.

**What the port/URL contract does *not* guarantee** — confirm before treating
the swap as no-touch:

1. **`model` field in the request must match what's actually being served.**
   `b50llama` answers to `qwen3-14b` (its GGUF filename-derived tag); this
   stack serves `llm-b-b50` (`served-model-name`, see `docker-compose.yml`).
   A client with a hardcoded model string breaks on swap in either direction.
   **Current exposure: low, not zero** — Open WebUI (today's only consumer)
   has no hardcoded model string; n8n is a planned consumer but not yet
   active, so this becomes relevant the day it is.
2. **Tool-calling response shape/reliability differs between backends** even
   though both claim OpenAI-compatible tool support — a known rough edge
   between llama.cpp and vLLM specifically, not just theoretical. **Doesn't
   apply yet** — no agents (Hermes-class, ADR-0026) are implemented against
   this endpoint; this becomes load-bearing the moment agent traffic starts,
   since that's precisely when a silent backend-vs-backend tool-call
   formatting difference would surface, likely mid-task rather than at
   smoke-test time.

## Notes

- **Thinking mode**: Qwen3 has built-in chain-of-thought reasoning, and it is
  **on by default** — validated 2026-07-14, a first test consumed ~270 of 512
  output tokens on reasoning before the actual answer. For concurrent
  throughput (per ADR-0024's no-serializing-backends requirement), prefer
  disabling it per-request (`"chat_template_kwargs": {"enable_thinking":
  false}` or the equivalent vLLM request field) for Reviewer/Test Engineer
  traffic where latency matters more than deliberation depth.
- **Auth**: this endpoint is currently **unauthenticated** and reachable both
  on the internal host port and publicly via `{{LLAMA_ON_B50_GPU}}.{{MY_DOMAIN}}`. Consider
  vLLM's `--api-key` flag before agent/n8n traffic starts hitting it
  unattended (issue #170) — not yet done as of this PR; possibly warrants its
  own issue rather than a drive-by change here.
- **DIUN**: this stack's image is not yet in the DIUN watch config (no
  `diun.yml` exists in this repo today — DIUN, per ADR-0009, is managed
  outside this repo). Add `intel/vllm` to whatever DIUN instance watches this
  host's images before/at deployment.
