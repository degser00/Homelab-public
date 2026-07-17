# Caddy (Public Reverse Proxy / TLS Termination) — DNS.box Deployment

## Overview

Caddy is the public-facing reverse proxy for `*.{{MY_DOMAIN}}`. It terminates
TLS (via Cloudflare DNS-01 ACME challenges) and routes:

- `www.{{MY_DOMAIN}}` → Ghost (`web:{{PORT}}`)
- `*.{{MY_DOMAIN}}` → a per-subdomain port on `{{TS_NAS_NAME}}`, resolved through
  an inline Caddyfile `map` block (subdomain label → backend port), with a
  redirect-to-`www` fallback for unmapped subdomains

Runs on the `DNS.box` VM (`HP800a.srv` appliance), alongside CoreDNS
(`../coredns/`). Deployed via Portainer per ADR-0017.

**Config delivery:** the Caddyfile is embedded inline in `compose.yaml`'s
top-level `configs:` block (Compose Spec `configs.content`) rather than
bind-mounted from a sibling `Caddyfile` on disk. Editing the config and
applying it are now both done in Portainer's stack editor — no SSH file edit
+ manual container restart. See **Verification** below for this mechanism's
confirmation status; it is **not yet confirmed working on this Portainer
instance** (see status and fallback there).

## Hardware

CPU-only; no GPU/driver/render-node involved. Runs as a standard container
on the `DNS.box` VM.

## Pre-deployment checklist

- [ ] **Pin `CADDY_VERSION`.** The image tag is guarded
  (`caddy:${CADDY_VERSION:?...}`) because the current stable Caddy release
  could not be confirmed against Docker Hub during authoring (no live
  registry access at authoring time). Look up the current stable tag at
  https://hub.docker.com/_/caddy/tags, then set `CADDY_VERSION` in the
  stack's environment (e.g. `2.8.4`) before deploying. Compose will fail
  loudly with this message if you forget.
- [ ] **Verify inline `configs.content` deploys cleanly on this Portainer
  instance, in standalone mode, before trusting this stack.** This is the
  core open item from issue #168 — see Verification below for exact steps
  and what to do if it fails.
- [ ] **Known pre-existing gap (not introduced by this change): confirm
  `CLOUDFLARE_API_TOKEN` actually reaches the Caddy process.** The Caddyfile
  references `{env.CLOUDFLARE_API_TOKEN}` for the DNS-01 challenge, but
  `compose.yaml` has no `environment:`/`env_file:` entry wiring it in — only
  Portainer stack-level env vars used for *compose-file interpolation* would
  reach it, and only if explicitly mapped. If cert issuance is failing,
  check this first. Out of scope for this conversion (config-delivery
  mechanism only); tracked as a known gap, not fixed here.

## Clean-host prerequisites

- Docker + Portainer agent already running on `DNS.box` (ADR-0017).
- The `infra-net` external Docker network must already exist.
- A `./data` directory (relative to the stack's working directory) must
  exist and be writable — this is where Caddy persists issued
  certificates/OCSP state across redeploys. This is unchanged by the inline
  `configs:` conversion; only the Caddyfile delivery mechanism changed.

## Stand-up procedure

1. In Portainer, open the `caddy` stack (or create it fresh, pointing at
   this `compose.yaml`, if standing up new).
2. Set `CADDY_VERSION` (and confirm `CLOUDFLARE_API_TOKEN` wiring — see
   checklist above) in the stack's environment variables.
3. Deploy/update the stack.
4. To change routing (e.g. add a subdomain to the port map), edit the
   `configs.caddyfile.content` block directly in Portainer's stack editor
   and redeploy — this recreates the container with the new config, no SSH
   access needed.

## Verification

### 1. Portainer `configs.content` support (issue #168's required check)

**Status: not yet confirmed — this must be done against the live Portainer
instance before this stack is trusted in production.**

This repo/PR was authored in a sandboxed environment with no network route
to the homelab's Portainer instance (and no external web-research access in
this run), so the "deploy a throwaway test stack and confirm" check the
issue calls for could not actually be performed here. What follows is the
best-effort reasoning behind implementing inline `configs.content` as the
primary approach anyway, plus exact steps to close the loop.

- The filed bug (portainer/portainer#12253, `"Additional property content
  is not allowed"`) is reported against **Swarm** stacks. Portainer's Swarm
  stack path runs config through Portainer's own (separately maintained,
  historically lagging) schema/converter before submitting to the Swarm
  API — that's the plausible source of the rejection.
