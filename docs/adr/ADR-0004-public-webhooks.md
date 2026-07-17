# ADR-0004: Public Webhook Exposure Strategy

**Date:** 2025-11-06  
**Status:** Accepted  
**Deciders:** Owner

---

## Context
External integrations (Trello, Slack, etc.) must deliver events to n8n via webhooks.  
Cloudflare Access blocks unauthenticated requests by default, so a separate public endpoint is needed.

---

## Decision
Expose webhooks on a dedicated public domain:
```
https://hooks.{{TEST_SECRET_ROOT_DOMAIN}}/webhook/*
```
routed through a **separate Cloudflare Tunnel** without Access authentication.

**Tunnel placement:** the webhook tunnel (cloudflared) runs **on the same node as n8n**, terminating at n8n locally, per the placement rule in ADR-0003. It is not hosted on or routed through the DMZ node.

Environment variable:
```bash
WEBHOOK_URL=https://hooks.{{TEST_SECRET_ROOT_DOMAIN}}/
```

Security controls:
- n8n webhooks rely on long random secret IDs.
- Prefer webhook signature verification where supported.
- Apply Cloudflare WAF and rate-limit rules to `hooks.{{TEST_SECRET_ROOT_DOMAIN}}`.
- Keep n8n local auth enabled with a random password as fallback.
- Use least-privilege credentials within workflows.

---

## Risk & Mitigation
| Threat | Likelihood | Impact | Mitigation |
|--------|-------------|---------|-------------|
| Webhook URL guessed | Very low | Limited (trigger abuse) | Random IDs + rate limiting |
| Webhook URL leaked | Medium | Workflow misuse | Validate signatures, sanitize inputs |
| DoS on webhook domain | Low | Service slowdown | Cloudflare WAF/rate limiting |

---

## Consequences
- Reliable webhook delivery without MFA wall.  
- Public endpoint still protected by obscurity + WAF.  
- Slight operational overhead managing extra tunnel.

---

## Future Considerations

- Security of the public webhook endpoint may need tightening as usage grows (e.g. mandatory signature verification per integration, stricter WAF rules, per-source allowlists). Revisit when a new external integration is added.

## Changelog

| Date | Change Type | Description | Decider |
|------|-------------|-------------|---------|
| 2026-06-10 | Updated | Status to Accepted. Documented tunnel placement: cloudflared co-located with n8n per ADR-0003 placement rule. Added future security-tightening note. | Owner |
