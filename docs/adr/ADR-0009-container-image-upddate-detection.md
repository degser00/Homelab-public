# ADR-0009: Container Image Update Detection (DIUN)

**Date:** 2025-11-13  
**Status:** Accepted  
**Deciders:** Owner  
**Relates to:** ADR-0023 (Slim Log-Based Alerting), ADR-0013 (Host-Level Log Collection), ADR-0017 (Container Management and Control Plane Selection), ADR-0021 (DMZ Node Operational Constraints)
**Note (2026-06-10):** ADR-0006/0007/0008 are superseded by ADR-0023. The DIUN detection flow below is unchanged — it rides the slim pipeline (Alloy → Loki → Grafana alert rules → n8n) defined there.

> This is a sub-decision under ADR-0023's alerting pipeline — ADR-0023 decides the pipe's shape (Alloy → Loki → Grafana alert rules → n8n); this document decides only which image-update-detection tool rides it and how it's populated.

---

## Context

All self-hosted services (n8n, Proxy, Ollama, Open WebUI, Grafana, Alloy, Loki, etc.) are deployed as **Docker containers**.  
These containers must be monitored for **new version releases** so that update *detection* is automated, auditable, and consistent across the entire stack. The update itself — the image/tag bump and container recreation — is never automated; it is a manual, human-performed action (see ADR-0017, and CLAUDE.md's manual-deployment rule).

Key requirements:

- Centralize detection of new image versions (no per-app hacks)
- Emit structured logs for Observability (Loki)
- Enable triggers for **Grafana → n8n** automation, ending at issue creation
- Never auto-apply updates — detection produces an auditable, human-actionable record, nothing more
- Maintain separation of duties (no app performs its own update checks)

Docker itself cannot detect new image versions.  
n8n should not perform this role directly (per ADR-0002).  
Therefore, a dedicated update-detection component is required.

---

## Decision

Adopt **DIUN (Docker Image Update Notifier)** as the **single system of record** for container image update detection.

DIUN will:

- Monitor configured Docker images (tags or digests), per a watch list populated as described in Scope & Placement below
- Detect when a new version is published (via Docker Hub / GHCR)
- Emit structured logs (stdout) describing the detected update
- Feed logs into **Alloy → Loki → Grafana → n8n** for automation

Alloy forwards logs to Loki over **OTLP** (Loki's native OTLP ingestion endpoint, not its older push API). This is the standard wire format for this pipeline, consistent with Alloy already being an OpenTelemetry Collector distribution — not a new decision, just making the existing pipe's transport protocol explicit rather than left to default.

### Update Detection Flow

```
DIUN (new version detected)
        ↓
Alloy (log collection, OTLP)
        ↓
Loki (log store)
        ↓
Grafana (alert rule: "new version available")
        ↓
Webhook → n8n
        ↓
n8n Workflow:
  - Generate diff / version report
  - Create GitHub issue (see below)
  - Emit audit log → Loki
```

DIUN **never applies updates**, and neither does n8n. n8n's role ends at creating a GitHub issue that records the detected update — the same automation boundary used everywhere else in this repo. The image/tag bump and container/stack recreation remain entirely manual, performed in Portainer (per ADR-0017).

The GitHub issue includes a direct deep link to the affected container or stack in Portainer — e.g. `https://portainer.<tailnet>/#!/<endpoint-id>/docker/containers/<container-id>`, or the stack editor when the service was deployed via a Compose stack rather than a standalone container (exact link target TBD at implementation — see Open Implementation Details below). GitHub is a notification and pointer plane only: it never holds credentials or a path capable of triggering execution against the homelab. The human opens the link over Tailscale, authenticated by network access and Portainer login, and performs the update themselves — either re-pulling the same tag, or editing the tag and then recreating/pulling.

#### Exception: TrueNAS-native apps

A small, named set of services remain TrueNAS SCALE native apps rather than Portainer-managed (currently: Tailscale, Immich — see ADR-00XX and #163, the source of truth for this list and the reasoning behind it). For these, n8n's generated issue links to the TrueNAS app UI instead of a Portainer deep link; the exact TrueNAS deep-link format is an open implementation detail (see below). Everything else is assumed migrated to Portainer per ADR-00XX. The watch-list mechanism described in Scope & Placement only covers Portainer-visible services — TrueNAS-native apps outside the named exception list are out of scope for this ADR's detection mechanism until migrated.

#### Open implementation details

- Exact mechanism by which n8n delivers the generated watch-list file to the ctrl node (e.g. Tailscale SSH/SCP, matching the pattern already used for DMZ node management per ADR-0021) — left to the implementing issue, not dictated here
- Whether the deep link targets the Portainer container view or stack view (stack preferred where the service is Compose-deployed, so a tag edit isn't lost on next stack redeploy)
- How n8n resolves a detected image back to its Portainer container/stack ID
- Exact TrueNAS deep-link format for the exception apps above

---

## Scope & Placement

- DIUN runs as a **single instance on the ctrl node**, watching centrally rather than per-node.
- DIUN **joins `obs-net`** so Alloy can collect its logs.
- DIUN's watch list is **populated from Portainer's aggregated cross-node container view**, not hand-maintained: an n8n job periodically queries Portainer's API and writes DIUN's File-provider YAML. DIUN's File provider **hot-reloads** that file — no restart needed.
- This avoids mounting `docker.sock` on every node (only Portainer's agents need that today) and means new stacks are picked up automatically the moment Portainer sees them running — no manual watch-list maintenance.
- Currently watched (via Portainer's aggregated view): core infrastructure (Alloy, Loki, Grafana), platform components (n8n, Proxy, `llm-a-n5`, `llm-b-b50`, Open WebUI), support services (cloudflared, DIUN itself), plus any business apps, databases, or other modules visible to Portainer. The old `Ollama` / `b50llama` stack names are deprecated per epic #124 — `llm-a-n5` and `llm-b-b50` are the endpoints actually watched going forward.
- TrueNAS-native exception apps (see above) are outside Portainer's view and therefore outside this watch-list mechanism until migrated.

---

## Security

- No external access; DIUN does not expose ports.
- DIUN does not mount `docker.sock` and does not access container metadata directly — it reads its watch list from the n8n-generated File-provider YAML only.
- Cloudflare Zero Trust controls **Grafana** exposure only.
- n8n's role is limited to issue creation; it holds no credentials capable of applying an update, and the GitHub issue it creates carries no execution path — only a pointer for the human to follow.

---

## Consequences

### Positive
- ✔ Standardized, predictable update detection for **all** Portainer-visible containers
- ✔ Update events fully observable via Loki logs
- ✔ Detection and issue creation are automated; the update itself remains a manual, auditable action performed in Portainer
- ✔ No app-specific logic; DIUN consolidates update monitoring
- ✔ Single centrally-run DIUN instance, auto-populated from Portainer — no per-node watch-list maintenance, no `docker.sock` sprawl
- ✔ Integrates cleanly with the ADR-0023 slim alerting pipeline

### Negative
- ⚠ Additional container to maintain
- ⚠ The n8n watch-list generator job becomes a dependency — if it stops running, newly deployed stacks silently stop being watched until it recovers
- ⚠ Update frequency depends on registry rate limits
- ⚠ Single DIUN instance on the ctrl node is a single point of detection failure (consistent with ctrl node already being central per ADR-0017)
- ⚠ TrueNAS-native exception apps are not covered until migrated to Portainer

---

## Out of Scope

- **Prometheus / metrics.** ADR-0023 dropped Prometheus and all metrics collection from the observability stack entirely. DIUN does expose a Prometheus metrics endpoint, but it is deliberately unused here, consistent with ADR-0023 — this document does not reintroduce a metrics-based path for update detection or alerting.

---

## Alternatives Considered

### 1. n8n GitHub/Docker API polling  
Rejected — duplicates logic, increases maintenance, spreads update detection across workflows, violates centralized telemetry design.

### 2. Application-native update checkers  
Rejected — inconsistent, unreliable, most apps do not emit update logs.

### 3. No update automation  
Rejected — incompatible with desired observability-driven update workflow.

### 4. GitHub-native tools (Dependabot, Renovate)  
Rejected on principle, not capability. This repo's GitHub presence is intentionally isolated from the running homelab — documentation and config proposals may flow *out* to GitHub (issues, PRs, the sanitized public mirror per ADR-0019), but GitHub must never be the party that *discovers or proposes* homelab infrastructure changes, nor a party capable of *triggering execution* of one. Detection must originate from the live system's own telemetry (Portainer/Alloy/Loki), with GitHub issues as a downstream, execution-incapable artifact of that — never the reverse. A cloud service inspecting the repo and independently deciding what changes are needed, or a GitHub Action wired to trigger an in-homelab automation, both violate that direction, regardless of how good the tool is.

---

## Future Extensions

- Automated changelog retrieval + diff analysis
- Multi-registry authentication support
- Integration with GitHub Projects for update tracking
