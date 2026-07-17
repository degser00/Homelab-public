# ADR-0023: Slim Log-Based Alerting Pipeline

**Date:** 2026-06-10
**Status:** Accepted
**Deciders:** Owner
**Relates to:** ADR-0013 (Host-Level Log Collection), ADR-0009 (Container Image Update Detection), ADR-0008 (Grafana → n8n Automation Bridge, superseded)
**Supersedes:** ADR-0006 (Observability Core), ADR-0007 (Telemetry & Action Flow), ADR-0008 (Grafana → n8n Automation Bridge)

---

## Context

The original observability stack (ADR-0006/0007/0008) defined a full dashboard-and-telemetry platform: Prometheus metrics, curated Grafana dashboards, per-service telemetry design, and compliance metric views. In practice:

- Dashboards are not looked at
- Per-service observability onboarding never happened and is not wanted
- The operational need is narrower: **be told automatically when something goes wrong**

ADR-0013 already solved the onboarding problem at the collection layer — one Grafana Alloy agent per host scrapes all Docker container stdout/stderr and system logs automatically. A new container is covered the moment it starts, with zero configuration.

What remains is to define a minimal consumer of those logs whose only job is alerting.

---

## Decision

Retain a **slim alerting pipeline** and deprecate the dashboard/metrics platform:

```
Alloy (one per host, per ADR-0013)
   ↓
Loki (single log store)
   ↓
Grafana — alert rules ONLY (no curated dashboards)
   ↓
Webhook → n8n
   ↓
Notification (Telegram / email) + optional automation
```

### Kept
- Alloy per host (unchanged, ADR-0013)
- Loki as the single log store
- Grafana strictly as an alert-rule engine
- n8n as the notification/automation endpoint
- DIUN update-detection events (ADR-0009) ride this same pipe unchanged

### Dropped
- Prometheus and all metrics collection
- Curated dashboards and dashboard maintenance
- Per-service telemetry/onboarding design (ADR-0007)
- Compliance metrics views for observability (ADR-0002 observability interface section — compliance DB itself is unaffected)

### Alert rule scope (initial)
- Container crash loops / repeated restarts (Docker engine logs)
- ERROR-level burst patterns per container
- Host-level: disk, ZFS, service lifecycle failures from journald
- DIUN "new image version" events (per ADR-0009)

New services require **no onboarding** — they are covered by host-level collection and generic rules by default. Service-specific rules are added only when a real incident motivates one.

---

## Consequences

✅ "If shit goes south, something noticed" — without operating a dashboard platform
✅ Zero per-service onboarding (inherited from ADR-0013)
✅ ADR-0009 update-detection pipeline preserved without modification
✅ Smaller footprint: Prometheus removed; Grafana reduced to alert engine

⚠️ No metrics: capacity/performance questions can't be answered from this stack
⚠️ Alert rule quality determines value; generic rules may need tuning to avoid noise
⚠️ Grafana+Loki remain two services to keep alive for the pipeline to function

---

## Alternatives Considered

| Option | Reason Rejected |
|--------|----------------|
| Full deprecation (no alerting) | Loses failure detection entirely; unacceptable for unattended services |
| Liveness-only via Portainer API polling from n8n | Considered as an even slimmer option; rejected because Portainer aggregates no logs — only container state — so ERROR-pattern and host-level (ZFS/disk) alerting would be lost |
| Keep full ADR-0006/0007 stack | Dashboards unused; per-service telemetry model abandoned; maintenance cost without consumption |

---

## Changelog

| Date | Change Type | Description | Decider |
|------|-------------|-------------|---------|
| 2026-06-10 | Created | Replaced the full observability platform (ADR-0006/0007/0008) with a slim alert-only pipeline: Alloy → Loki → Grafana alert rules → n8n → notify. Prometheus and dashboards dropped. | Owner |
