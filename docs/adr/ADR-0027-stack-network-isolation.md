# ADR-0027: Stack Network Isolation

## Status

Accepted

**Date:** 2026-07-15
**Last updated:** 2026-07-15
**Relates to:** ADR-0017 (Container Management), ADR-0016 (Network Zones and Trust Boundaries)

## Context

Several compose stacks (`n8n`, `ollama`, `openwebui`, `obs-core`, and
originally `llm-a-n5`/`llm-b-b50`) declare a docker network named `ai-net`
as `external: true`, intending it as a shared network so containers could
reach each other directly by container name. This convention was never
codified: none of the 26 ADRs in `docs/adr/` prior to this one reference it,
and it exists only as copy-pasted `networks:` blocks across compose files.

On the real host (CORE-NAS), `ai-net` does not exist — every app runs on its
own isolated `ix-*` TrueNAS network. A compose file declaring `ai-net` as
`external: true` fails outright at `docker compose up` with `network ai-net
declared as external, but could not be found`, unless someone has manually
created it out-of-band first. `llm-b-b50` hit exactly this failure during
deployment (#170) and was fixed by dropping the network dependency entirely
in favor of its already-published host port
(`http://<core-host>:{{LLAMA_ON_B50_PORT}}/v1`, routed externally via
`{{LLAMA_ON_B50_GPU}}.{{MY_DOMAIN}}`) — a pattern already proven working, not a new one
invented for this ADR.

ADR-0017 (Container Management) already diagnoses the underlying problem
this shared network was most likely a workaround for:

> Internal networks are hidden and non-deterministic... n8n, intended as a
> glue/orchestration service, is effectively isolated and requires complex
> workarounds to reach other services

`ai-net` was an attempt to solve that problem by hand — a shared external
docker network so services could bypass per-app network isolation. ADR-0017
already rejected TrueNAS SCALE's native-app model for the same underlying
isolation behavior and adopted Portainer/Compose instead; `ai-net` never
went through an equivalent decision and directly fights the platform's
real, isolated-by-default networking substrate.

## Decision

**Docker networks are stack-local only.** No compose file in this repo
declares or depends on a network shared with another stack's compose file.
Cross-stack and cross-node service access happens over the regular network
via published host ports — the pattern already validated for `llm-b-b50`
(#170): a fixed host port, optionally fronted by Caddy/CoreDNS for a
friendly hostname (`*.{{MY_DOMAIN}}`), with no dependency on any other
stack's compose file or lifecycle.

`llm-b-b50` is the first stack compliant with this policy and serves as the
reference example for any stack that still needs to migrate off `ai-net`.

## Alternatives Considered

- **Keep `ai-net` as a manually pre-created shared network.** Rejected —
  requires an undocumented, host-level manual step before any dependent
  stack can deploy, with no single compose file owning its lifecycle or
  teardown. Directly reproduces the "hidden and non-deterministic internal
  networks" failure mode ADR-0017 already rejected TrueNAS SCALE apps for.
- **Formalize `ai-net` via an explicit `docker network create` step
  documented in this repo.** Rejected — still creates undeclared
  start-order dependencies between otherwise-unrelated stacks (the stack
  that creates the network must come up before any stack that depends on
  it), and still needs a human to remember the step on every clean-host
  bring-up. Published host ports need no such ordering.
- **Docker Compose's `com.docker.compose.network.default` shared bridge
  auto-created per project.** Not applicable — that mechanism scopes a
  network to one compose project by default; achieving cross-stack sharing
  still requires the same `external: true` pattern this ADR rejects.

## Consequences

### Positive

- No undeclared start-order dependency between unrelated stacks — each
  compose file is deployable and tearable-down independently.
- A single compose file fully owns its own network lifecycle; there is no
  shared resource with ambiguous ownership.
- Matches the platform's real substrate: TrueNAS SCALE app networking is
  already isolated-by-default (`ix-*` per-app networks); this stops fighting
  that default instead of hand-rolling a workaround for it.
- Consistent with ADR-0016: cross-service access already goes through
  identity/route-based paths (Caddy/CoreDNS, Tailscale) rather than raw
  network-location trust — stack-local docker networking now matches that
  principle at the compose level, not just at the zone level.

### Negative / Risks

- Published host ports must be tracked and kept from colliding across
  stacks on the same host — there is no docker-network-level namespacing to
  fall back on. Mitigated the same way `llm-b-b50` already documents port
  reuse deliberately (see its README's "Port reuse" section).
- Container-to-container calls that previously resolved by container name
  over `ai-net` now go over the host network and published port instead —
  marginally higher latency and one more hop through the host's network
  stack. Not measured as significant for any current consumer.
- Endpoints reached only via published host port are, by default, reachable
  by anything that can reach that host port — the same trust model
  `llm-b-b50`'s README already flags as an open item (currently
  unauthenticated). This ADR does not itself add authentication; stacks
  that need it should add it individually (e.g. vLLM's `--api-key`).

### Neutral

- Six stacks currently reference `ai-net` in a compose file: `n8n`,
  `ollama`, `openwebui`, `obs-core`, and (until this issue) `llm-a-n5` and
  `llm-b-b50`. `llm-a-n5` and `llm-b-b50` are corrected as part of the same
  change that adds this ADR. `n8n`, `ollama`, `openwebui`, and `obs-core`
  are not currently in active deployment and are explicitly out of scope
  here — they are corrected as part of the ADR-0017 Portainer migration
  work, not this ADR.

## Out of Scope

- Migrating `n8n`, `ollama`, `openwebui`, and `obs-core` off `ai-net` —
  tracked separately under the ADR-0017 Portainer migration.
- Adding authentication to published endpoints — a per-stack concern, noted
  above as a risk but not decided here.

## References

- ADR-0017 (Container Management) — diagnoses the underlying
  service-to-service connectivity problem `ai-net` was an unratified
  workaround for.
- ADR-0016 (Network Zones and Trust Boundaries) — identity/route-based
  cross-zone access, the same principle this ADR applies at the stack
  level.
- Issue #170 / `modules/Core.srv/llm-b-b50` — the precedent this policy
  generalizes from.
