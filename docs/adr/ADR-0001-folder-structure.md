# ADR-0001: Modular Folder Structure with Self-Contained Compose Files
Date: 2025-11-05
Status: Accepted
Deciders: Owner
Context:
- The stack is operated by a non-developer and must be easy to back up, migrate, and restore.
- Services are grouped into logical modules (e.g., `n8n`, `ollama`, `openwebui`, `proxy`, `grafana-stack`, `nocodb`, `cloudflared`, `playwright`, `common`).
- Each module may evolve independently over time.

Decision:
- Use a **fully modular structure** where **each module lives in its own folder** with:
  - Its **own `docker-compose.yml`** (self-contained).
  - Its **own `.env`** and volume/bind mounts.
  - Clear **README** per module describing purpose, prerequisites, and start/stop steps.
- No single root compose file; orchestration is done by running individual modules as needed.
- Shared items (e.g., networks, labels, base env templates) live in `common/` but **modules do not depend on a root compose**; they include required shared definitions locally or via documented copy.
- Backups and restores are performed per-module to support partial migrations.

Consequences:
- ✅ Easier migration/restore: move modules independently.
- ✅ Reduced blast radius: issues in one module rarely impact others.
- ✅ Clearer ownership and lifecycle: disable/enable modules without touching the rest.
- ⚠️ Slight duplication (e.g., repeating network names) across modules.
- ⚠️ Cross-module wiring (e.g., pointing `n8n` to `proxy`) must be documented in each module’s README.

Scope:
- Applies to all current modules and any future additions.
- Does not prescribe a reverse proxy choice or TLS strategy (left to module READMEs).

Alternatives Considered:
- **Monolithic root compose**: simpler start/stop, but harder to migrate/restore and reason about.
- **Root compose with `-f` includes**: reduces duplication, but reintroduces coupling and central coordination.

Operational Notes (overview only):
- Back up volumes and per-module `.env` files.
- Use consistent naming conventions for networks and volumes across modules to avoid conflicts.

Future Work:
- Optional tooling to enumerate and start/stop modules in a defined order (without central compose coupling).