- Portainer's **standalone** ("Regular"/non-Swarm) Compose stacks are
  deployed by invoking the real Docker Compose engine (CLI plugin or
  bundled library) against the stack file, the same as running
  `docker compose up -d` directly — not Portainer's own parser. Compose
  Spec `configs.content` has been supported by Docker Compose since 2.23.1.
  On that basis, standalone mode plausibly does not hit the same rejection
  — but this depends on the Compose engine version bundled with/available
  to this specific Portainer install, which is unconfirmed.
- **Required step before relying on this:** deploy this stack (or a
  throwaway copy) via Portainer's stack editor on the `DNS.box` endpoint and
  confirm it deploys without a schema-validation error. Record the outcome
  on issue #168 — #169 (CoreDNS conversion) depends on this result and
  should not repeat the check.

### 2. If inline `configs.content` fails on this Portainer version

Fall back to a **Git-repo-backed Portainer stack with polling-based
auto-redeploy**, keeping the Caddyfile as a bind-mounted file (as today) but
sourcing the stack from this repo instead of a hand-edited on-disk copy:

- Point the Portainer stack at this repo, subpath
  `modules/HP800a.srv/DNS.box/caddy`, with a **pull-interval / polling**
  auto-update configured (Portainer periodically fetches from GitHub on its
  own schedule).
- **Use polling, not a webhook.** A webhook would have GitHub push into the
  homelab to trigger a redeploy — that conflicts with ADR-0009's stated
  principle that GitHub must never be the party capable of *triggering
  execution* against homelab infrastructure (stated there for image-update
  detection specifically, but the principle — detection/action must
  originate from inside the homelab, not be pushed from GitHub — applies
  here too). Polling keeps the homelab as the initiator; a webhook would not.
  Flagging this per CLAUDE.md's instruction to note ADR conflicts rather
  than silently deviate.
- Editing the Caddyfile then becomes: commit the change to this repo, wait
  for (or manually trigger) Portainer's next pull, confirm redeploy. Still a
  single Portainer-mediated action, just pull-based instead of inline-edit.
- If this fallback is adopted, revert `compose.yaml` to the bind-mount form
  and update this README's "Config delivery" note accordingly.

### 3. Functional verification (routing/behavior unchanged)

This conversion is config-delivery-mechanism-only; the Caddyfile content
embedded in `compose.yaml` is byte-for-byte the content confirmed live on
the host (per issue #168's comment), not the previously-stale checked-in
copy. After deploying:

- `curl -sv https://www.{{MY_DOMAIN}}` → expect Ghost response, valid cert.
- `curl -sv https://<mapped-subdomain>.{{MY_DOMAIN}}` for a few entries in the
  port map (e.g. `n8n`, `drive`) → expect proxied response from
  `{{TS_NAS_NAME}}:<port>`.
- `curl -sv https://<unmapped-subdomain>.{{MY_DOMAIN}}` → expect a 302 to
  `https://www.{{MY_DOMAIN}}`.
- `docker logs caddy` → confirm no ACME/DNS-01 errors (see the
  `CLOUDFLARE_API_TOKEN` gap noted above — a silent cert-issuance failure
  is the likely failure mode to watch for, not a routing failure).

### Detecting a silent failure

Unlike a CPU-fallback-masquerading-as-GPU scenario, a bad `configs.content`
deploy fails loudly at `docker compose`/Portainer parse time (the
`"Additional property content is not allowed"` error), not silently at
runtime — that error output itself is the primary signal to watch for
during verification.

## Benchmark methodology + results

Not a compute-bound service; no throughput/latency benchmark has been run.
Scaffolded per repo convention:

- **Method (TBD):** TLS handshake + first-byte latency for a mapped
  subdomain, measured from inside `infra-net` and from a Tailscale client.
- **Results:** not yet collected.

## Connections

| From | To | Purpose |
|---|---|---|
| Internet (Cloudflare) | `caddy:443` | Public HTTPS ingress for `*.{{MY_DOMAIN}}` |
| `caddy` | `web:{{PORT}}` | Ghost (`www.{{MY_DOMAIN}}`) |
| `caddy` | `{{TS_NAS_NAME}}:<mapped-port>` | Per-subdomain backend routing |
| `caddy` | Cloudflare API | DNS-01 ACME challenge (`CLOUDFLARE_API_TOKEN`) |

## DIUN note

Per ADR-0009, DIUN's watch list is auto-populated from Portainer's
aggregated container view — no manual watch-list entry is needed in this
repo for the `caddy` image.
