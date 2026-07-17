# n8n Module

## Overview
This module provides a **self-contained deployment of n8n**, the automation and workflow orchestration engine used for system-wide integrations (e.g., Ollama AI workflows, headless browser automation).  
It is designed to be **portable and replaceable**, following the modular stack ADR.

---

## Dependencies
- **PostgreSQL** – persistent storage for workflows and credentials.  
  - Uses shared instance (dedicated DB and credentials for `n8n`).  
- **Redis** – used for queue mode and caching.  
- **Cloudflared Tunnel** – exposes the n8n web interface securely to the internet.  
- **Docker Engine** (compose v2 or later).

---

## Folder Structure
```
/modules/n8n/
│
├── compose.yaml          # Deployment definition (n8n + PostgreSQL + Redis)
├── .env.example          # Template for environment variables
├── README.md             # This file
└── data/                 # Local volume mounts (logs, backups, etc.)
```

---

## Deployment
1. Copy `.env.example` → `.env` and adjust parameters.  
2. Deploy using:
   ```bash
   docker compose up -d
   ```
3. Access n8n through:
   - **Local:** `http://localhost:5678`
   - **Public (Cloudflare):** via configured tunnel (e.g., `https://n8n.{{TEST_SECRET_ROOT_DOMAIN}}`)

> The `compose.yaml` defines containers for `n8n`, `postgres`, and `redis` within this module.  
> Refer to that file for detailed container configuration.

---

## Networking & Access
- Internal Docker network name: `n8n_net`
- External exposure through **Cloudflared service container** (separate or shared with other modules).
- Default port: `5678`
- Authentication handled via n8n’s built-in credentials or SSO (if enabled globally).

---

## Environment Variables
Defined in `.env`, including:
- `N8N_BASIC_AUTH_USER` / `N8N_BASIC_AUTH_PASSWORD`
- `DB_TYPE=postgresdb`
- `DB_POSTGRESDB_DATABASE`, `DB_POSTGRESDB_HOST`, `DB_POSTGRESDB_USER`, `DB_POSTGRESDB_PASSWORD`
- `REDIS_HOST`, `REDIS_PORT`
- `WEBHOOK_URL` (set to public Cloudflare tunnel address)

> Always generate unique secrets per instance.  
> Avoid storing credentials directly in workflows; use n8n’s credentials store.

---

## Persistence
- **PostgreSQL volume** for workflow and credential data.  
- **Redis** uses ephemeral storage (no persistence required).  
- Optional bind mount for `/home/node/.n8n/` to retain user settings and logs.

---

## Version Pinning
- **n8n image:** `n8nio/n8n:<pinned-version>`  
- **PostgreSQL:** `postgres:<pinned-version>`  
- **Redis:** `redis:<pinned-version>`  

Version tags are fixed in `compose.yaml` and updated manually after validation in staging.

---

## Migration & Portability
- The module can be migrated independently by moving:
  - `/modules/n8n/compose.yaml`
  - `/modules/n8n/.env`
  - `/modules/n8n/data/`
- Connects to shared services via defined network bridge or central `.env` references.
- No dependency on external configuration files beyond this folder.
