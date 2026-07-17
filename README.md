# 🧱 My Docker Tech Stack

This repository documents and backs up my personal Docker-based infrastructure setup.  
It includes configurations, compose files, and supporting services for a modular, self-hosted automation and AI environment.

The goal is to keep things **reproducible**, **observable**, and **securely connected**, with clear documentation and minimal manual setup.

---

## 📦 Contents

- `cloudflared/` – Secure remote access tunnels via Cloudflare
- `ghost/` – Website
- `n8n/` – Automation workflows and supporting services  
- `ollama/` – Local LLM backend  
- `openwebui/` – Web UI interface for Ollama  
- `proxy/` – Middleware between n8n and Ollama/OpenWebUI  
  - Logs all prompts and responses  
  - Stores logs in a dedicated PostgreSQL instance (separate from workflow DBs)  
  - Enables structured analysis and auditing  
- `nocodb/` – Lightweight data interface for managing structured data  
- `playwright/` – Browser automation and E2E testing for workflows and integrations  
- `observability/` – observability stack
  - `grafana/` – Observability dashboards
  - `obs-core` - Loki, Prometheus, Alloy, OTel collector 

---

## 🧭 Goals

- Ensure reproducible infrastructure for automation and AI experiments  
- Enforce separation of operational, logging, and business databases  
- Integrate observability and compliance-friendly logging  
- Provide a modular foundation for iterative improvements and experimentation  

---

## ⚙️ Requirements

- Docker & Docker Compose  
- PostgreSQL (one or multiple instances)  
- Optional: Traefik / Cloudflare Tunnel for HTTPS and remote access  
- Hardware: a capable workstation or NAS (e.g., Minisforum or Intel NUC)  

---

## 🚀 Usage

Each service folder contains its own `docker-compose.yml` and `.env` (if needed).  
Bring up a service or the entire stack with:

```bash
docker compose up -d
```

> ⚠️ Review environment variables and mount paths before starting.  
> Default settings assume local, single-host deployment.

---

## 🧰 Additional Components

### 🧩 Proxy Layer
- Routes communication between **n8n**, **Ollama**, and **OpenWebUI**.  
- Intercepts and logs all prompt–response pairs into a dedicated **PostgreSQL** database.  
- Supports future extensions like:
  - Request tagging for compliance auditing  
  - Structured analytics and trace visualization  
  - Integration with Grafana dashboards  

### 🧪 Playwright
- Used for automated UI testing and workflow validation.  
- Helps ensure consistent n8n node execution and OpenWebUI integration performance.

### 🔄 Planned: Automated Upgrades
- Future service for detecting and safely applying container image updates.  
- Will include:
  - Version tracking and changelog parsing  
  - Pre-update backups  
  - Rollback on failure  

---

## 🧩 Notes

This setup is intended for **personal self-hosting**, **testing**, and **documentation** — not a production deployment.  
You’re welcome to adapt the structure and compose files to fit your own use case.

---

## ⚠️ Disclaimer

This project is maintained for personal use and educational documentation.  
It is **not** intended as a ready-made production platform.

---

## 🔐 Security

This repository contains no production secrets, no live endpoints, and no access keys.  
It documents a personal self-hosted environment and does not represent a public service.

Please do not submit security reports or disclosures.

---

## 🤝 Contributing

This repository is **read-only**.  
Pull requests and issues are not accepted.

You are welcome to fork the project for your own use.

---

## 📜 License

This repository is licensed under the **MIT License**.  
You are free to use, modify, and distribute this code, provided that the original copyright
and license notice are included in all copies or substantial portions of the software.

See the [LICENSE](LICENSE) file for full details.
