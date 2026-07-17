# CLAUDE.md — Homelab Repository Conventions

This repository holds architecture decision records (ADRs), documentation, and
deployment configuration (Docker Compose) for a self-hosted homelab platform.
Claude works here as a docs/config author. A human reviews every PR and performs
all deployments manually — never add deployment automation, CI deploy steps, or
anything that applies changes to live infrastructure.

## ADR rules (strict)

- ADRs live in `docs/adr/`, named `ADR-NNNN-kebab-case-title.md`, numbered
  sequentially. Before creating one, check the highest existing number.
- **Default: edit in place.** Factual corrections, updated values, fixed
  claims — edit the text directly at the point of the error. The PR
  description (already required elsewhere in this file) states what changed
  and why. Git history carries the rest; the document should always read as
  current truth, not an accretion of patches.
- **Exception — whole-decision reversals only:** when the ADR's core
  recommendation itself changes (not a supporting detail), either
  (a) write a new ADR and mark the old one `Status: Superseded by ADR-NNNN`,
  or (b) if staying in the same document, note the change concisely at the
  point of the decision itself — not as a growing stack of blockquotes at
  the top of the file.
- Status vocabulary: Proposed, Accepted, Adopted, Superseded (by ADR-NNNN),
  On hold. Update `Last updated:` on any substantive edit, not just decision
  reversals.
- New ADRs follow the house structure: Status/Date header, Context, Decision,
  Alternatives Considered, Consequences (Positive / Negative-risks / Neutral),
  Out of Scope where relevant, References to related ADRs.
- Cross-reference related ADRs by number. Key standing decisions:
  - ADR-0015: platform topology (AI compute node = bare-metal Ubuntu Server)
  - ADR-0017: containers are managed via Docker + Portainer; TrueNAS native
    apps are rejected platform-wide
  - ADR-0009: image updates are detected (DIUN) but never auto-applied
  - ADR-0024: LLM serving architecture (Lemonade primary / dual-vLLM fallback,
    OpenAI-compatible concurrent endpoints required)

## Hardware & vendor-support claims

- Before citing vendor support claims (ROCm/CUDA compatibility, driver
  support, etc.), verify the exact chip codename against the specific
  CPU/GPU model's spec sheet — do not infer from product-family marketing
  names. "Ryzen AI" / "Strix" branding spans multiple distinct silicon dies
  (e.g. Strix Point gfx1150 vs. Strix Halo gfx1151) with different
  capabilities.

## Repository layout

- Deployment config lives under `modules/`, structured as:
  `modules/<appliance>/<vm>/<service>/compose.yaml` (plus any files that
  service needs). Example: `modules/HP800a.srv/WEB.box/Ghost/`.
  Appliances are physical machines (e.g. `Core.srv`, `HP600a.srv`,
  `HP800a.srv`); VMs/hosts are the `.box` level; services are the leaf.
- Place new stacks at the correct appliance/VM path — never at the repo root
  or in a flat directory.
- Documentation lives under `docs/` (ADRs in `docs/adr/`, networking docs in
  `docs/networking/`).

## Docker Compose conventions

- **Always pin image tags to a specific version.** Never `latest`. New images
  must be added to the DIUN watch config (per ADR-0009).
- The compose file is named `compose.yaml`, one service stack per leaf
  directory as described in Repository layout.
- Containers log to stdout/stderr only (host-level log collection per
  ADR-0013); no logging drivers or sidecar log shippers.
- No TrueNAS-native-app assumptions; stacks must be deployable via Portainer.
- **When a pinned value (image tag, model revision, etc.) cannot be
  confirmed** against a live registry/source during authoring (no network
  access, host unreachable, etc.), do not guess a plausible-looking value.
  Use Compose's `${VAR:?error message}` syntax (or equivalent) so the stack
  fails loudly at `docker compose up` with a clear message, and document the
  exact confirmation steps needed in the README's pre-deployment checklist.
- LLM endpoint names follow `llm-<letter>-<hardware>` — the hardware slot is
  the stable identity; the letter is the instance ordinal on that hardware
  (a = first, b = second, …). The name deliberately drops the serving
  backend (vLLM/Ollama/llama.cpp): backend software and the model running on
  it are both expected to change over time, and may be swapped
  interchangeably on the same hardware. Examples: `llm-a-n5`, `llm-b-b50`,
  `llm-c-9700`, `llm-d-9700`. This is the current convention — if it
  conflicts with newer repo docs, follow the repo. All model endpoints must
  be OpenAI-compatible and support concurrent consumers (ADR-0024).
- **`served-model-name` must match the endpoint name, not the underlying
  model's own name** — e.g. the OpenAI-compatible `model` field a consumer
  requests should be `llm-b-b50`, not `qwen3-14b`, so a consumer configured
  against the endpoint doesn't need reconfiguring when the underlying model
  changes. Document the actual model identity/version in the stack's README
  and container labels instead.
- Include explicit resource/memory limits for LLM serving containers (shared
  iGPU memory pool — see ADR-0024 consequences).

## Service stack README skeleton

Each service stack (`modules/<appliance>/<vm>/<service>/README.md`) should
follow this structure, in order:

1. **Overview** — what the service is, why it's deployed, its role.
2. **Hardware** — device, driver, render node.
3. **Pre-deployment checklist** — unconfirmed values that must be verified
   before `docker compose up` (see the required-env-var guard above).
4. **Clean-host prerequisites** — packages/drivers/config the host needs
   before this stack will run.
5. **Stand-up procedure** — steps to deploy.
6. **Verification** — how to confirm the service is actually working,
   including how to detect a silent fallback (e.g. a CPU fallback
   masquerading as a working GPU deployment).
7. **Benchmark methodology + results** — include this section (scaffolded)
   even if results aren't filled in yet.
8. **Connections** — table of internal/external URLs.
9. **DIUN note** — per ADR-0009.

## Known host quirks

Host-specific gotchas that aren't obvious from spec sheets, recorded here so
they don't have to be rediscovered per-issue:

- **CORE-NAS**: the 96GB RAM pool is shared with ZFS ARC/L2ARC. Nominal free
  RAM is not real headroom — check `arc_summary` before assuming capacity is
  available for a new workload.

Add new quirks here as they're found.

## General

- Keep PRs small and single-purpose: one ADR or one stack per PR.
- PR descriptions state which ADR(s) motivate or are affected by the change.
- If a task conflicts with an accepted ADR, do not silently deviate — note the
  conflict in the PR description and implement per the ADR unless the issue
  explicitly says otherwise.
- Never commit secrets, tokens, API keys, or Tailscale/Cloudflare credentials.
  Reference them as environment variables or external secret files.
- **All `.md` files in this repo are mirrored to a public repo** after
  placeholder substitution (see `.github/workflows/publish.yml`). When writing
  docs: prefer existing `{{PLACEHOLDER}}` tokens already used in neighbouring
  docs over inventing literal values, and if a doc must introduce a new
  sensitive or infrastructure-identifying value (hostname, domain, tunnel ID,
  IP, etc.), explicitly flag it in the PR description as "needs SECRETS_MAP
  entry before merge".
- Documentation tone: concise, factual, no marketing language.
