# Observability Core Stack

Centralized telemetry collection using **Alloy**, **Prometheus**, **Loki**, and integrated **side-car exporters**.  
All components run inside a **single docker-compose project**.

## Directory Structure

```
observability/obs-core/
├── docker-compose.yml
├── .env
├── alloy-config.alloy
├── prometheus/
│   └── prometheus.yml
└── loki/
    └── loki-config.yml

```

## Components

- **Alloy** – unified collector for logs & metrics  
- **Prometheus** – metrics storage and scraping  
- **Loki** – log aggregation  
- **Side-car exporters**:  
  - Postgres exporter (9187)  
  - Playwright exporter (9113)  
  - Ollama exporter (9114)  
  - OpenWebUI exporter (9115)

## Metrics Collection

### Auto-discovery via service labels  
Any container can expose metrics automatically:

```yaml
labels:
  prometheus.io/scrape: "true"
  prometheus.io/port: "{{OLLAMA_B50_PORT}}"
  prometheus.io/path: "/metrics"
```

### Docker Metrics (optional)

```
/etc/docker/daemon.json:
{
  "metrics-addr": "0.0.0.0:9323",
  "experimental": true
}
```

Prometheus scrapes:
- Alloy internal metrics  
- Prometheus internal metrics  
- Loki ingestion metrics  
- Docker daemon metrics (if enabled)  
- All side-car exporters  
- Any service with discovery labels  

## Log Collection

Alloy collects stdout/stderr for every container on the Docker host.  
Logs are sent to Loki with labels:

- `container`
- `compose_service`
- `compose_project`
- `stream`

## Ports & Exposure

Documented in `/docs/networking/ports.md`, including:
- All internal ports
- Host-exposed exporter ports
- Which components scrape which targets
- DMZ/Core/Host exposure level

## Deployment

1. Copy `.env.example` → `.env`
2. Adjust hostname variables if needed  
3. Start the stack:

```bash
docker compose up -d
```

4. Verify services:

```bash
docker compose ps
```

## Health Checks

### Prometheus  
```
curl http://localhost:9090/api/v1/targets
```

### Loki  
```
curl http://localhost:3120/ready
```

### Alloy  
```
docker compose logs alloy | grep error
```

## Troubleshooting

**Port conflicts:**  
```
sudo lsof -i :PORT
```

**Prometheus target DOWN:**  
- Check wrong port  
- Wrong labels  
- Exporter container not running  

**Missing logs in Loki:**  
- Alloy cannot access Docker socket  
- Loki ingestion errors  
- Wrong Loki URL in alloy-config.alloy  

## Retention

- **Prometheus**: 180 days  
- **Loki**: 90 days  

## Service Endpoints (Internal Network)

- **Prometheus:** `http://prometheus:9090`
- **Loki:** `http://loki:3100`
- **Alloy UI:** `http://alloy:12345`

Localhost exposure is for development only (Grafana replaces this later).

## Next Steps

- Deploy Grafana  
- Bind Grafana to Prometheus/Loki  
- Add Cloudflare Tunnel for external Grafana access  
- Add AI Proxy metrics  
- Configure alert → n8n automation workflows  
