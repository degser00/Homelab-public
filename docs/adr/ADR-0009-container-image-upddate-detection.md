# ADR-0009: Container Image Update Detection (DIUN)

**Date:** 2025-11-13  
**Status:** Accepted  
**Deciders:** Owner  
**Relates to:** ADR-0006 (Observability Core), ADR-0007 (Telemetry & Action Flow), ADR-0008 (Grafana → n8n Automation Bridge)

---

## Context

All self-hosted services (n8n, Proxy, Ollama, Open WebUI, Grafana, Alloy, Prometheus, Loki, etc.) are deployed as **Docker containers**.  
These containers must be monitored for **new version releases** so that update workflows can be automated, auditable, and consistent across the entire stack.

Key requirements:

- Centralize detection of new image versions (no per-app hacks)
- Emit structured logs for Observability (Loki)
- Enable triggers for **Grafana → n8n** automation
- Avoid auto-updating without audit/report steps
- Maintain separation of duties (no app performs its own update checks)

Docker itself cannot detect new image versions.  
n8n should not perform this role directly (per ADR-0002 + 0007).  
Therefore, a dedicated update-detection component is required.

---

## Decision

Adopt **DIUN (Docker Image Update Notifier)** as the **single system of record** for container image update detection.

DIUN will:

- Monitor all configured Docker images (tags or digests)
- Detect when a new version is published (via Docker Hub / GHCR)
- Emit structured logs (stdout) describing the detected update
- Feed logs into **Alloy → Loki → Grafana → n8n** for automation

### Update Detection Flow

```
DIUN (new version detected)
        ↓
Alloy (log collection)
        ↓
Loki (log store)
        ↓
Grafana (alert rule: "new version available")
        ↓
Webhook → n8n
        ↓
n8n Workflow:
  - Generate diff / version report
  - Create GitHub issue
  - Perform controlled update
  - Emit audit log → Loki
```

DIUN **never applies updates**.  
It only emits telemetry that drives the automation pipeline described in ADR-0007 and ADR-0008.

---

## Scope & Placement

- DIUN runs as a **standalone container** in `/modules/update-checker` or `/modules/observability/diun`.
- DIUN **joins `obs-net`** so Alloy can collect its logs.
- DIUN watches:
  - Core infrastructure (Alloy, Prometheus, Loki, Grafana)
  - Platform components (n8n, Proxy, Ollama, OpenWebUI)
  - Support services (cloudflared, Diun itself)
  - Optional: business apps, databases, other modules

The configuration lives in a managed `diun.yml` file.

---

## Security

- No external access; DIUN does not expose ports.
- DIUN reads container metadata only (no volumes with secrets).
- Cloudflare Zero Trust controls **Grafana** exposure only.
- n8n webhook for update automation remains protected per ADR-0008.

---

## Consequences

### Positive
- ✔ Standardized, predictable update detection for **all** containers  
- ✔ Update events fully observable (Loki logs + Prometheus metrics if extended)  
- ✔ Update workflows fully automated but still auditable  
- ✔ No app-specific logic; DIUN consolidates update monitoring  
- ✔ Integrates cleanly with ADR-0006/0007/0008 pipelines  

### Negative
- ⚠ Additional container to maintain  
- ⚠ Need to manage DIUN configuration for watched images  
- ⚠ Update frequency depends on registry rate limits  

---

## Alternatives Considered

### 1. n8n GitHub/Docker API polling  
Rejected — duplicates logic, increases maintenance, spreads update detection across workflows, violates centralized telemetry design.

### 2. Application-native update checkers  
Rejected — inconsistent, unreliable, most apps do not emit update logs.

### 3. No update automation  
Rejected — incompatible with desired observability-driven update workflow.

---

## Future Extensions

- Emit DIUN metrics into Prometheus (either via Alloy transform or custom exporter)
- Automated changelog retrieval + diff analysis
- Multi-registry authentication support
- Integration with GitHub Projects for update tracking
