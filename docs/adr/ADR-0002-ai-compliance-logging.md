# ADR-0002: Split of Concerns for AI Compliance Logging and Observability
Date: 2025-11-05
Status: Accepted
Deciders: Owner

## Context:
- We need compliant handling of AI prompts/responses with **separation of duties**.
- n8n triggers AI calls but should not access full prompt/response logs.
- Observability should measure **operational metrics** without exposing content.
- Single PostgreSQL instance will host **four separate databases** with distinct credentials:
  1) n8n workflow/queue DB  
  2) Business workflows DB  
  3) AI compliance logs DB  
  4) Observability summaries/aux (if needed)

## Decision:
- Introduce a **Proxy** service between n8n and Ollama/OpenWebUI:
  - The Proxy is the **sole writer** of **AI compliance logs** (prompts, responses, metadata) into the **Compliance DB**.
  - n8n has **no read access** to compliance content; it only receives the model’s response.
  - Observability stack (Grafana/Prometheus/Loki) has **read-only access** to **aggregated/metrics views** (no raw content).
  - NocoDB provides **human/audit read-only** access to the Compliance DB for authorized users.
- Enforce least-privilege DB roles:
  - **proxy_rw** → R/W on compliance schema only.
  - **obs_ro** → R/O on metrics views only.
  - **nocodb_ro** → R/O on full compliance schema (auditors/admins).
  - **maint_runner** → delete/archive privileges for retention jobs.
- Retention handled **outside n8n** by a scheduled maintenance job/container:
  - Periodic deletion of rows older than the policy window.
  - Optional export to encrypted cold storage before deletion.

## Consequences:
- ✅ Clear separation: runtime automation (n8n) vs. logging (Proxy) vs. visibility (Observability) vs. audit (NocoDB).
- ✅ Reduced risk of accidental data exposure via dashboards.
- ✅ Auditable access path for sensitive content.
- ⚠️ Additional components (Proxy + Maintenance job) increase operational surface.
- ⚠️ More credentials to manage; must document rotation.

## Scope (Current):
- Confidential-but-queryable logs (admins/auditors via NocoDB).
- Single PostgreSQL instance, four DBs with distinct users/permissions.
- Observability limited to metrics/summaries (no content).

## Workflow (High-Level):
1) **n8n → Proxy → Ollama/OpenWebUI** for model calls.  
2) **Proxy → Compliance DB** writes prompt/response/metadata.  
3) **Observability** reads **metrics views only** for dashboards.  
4) **NocoDB** provides **read-only** human access to compliance contents.  
5) **Maintenance job** enforces retention/archival.

## Observability Interface (non-content)

- The Observability stack now uses **Grafana Alloy** as the unified telemetry agent.
- Alloy collects:
  - **Metrics** → forwarded to **Prometheus**
  - **Logs** → forwarded to **Loki**
  - *(optional)* **Traces** → future extension via Tempo
- Alloy replaces Promtail, cAdvisor, and Node Exporter.
- Observability has **no direct read** of raw AI prompts/responses in compliance tables.
- Database access for Observability remains restricted to **`obs_ro`** on **metrics/summary views** only.
- Operational KPIs (token usage, latency, error rates) are exported by the Proxy as Prometheus metrics.


## Ingress & Exposure

- Only **Grafana** is exposed externally via **Cloudflare Zero Trust**.
- **Alloy**, **Prometheus**, and **Loki** run internally on the `obs-net` network; no host ports are exposed.
- n8n and the Proxy each expose **Prometheus-compatible endpoints** (`/metrics`) and write structured logs; Alloy scrapes and forwards these internally.

## Proxy Placement (for clarity)

- The **Proxy** is a standalone service that sits **between n8n and Ollama/OpenWebUI**.
- Data flow: `n8n → Proxy → Ollama/OpenWebUI`; n8n does **not** write compliance content.
- The Proxy writes prompts/responses/metadata to the **Compliance DB** using `proxy_rw`.
- Observability reads **metrics/logs** from the Proxy only (no content).

The Observability layer monitors Proxy and n8n performance through Alloy collectors.
This ensures full operational visibility (CPU, RAM, latency, error rates) while maintaining
strict separation between **content (Compliance DB)** and **telemetry (Prometheus/Loki)**.


## Alternatives Considered:
- Logging directly from n8n: violates separation, expands n8n permissions.
- Observability with full-content access: increases exposure risk and scope.

## Future Extensions (separate ADRs later):
- **High-sensitivity tier**: encryption-at-rest per column, view-level redaction, key management.
- **Automated redaction** (PII/secret scrubbing) at Proxy before persistence.
- **Cold storage** archival (e.g., S3/Blob, Parquet/CSV) with lifecycle policies.
- **Access reviews** and alerting on anomalous queries to compliance tables.
