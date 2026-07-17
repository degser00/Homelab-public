# Port & Exposure Matrix – Tech-Stack + Observability

| Service | Purpose | Container Port | Host Port | Exposure | Notes |
|---------|---------|----------------|-----------|----------|-------|
| **n8n UI** | Workflow automation | 5678 | 5678 | ✅ Cloudflare tunnel `cloudflared-n8n` | Primary UI |
| **n8n Webhooks** | Trigger endpoints | 5678 | 5678 | ✅ Cloudflare tunnel `cloudflared-hooks` | Path `/webhook/*` |
| **PostgreSQL (n8n)** | App database | 5432 | – | 🔒 `n8n-net` only | Not exposed |
| **Redis (n8n)** | Cache / queue | 6379 | – | 🔒 `n8n-net` only | Not exposed |
| **Ollama** | Local LLM API | {{OLLAMA_AMD_PORT}} | – | 🔒 `ai-net` only | Accessed via Open-WebUI |
| **Open-WebUI** | LLM front-end | {{OLLAMA_B50_PORT}} | 3000 | ✅ Cloudflare tunnel `cloudflared-llm` | Zero-Trust protected |
| **Playwright** | Test runner | 3000 | 3100 | 🔒 Local only | Host-bound, no external ingress |
| **Cloudflared (n8n)** | Tunnel client | – | – | n/a | Handles `n8n` & `hooks` tunnels |
| **Cloudflared (LLM)** | Tunnel client | – | – | n/a | Handles `open-webui` tunnel |
| **Cloudflared (Hooks)** | Tunnel client | – | – | n/a | Dedicated webhook ingress |
| **Observability Core** | | | | | |
| **Prometheus** | Metrics TSDB | 9090 | – | 🔒 `obs-net` only | Accessed via Grafana |
| **Loki** | Log store | 3100 | – | 🔒 `obs-net` only | Accessed via Grafana |
| **Alloy** | Log collector / routing | 12345 | – | 🔒 `obs-net` only | No direct external access |
| **OTel Collector** | OTLP ingest / export | 4317/4318, 8889 | – | 🔒 `obs-net` only | Scraped by Prometheus |
| **Exporter Ports (host network)** | | | | | |
| **postgres-exporter** | Postgres metrics | 9187 | 9187 | 🔒 Host only | Side-car to postgres |
| **playwright-exporter** | Playwright metrics | 9115 | 9115 | 🔒 Host only | Side-car to playwright |
| **ollama-exporter** | Ollama metrics | 9113 | 9113 | 🔒 Host only | Side-car to ollama |
| **open-webui-exporter** | Open-WebUI metrics | 9114 | 9114 | 🔒 Host only | Side-car to open-webui |

### Exposure Legend

| Symbol | Label | Meaning |
|:-------|:------|:--------|
| ✅ **Cloudflare Tunnel** | Secure public ingress via Cloudflare Zero Trust | Auth-protected, no direct host exposure |
| 🔒 **Internal Network** | Docker bridge network only | Inter-service traffic (e.g. n8n → Postgres) |
| 🔒 **Local only** | Bound to `127.0.0.1` on host | Reachable from host, not external |
| 🔒 **Host only** | Bound to `0.0.0.0:PORT` on host but not CF-tunnelled | Used by side-car exporters, scraped by Prometheus inside Docker |

### Observability Metrics Sources

| Container | Metrics Today | Scrape Job | Port | Type |
|-----------|---------------|------------|------|------|
| **n8n** | native `/metrics` | `n8n` | 5678 | native |
| **postgres-exporter** | side-car `/metrics` | `postgres` | 9187 | side-car |
| **cloudflared tunnels** | native `/metrics` | `cloudflared` | 4443-4445 | native |
| **playwright** | side-car `/metrics` | `playwright` | 9115 | side-car |
| **ollama** | side-car `/metrics` | `ollama` | 9113 | side-car |
| **open-webui** | side-car `/metrics` | `open-webui` | 9114 | side-car |

All jobs defined in `obs-core/prometheus/prometheus.yml`.

### Upgrade Safety

- Side-car exporters use official images: `docker compose pull && docker compose up -d` upgrades everything without losing metrics.
- Native endpoints (n8n, Cloudflared) survive image updates automatically.
- No code changes required inside application containers.

### Troubleshooting

| Symptom | Check |
|---------|-------|
| Target **DOWN** in Prometheus | `docker exec prometheus wget -qO- localhost:9090/api/v1/targets` |
| No logs in Loki | `docker exec alloy wget -qO- localhost:12345/-/ready` |
| Missing metrics | `docker logs otel-collector --tail 20` |
| Port conflict / bind error | `ss -ltn | grep :PORT` |

### Security Notes

- All external traffic enters via **Cloudflare Zero Trust tunnels**; no direct host exposure.
- Exporter ports are **host-bound** but **not tunnelled** — scraped only by Prometheus inside Docker.
