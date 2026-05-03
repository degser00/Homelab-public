# ADR-0003: Zero-Trust Ingress via Cloudflare Tunnels

**Date:** 2025-11-06  
**Status:** Approved (in progress) 
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
| Observability stack | `obs.{{TEST_SECRET_ROOT_DOMAIN}}` | `observability-ui-tunnel` | `*` | ✅ Enabled | N/A |
| n8n Webhooks | `hooks.{{TEST_SECRET_ROOT_DOMAIN}}` | `n8n-webhook-tunnel` | `/webhook/*` | 🚫 Disabled (public) | Ready |

Each tunnel uses its own routing and policy under Cloudflare Zero Trust Access.  
Identity enforcement (SSO, MFA, posture checks) is handled entirely by Cloudflare.  

---

## Consequences
- Clear service isolation and fine-grained policy control.  
- Simplified monitoring and Access logs per service.  
- Slightly more DNS and tunnel management overhead (acceptable).  
