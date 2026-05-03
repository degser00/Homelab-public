# ADR-0006: Observability Stack (Modular Deployment – Alloy + Prometheus + Loki)

**Date:** 2025-11-13  
**Status:** Accepted  
**Deciders:** Owner  
**Relates:** ADR-0002 (Separation of Duties & Compliance), ADR-0007 / 0008 (Grafana → n8n Bridge), ADR-0013 (Host-Level Log Collection)

---

## Context
We require a **self-hosted observability stack** to monitor health, performance, and compliance metrics across **n8n**, the **AI Proxy**, **Ollama/OpenWebUI**, and related services — without collecting raw AI content.  

The previous design used multiple agents (**Promtail**, **cAdvisor**, **Node Exporter**) to feed **Loki** and **Prometheus**, and in some cases per-application log sidecars.  
We now replace those with **Grafana Alloy**, a single lightweight telemetry agent capable of collecting **logs, metrics, and traces**.

Observability must consume only **metrics/logs** from the Proxy (see ADR-0002); content remains isolated.

---

## Decision
Adopt a **modular observability stack** built on:

- **Grafana Alloy** — unified agent for log & metric collection  
- **Prometheus** — time-series database & scrape engine for metrics  
- **Loki** — central log store  
- **Grafana (OSS)** — visualization, alerting, correlation layer  
- *(Optional)* **Tempo** — trace backend (future extension)

---

### Log Collection Model

Log collection is performed at the **host / Docker-engine level**, not at the application level.

- Grafana Alloy runs once per node
- All container logs are collected automatically
- No per-compose or per-container log configuration is permitted

This model is defined in **ADR-0013 (Host-Level Log Collection)** and applies to all environments, including TrueNAS SCALE Apps.

---

### Deployment Split (Compose Projects)

> Observability remains modular. The **Proxy** is separate but monitored through this stack.

| Module | Path | Services |
|---------|------|-----------|
| **Core** | `/modules/observability/core` | `alloy`, `prometheus`, `loki` |
| **Grafana** | `/modules/observability/grafana` | `grafana` |
| **Ingress (Cloudflare)** | `/modules/ingress/cloudflared-grafana` | `cloudflared` tunnel for Grafana UI |

**Related external service**
- **Proxy** (`/modules/proxy`) — sits between `n8n → Proxy → Ollama/OpenWebUI`; exports `/metrics` & structured logs for Alloy collection.

---

### Networks
- `obs-net` — shared between core, Grafana, cloudflared; **no host ports** exposed  
- `internal-ai` — n8n / Proxy / Ollama communication  
- `postgres-net` — Proxy ↔ Compliance DB / NocoDB  

Alloy instances join relevant networks to collect logs & metrics locally and forward to Loki / Prometheus over `obs-net`.

---

### Scope & Boundaries
- Ingests:
  - Container & system metrics via Alloy → Prometheus  
  - Application logs via Alloy → Loki  
  - Proxy telemetry (token usage, latency, error rate)
- No raw AI content ingested  
- Observability DB access limited to read-only role `obs_ro`

---

### Security
- Only **Grafana** exposed externally through **Cloudflare Zero Trust**  
- All credentials managed through environment secrets / Vault  
- Grafana roles: Admin (Owner), Editor (Operators), Viewer (Read-only)

---

### Data Retention
| Component | Default | Notes |
|------------|----------|-------|
| **Prometheus** | 180 days | configurable per storage policy |
| **Loki** | 90 days | log retention only |
| **Tempo** *(future)* | 30 days | optional traces |
| **Compliance DB** | per ADR-0002 | independent maintenance |

---

### Alerting Baseline
- Host / container down or scrape failure  
- High CPU, RAM, or disk pressure  
- n8n workflow error rate spike  
- Proxy error rate / latency SLO breach  
- Token throughput or in/out imbalance  

Alerts are raised in Grafana and forwarded to **n8n** via webhook.

---

### Alternatives Considered
| Option | Reason Rejected |
|---------|----------------|
| SaaS Observability | Cost & data sovereignty |
| Multi-agent model (Promtail + cAdvisor + Node Exporter) | Higher complexity and maintenance |
| Monolithic Compose Stack | Larger blast radius / upgrade risk |

---

### Consequences
✅ Unified telemetry pipeline (logs + metrics + traces)  
✅ Simpler agent management via Alloy  
✅ Zero-Trust exposure limited to Grafana  
⚠️ Prometheus and Loki storage tuning required over time  
⚠️ Multiple compose projects to manage (runbook mitigation)

---

## Rollout Plan
1. Create `obs-net`; deploy **Alloy + Loki + Prometheus** core.  
2. Deploy **Grafana**; configure Prometheus and Loki as data sources.  
3. Deploy **cloudflared** tunnel for Grafana; enforce Zero Trust access.  
4. Update **Proxy** to expose `/metrics`; Alloy scrapes logs & metrics.  
5. Connect n8n and Proxy telemetry sources; validate dashboards & alerts.  
6. Document credentials, retention policy, and runbooks.

---

## Documentation Layout
- `/docs/adr/ADR-0002.md` → add “Observability Interface” and “Proxy Telemetry” sections  
- `/docs/adr/ADR-0006.md` → this ADR  
- `/docs/runbooks/observability/`  
  - `bootstrapping.md` – deployment / backups / upgrade  
  - `alerts.md` – alert definitions & remediation  
  - `dashboards.md` – IDs / ownership  
- `/docs/security/zero-trust.md` – Cloudflare Grafana policy  
- `/modules/observability/...` – compose projects and configs  
- `/modules/proxy/...` – standalone Proxy compose + metrics config

## Changelog

| Date       | Change Type | Description | Decider |
|------------|-------------|-------------|---------|
| 2025-11-13 | Created | Initial definition of the modular observability stack using Grafana Alloy, Prometheus, Loki, and Grafana. Replaced multi-agent model with a unified telemetry agent. | Owner |
| 2026-01-10 | Updated | Clarified log collection model as **host / Docker-engine level only**. Explicitly removed sidecar-based logging patterns. Added reference to ADR-0013 (Host-Level Log Collection). | Owner |
