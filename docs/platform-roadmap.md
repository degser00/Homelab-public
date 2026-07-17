# Platform Roadmap

This document outlines the long-term evolution of the entire self-hosted platform, including third-party dependencies, identity, hosting strategy, modules, and access model.

**Last updated:** 2026-04-19

## Architectural Alignment

This roadmap is aligned with the following architectural decisions:
- ADR-0015 (Platform Topology and Device Classes)
- ADR-0016 (Network Zones and Trust Boundaries)

All roadmap stages assume:
- Clear separation between DMZ, Internal, and Management zones
- Stateless services run in the Internal zone
- Authoritative user data resides only in the Management zone
- Public exposure is limited to explicitly designated DMZ services

## 1. Third-Party Services (kept)

- **VPN** — general web browsing privacy
- **Password Manager** — secure credential storage
- **Email DNS settings** point to an external email provider  
  (currently ImprovMX for forwarding, later possibly MXRoute for full inbox hosting)

## 2. Self-Hosted Services
### 1. Current
- **n8n** — automation
- **OpenWebUI** — LLM interface
- **Ollama** — model runtime

### 2. Future Planned
- **Ghost** — public website/blog
- **Observability stack** — Grafana, Loki, Prometheus, alerting, update-check automations
- **Identity (Authentik)** — SSO provider for private modules

### 3. Future Considerations
- **Immich** — photo/video library
- **Nextcloud AIO** — personal cloud + file storage
- **OnlyOffice** document server 
- **Obsidian (self-hosted plugins, remote vault)** — knowledge management
- **Mealie** — recipe + meal planning


## 3. Access Model

Access categories describe exposure intent only.  
Actual placement and trust boundaries are defined by network zones per ADR-0016.

### Private (VPN-only)
- Immich
- Nextcloud
- n8n
- OpenWebUI
- Observability
- Reporting

### Public (Cloudflared)
- Ghost website
- Public n8n webhooks

## 4. Roadmap Stages
### Stage 0 — Current
Mixed access model.

Some services are exposed via Cloudflared, while others are already private or locally accessed.  
This stage predates full separation of DMZ, Internal, and Management zones.

### Stage 1 — VPN Separation
Private services over Tailscale, public stays on Cloudflared.

### Stage 2 — Full Self-Hosting of Identity + Mesh Network
Move from Tailscale → Headscale.
Move from Google OIDC → Authentik.

### Stage 3 — Module Hardening & Integration
Per-module improvements (see docs/modules/*)

## 5. DMZ Evolution Roadmap

This section describes the staged introduction of the DMZ node and its current state.

### **0. Completed — DMZ Proxmox Host (hp600a)**

- hp600a running **Proxmox VE** on bare metal (bare metal Ubuntu Server approach was abandoned)
- DMZ workloads will run as **VMs under Proxmox** — consistent with hp800 operational model
- Physical uplink: guest WiFi (isolated from internal LAN)
- Proxmox management access: **Tailscale only** — not exposed on guest WiFi network
- SSH via Tailscale SSH; restricted to `{{SSH_USER}}` / `tag:sudoer` per ACL policy
- VT-x / VT-d enabled for full KVM support
- No-subscription Proxmox repositories configured
- Tailscale installed and node enrolled as `tag:dmz tag:ssh`

**Why Proxmox instead of bare metal Ubuntu:**
- Aligns with hp800 operational model — same hypervisor, same backup/restore procedures
- VM isolation between DMZ workloads
- Easier rebuild: restore from Proxmox backup rather than re-provisioning bare metal
- Future flexibility: run multiple isolated VMs (Ghost, n8n webhooks, Home Assistant, etc.)

### **1. Next — DMZ Services VM**

- Deploy DMZ services VM on hp600a (Ubuntu Server)
- Migrate Ghost and n8n webhook ingress from wherever they currently run into this VM
- Install Cloudflared tunnel client in the VM
- Install Portainer agent in the VM (not on Proxmox host)
- Apply strict Tailscale ACLs so DMZ VM can only reach specific internal ports

### **2. Next — Tailscale Integration**

- DMZ forwards webhooks → Internal services via private Tailscale mesh
- Core services (n8n, Observability, Authentik, etc.) remain fully private
- Strict ACLs already in place for hp600a management; extend to service-level flows

### **3. Later — Headscale Migration (Self-Hosted Mesh)**

- Replace the Tailscale control plane with a self-hosted **Headscale** instance
- All nodes continue using the official Tailscale client, but authenticate against Headscale instead of Tailscale's cloud
- Headscale automatically manages:
  - **WireGuard key generation**
  - **Secure key distribution**
  - **Automatic key rotation**
  - **Device identity and ACL policy**
- No manual WireGuard configuration or key rotation is required
- Establish Headscale as the authoritative mesh coordinator (recommended on an Infra VM, *not* the DMZ)
- DMZ ↔ Core communications remain encrypted, authenticated, and isolated

### **Future Improvement — Proxmox Cluster (hp800 + hp600a)**

Both nodes run Proxmox VE 9.x. Clustering would provide a unified management view from a single web UI and enable live migration of VMs between nodes.

**Deferred because:** hp600a is on guest WiFi (192.168.101.0/24) and hp800 is on internal LAN — Proxmox clustering requires low-latency direct network connectivity between nodes. Revisit when network topology allows direct connectivity (e.g. dedicated VLAN or physical link).

### **Outcome**
- Public apps isolated in VMs on the DMZ Proxmox host
- Private apps protected behind zero-trust mesh
- No dependency on Tailscale's cloud (post-Headscale migration)
- Minimal blast radius if DMZ is ever compromised
- Consistent operational model across all Proxmox nodes

## 6. What This Achieves
- Private stack fully isolated
- Public entrypoints protected by Cloudflare
- Professional email with no server maintenance
- Clean separation between public vs private access
- Expandable platform for modules, automations, AI, and observability

---

**Note:**  
This roadmap describes *when* changes happen.  
*What is allowed to talk to what* and *under which trust assumptions* is defined in ADR-0015 and ADR-0016.

---

## Changelog

| Date       | Change                                                                 | Author |
|------------|-------------------------------------------------------------------------|--------|
| 2025-11-16 | Initial creation of platform roadmap                                   | Owner  |
| 2026-01-11 | Aligned roadmap with ADR-0015 and ADR-0016; clarified DMZ trust model; corrected Stage 0 access description | Owner  |
| 2026-04-19 | Updated DMZ section: hp600a now running Proxmox VE (bare metal Ubuntu abandoned); documented completed DMZ setup; added Proxmox cluster as future improvement; updated stage numbering | Owner  |
