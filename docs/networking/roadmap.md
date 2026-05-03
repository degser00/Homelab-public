# Networking Roadmap

This document outlines the planned evolution of the network and access architecture for the self-hosted platform. The goal is to gradually improve privacy, security, and independence from external services without breaking existing functionality.

---

## 0. Current State — Cloudflared Everywhere
All modules (n8n, Ollama, OpenWebUI, Observability, Reporting, Website) are exposed through **Cloudflare Tunnels**.

**Pros**
- Simple to maintain  
- Strong Cloudflare protections (WAF, rate limits, bot filtering)  
- Zero public IP needed  

**Cons**
- Cloudflare terminates TLS → Cloudflare can see all plaintext  
- Private services (UI, admin panels, APIs) unnecessarily routed through Cloudflare  
- No isolation between public and private ingress  

---

## 1. First Upgrade — Private Services via Tailscale
Move all **private-only** modules to **Tailscale** (WireGuard E2E).  
Leave **public endpoints** on Cloudflared.

### Private via Tailscale:
- n8n UI  
- Ollama API  
- OpenWebUI  
- Observability stack (Grafana, Loki, Prometheus)  
- Reporting stack  
- Databases  
- Internal dashboards  

### Public via Cloudflared:
- Website  
- n8n public webhooks  
- Any future public endpoints  

**Benefits**
- Eliminates Cloudflare visibility into private traffic  
- Reduces attack surface  
- Keeps Cloudflare benefits for public endpoints  
- Minimal operational change  

---

## 2. Final Upgrade — Headscale + Authentik
Migrate from SaaS-controlled identity/network to fully self-hosted control-plane.

### 2.1 Tailscale → Headscale
Run **Headscale** on a low-power always-on device (e.g., Raspberry Pi).

**Benefits**
- Fully self-hosted, no SaaS limits  
- Unlimited users/devices  
- Same UX and features as Tailscale Mesh  
- Improves long-term reliability and sovereignty  

### 2.2 Google OIDC → Authentik
Deploy **Authentik** as the identity provider for:
- Headscale  
- n8n  
- Observability  
- Custom services  
- Any OAuth/OIDC integrations  

**Benefits**
- Unified SSO for the entire tech stack  
- Full control over user lifecycle  
- Strong privacy and no external dependency  

---

## Summary Roadmap

| Stage | Description | Outcome |
|-------|-------------|---------|
| **0. Current** | Everything exposed via Cloudflared | Simple but Cloudflare sees all traffic |
| **1. Private → Tailscale** | Only public endpoints remain on Cloudflared | E2E privacy for internal services |
| **2. Headscale + Authentik** | Full self-hosted identity & control-plane | No SaaS dependence, complete sovereignty |

---

## Notes
- Cloudflared remains the best choice for public ingress long-term.  
- Headscale + Authentik provide strong privacy for internal services.  
- Raspberry Pi 3B+ is sufficient for Headscale, but Authentik is better on a Pi 4 or mini-PC.  
- This roadmap aligns with blue/green rollouts and modular architecture.

