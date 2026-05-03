# ADR-0007: Observability Stack — Centralized Telemetry and Action Flow

**Date:** 2025-11-10  
**Status:** Approved  
**Deciders:** Owner  
**Supersedes:** None  
**Relates to:** ADR-0006 (Observability Core), ADR-0008 (Grafana → n8n Automations), ADR-0013 (Host-Level Log Collection)

---

## Context
Multiple self-hosted services (n8n, Open WebUI, Ollama, Diun, etc.) generate logs and metrics.  
We need a unified observability pipeline that ingests both logs and metrics, visualizes them in **Grafana**, and drives automated actions through **n8n** — all with full traceability and separation of duties.

### Telemetry Source Model

All logs are collected at the **host / Docker-engine level**, not emitted or shipped by applications directly.

- Applications are responsible only for writing meaningful stdout/stderr logs
- Grafana Alloy runs once per node and collects:
  - Docker container logs
  - Host system logs
- No per-application, sidecar, or compose-level log shipping is permitted

This model is defined in **ADR-0013 (Host-Level Log Collection)** and applies uniformly across all environments.

---

## Decision
Adopt a centralized observability flow using **Grafana Alloy** for collection, **Prometheus** for metrics, **Loki** for logs, and **Grafana** as the unified visualization and alerting layer.

### Telemetry Data Flow
```
All sources (n8n, Proxy, Ollama, Open WebUI, Diun, etc.)
        ↓
     Grafana Alloy (collects logs + metrics)
        ↓
     ├──→ Loki (log store & query layer)
     └──→ Prometheus (metrics TSDB & scrape engine)
        ↓
     Grafana (visualization, alert management)
```

### Actionable Flow
Actionable events may originate from **logs** or **metrics**.

```
Grafana alert (Loki pattern or Prometheus threshold)
        ↓
     Webhook trigger → n8n
        ↓
     n8n performs actions (corrective or maintenance workflows)
```

This ensures:
- **Traceability** — every automation stems from a logged event.  
- **Observability-first architecture** — logs precede actions.  
- **Auditability** — all system changes trace back to recorded log entries.

---

## Consequences
- All services must expose Prometheus metrics and log to stdout/stderr in structured form; log shipping is handled exclusively at the host level.  
- Grafana Alloy replaces Promtail as the unified collector for both logs and metrics.  
- Grafana serves as the single decision layer for alerting and action logic.  
- n8n executes actions only on verified alert events from Grafana (log- or metric-driven).  

---

## Notes
Follow-up design for **Grafana → n8n** integration and workflow definitions will be covered in **ADR-0008**.
Future extension may include trace correlation (Tempo) once adopted in the observability core.

---

## Changelog

| Date       | Change Type | Description | Decider |
|------------|-------------|-------------|---------|
| 2025-11-10 | Created | Initial definition of centralized observability, telemetry ingestion, and Grafana-driven action flow via n8n. | Owner |
| 2026-01-10 | Updated | Aligned telemetry model with host-level log collection using Grafana Alloy. Removed implicit support for per-application or sidecar-based log shipping. Added reference to ADR-0013. | Owner |
