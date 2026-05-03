# Ollama Module

## Overview
This module provides a **standalone deployment of Ollama**, a local large-language-model runtime used for AI inference and retrieval-augmented workflows.  
It is **self-contained and replaceable**, in line with the modular architecture ADR.

---

## Dependencies
- **Docker Engine** (compose v2 or later).  
- **Local persistent storage** for models and embeddings.  
- **Internal Docker network** for connections from n8n or other internal services.  
- *(Optional future)* **GPU passthrough** (e.g., Intel Arc / NVIDIA) for accelerated inference.

---

## Folder Structure
```
/modules/ollama/
│
├── compose.yaml          # Deployment definition for Ollama
├── .env.example          # Template for environment variables
├── README.md             # This file
├── models/               # Persistent storage for model files
└── model-list.md         # Optional document listing models and intended uses
```

---

## Deployment
1. Copy `.env.example` → `.env` and set desired ports, data paths, and network name.  
2. Start the service:
   ```bash
   docker compose up -d
   ```
3. Access internally via:
   ```
   http://ollama:{{OLLAMA_AMD_PORT}}
   ```
   (Use this endpoint from other containers such as n8n.)

> The `compose.yaml` defines a single `ollama` container with bound model storage and configured internal network.

---

## Networking & Access
- Default port: `{{OLLAMA_AMD_PORT}}`  
- Service name: `ollama` (reachable inside Docker network)  
- Not exposed publicly.  
  - Future public access will occur **through Open WebUI** instead of direct exposure.

---

## Environment Variables
Defined in `.env`, typically:
- `OLLAMA_HOST=0.0.0.0`
- `OLLAMA_PORT={{OLLAMA_AMD_PORT}}`
- `OLLAMA_MODELS_PATH=/models`

> Extend with custom variables as needed when adding GPU support or authentication.

---

## Persistence
- **Models and embeddings** stored in the `models/` directory, mounted as a volume into `/root/.ollama`.  
- When migrating or upgrading, preserve this directory to avoid re-downloading models.  
- Logs can optionally be mounted to `/var/log/ollama`.

---

## Version Pinning
Currently using `ollama/ollama:latest`.  
Future policy (recommended):
- Pin a tested version tag (e.g. `ollama/ollama:<version>`) once automated version-tracking is in place.  
- Update manually only after validation in staging.

---

## Model Inventory
A supplementary `model-list.md` file should document:
- Model name (e.g. `llama3:8b`, `mistral:7b`, etc.)  
- Purpose (chat, code, embedding, etc.)  
- Approximate RAM/VRAM requirements  
- Dependencies or workflows that rely on it (e.g., “used by n8n AI summarization workflow”)

---

## Migration & Portability
To migrate this module, copy:
- `/modules/ollama/compose.yaml`
- `/modules/ollama/.env`
- `/modules/ollama/models/`
- `/modules/ollama/model-list.md`

Re-deploy with `docker compose up -d` on the new host.  
No external dependencies beyond this folder.
