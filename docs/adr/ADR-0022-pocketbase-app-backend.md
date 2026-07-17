# ADR-0022: PocketBase as Self-Hosted REST API Backend for Ad-Hoc Applications

**Date:** 2026-06-10
**Status:** Accepted
**Deciders:** Owner
**Relates to:** ADR-0003 (Zero-Trust Cloudflare Ingress), ADR-0012 (Compute-Centric Data Ownership), ADR-0016 (Network Zones and Trust Boundaries)

---

## Context

A recurring need exists for lightweight, short-lived applications — humidity trackers, inventory tools, single-purpose dashboards, and similar utilities — that require persistent storage and a REST API backend.

These applications share common characteristics:

- Built rapidly with AI assistance (Claude), not through traditional development workflows
- Scoped to a specific task or time period
- Require persistent read/write storage accessible from a browser
- Shared with a small number of external users (read-only) while the owner retains full write access
- Do not warrant dedicated infrastructure, documentation overhead, or multi-tenancy design

Previously, Firebase was used as a transient backend for these apps. This creates dependency on an external service with a 30-day free tier and no data sovereignty.

A self-hosted alternative is needed that:

- Requires no development or ongoing maintenance
- Produces a REST API automatically from a schema definition
- Supports differential access (public read-only vs. authenticated full access)
- Integrates cleanly with the existing Cloudflare Tunnel + Tailscale stack
- Is lightweight enough to run alongside other workloads on CORE-NAS

---

## Decision

Adopt **PocketBase** as the standard self-hosted backend for ad-hoc applications.

PocketBase runs as a single Docker container on **CORE-NAS**. It exposes a REST API automatically for any collection defined via its admin UI. No code is required to create, modify, or query collections.

Public exposure follows ADR-0003: a **dedicated Cloudflare Tunnel** runs as a cloudflared container **on CORE-NAS**, terminating directly at PocketBase (`localhost`-scoped Docker network). The tunnel does **not** transit the DMZ node, and no shared reverse proxy sits in the public path.

---

## Deployment

- **Host:** CORE-NAS (Docker/Compose stack, own module folder per ADR-0001)
- **Data:** Persisted to local ZFS storage via Docker volume, consistent with ADR-0012 compute-centric ownership model
- **Admin UI:** Accessible via Tailscale only — never routed through the public tunnel
- **Public API:** Dedicated cloudflared container on CORE-NAS → PocketBase, read-only (see Access Model)

---

## Access Model

Access is split across two paths aligned with the existing Tailscale split-DNS topology.

**Standing assumption:** the Owner is always on Tailscale. If not on Tailscale, the Owner is treated as any other public visitor (read-only). No authenticated public write path exists or is planned.

### Public path (Cloudflare Tunnel on CORE-NAS → PocketBase)

- Read-only
- Enforced at two layers:
  1. **PocketBase collection API rules** (primary): `list`/`view` open; `create`/`update`/`delete` restricted to admin — applied as the default template for every new collection
  2. **Cloudflare WAF rule** (secondary, defense-in-depth): block non-`GET`/`HEAD`/`OPTIONS` methods on the PocketBase hostname
- No authentication required for reads
- No PocketBase credentials exposed externally; admin endpoints (`/_/`) blocked at the Cloudflare layer

### Internal path (Tailscale)

- Full read/write access
- Reaches PocketBase directly via internal DNS
- Admin UI accessible on this path only

---

## Scope

### PocketBase is used for

- Ad-hoc, short-lived single-purpose applications
- Applications generated with AI assistance requiring a persistent backend
- Any app where schema design, querying, and API access can be handled without writing backend code

### PocketBase is NOT used for

- Core infrastructure services
- Applications with formal SLAs or multi-user authentication requirements
- Workloads requiring relational joins, complex queries, or high write throughput
- Replacing NocoDB, n8n, or other established service-specific backends

---

## Data Lifecycle

Applications and their collections are ephemeral by design. When an app is no longer needed:

- The collection is deleted via the admin UI
- No formal decommission process is required

PocketBase data is stored on CORE-NAS ZFS storage and is included in normal volume-level backup processes per ADR-0012. Individual app data is not considered critical and does not require dedicated backup policy.

---

## Consequences

✅ Zero backend development required — schema defined via UI, REST API generated automatically
✅ Single container, minimal resource footprint on CORE-NAS
✅ Public path never touches the DMZ — consistent with ADR-0016 (DMZ → Management disallowed)
✅ Per-service tunnel isolation consistent with ADR-0003 — tunnel compromise affects PocketBase only
✅ Data sovereignty — no external service dependency
✅ AI-assisted app generation works immediately — PocketBase's API is well-understood by Claude
✅ Consistent with compute-centric data ownership (ADR-0012)

⚠️ Not suitable for applications requiring complex relational data models
⚠️ Read-only enforcement depends on collection rule discipline — a misconfigured collection rule could expose write access; the Cloudflare WAF method-block is the backstop
⚠️ Owner cannot write when off Tailscale (accepted by design)
⚠️ No per-app access logging or audit trail

---

## Alternatives Considered

| Option | Reason Rejected |
|--------|----------------|
| **Firebase** | External dependency, 30-day free tier, no data sovereignty |
| **Public path via DMZ + Caddy (CF Tunnel → DMZ → Caddy → CORE-NAS)** | Originally proposed in the first draft of this ADR. Rejected: creates a standing DMZ → Management traffic path, violating ADR-0016 default-deny; concentrates routing in a shared proxy contrary to ADR-0003 per-tunnel isolation; DMZ compromise would sit in the data path |
| **FastAPI + SQLite** | Requires development work per app; contradicts the no-code, AI-generated app model |
| **Supabase (self-hosted)** | Significantly heavier stack (Postgres, Kong, GoTrue, PostgREST, Studio); operational overhead disproportionate to use case |
| **NocoDB** | Spreadsheet-UI-oriented; better suited for human data browsing than as an API backend for generated apps. Already in use for its intended purpose |
| **Directus** | Complex configuration and heavier runtime; requires more setup than the use case warrants |
| **JSON Server** | No auth, no admin UI, not suitable for shared external access |

---

## Changelog

| Date | Change Type | Description | Decider |
|------|-------------|-------------|---------|
| 2026-06-10 | Created | Initial decision to adopt PocketBase as the standard backend for ad-hoc AI-generated applications, with public read-only access via Cloudflare Tunnel and full access via Tailscale. | Owner |
| 2026-06-10 | Updated | Corrected ADR number (was mislabelled 0015). Public path redefined: dedicated cloudflared on CORE-NAS terminating directly at PocketBase; DMZ and Caddy removed from public path per ADR-0016/ADR-0003. Read-only enforcement moved to PocketBase collection rules with Cloudflare WAF method-block as secondary layer. Documented standing assumption that Owner writes occur via Tailscale only. | Owner |
