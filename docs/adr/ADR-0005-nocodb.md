# ADR-0005: NoCoDB Exposure Policy

**Date:** 2025-11-06  
**Status:** Proposed  
**Deciders:** Owner

---

## Context
NoCoDB is used for internal data access and administration.  
It should not be internet-exposed.

---

## Decision
Keep NoCoDB fully local:
- No Cloudflare tunnel or public route.  
- Accessible only over LAN or VPN (e.g., WireGuard, Tailscale).  
- If external access is ever required, it must go behind Cloudflare Access with MFA and device posture enforcement.

---

## Consequences
- Minimal attack surface.  
- Requires VPN/bastion for remote access (acceptable trade-off).  
