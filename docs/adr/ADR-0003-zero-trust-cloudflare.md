# ADR-0003: Zero-Trust Ingress via Cloudflare Tunnels

**Date:** 2025-11-06  
**Status:** Accepted
**Deciders:** Owner

---

## Context
Multiple self-hosted services are exposed to the internet.  
We need a consistent approach for authentication, isolation, and auditability using Cloudflare Zero Trust.

---

## Decision
Adopt **Cloudflare Zero Trust** as the single ingress layer, with **separate tunnels per logical service**:

| Service | Subdomain | Tunnel | Path | Cloudflare Access | Status |
|----------|------------|---------|------|--------------------|--------|
| LLM (Open WebUI) | `llm.{{TEST_SECRET_ROOT_DOMAIN}}` | `llm-ui-tunnel` | `*` | ✅ Enabled | Ready |
| n8n | `n8n.{{TEST_SECRET_ROOT_DOMAIN}}` | `n8n-ui-tunnel` | `*` | ✅ Enabled | Ready |
| PocketBase (public API) | `api.{{TEST_SECRET_ROOT_DOMAIN}}` | `pocketbase-tunnel` | `*` | 🚫 Disabled (public read-only, see ADR-0022) | Planned |
| n8n Webhooks | `hooks.{{TEST_SECRET_ROOT_DOMAIN}}` | `n8n-webhook-tunnel` | `/webhook/*` | 🚫 Disabled (public) | Ready |

Each tunnel uses its own routing and policy under Cloudflare Zero Trust Access.  
Identity enforcement (SSO, MFA, posture checks) is handled entirely by Cloudflare.  

**Tunnel placement rule:** each cloudflared instance runs **on the node hosting the service it exposes**, terminating at that service locally. Tunnels are never hosted on a different node (e.g. the DMZ) forwarding inward — doing so would create standing cross-zone paths contrary to ADR-0016.

---

## Consequences
- Clear service isolation and fine-grained policy control.  
- Simplified monitoring and Access logs per service.  
- Slightly more DNS and tunnel management overhead (acceptable).

---

## Changelog

| Date | Change Type | Description | Decider |
|------|-------------|-------------|---------|
| 2026-06-10 | Updated | Status to Accepted. Removed deprecated observability tunnel (stack superseded by ADR-0023). Added PocketBase tunnel (ADR-0022). Added explicit tunnel placement rule: cloudflared runs on the node hosting the exposed service. | Owner |
