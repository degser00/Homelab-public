# Network Topology

This document defines the **current network topology** after consolidation around Cloudflare + DMZ ingress + internal Proxmox + NAS tiers. It focuses on trust zones, communication paths, and allowed flows.

**Last updated:** 2026-04-19

---

## Trust Zones

### 1. Public Zone
- Internet users
- Cloudflare edge
- No direct access to internal services

### 2. Guest WiFi DMZ
- Untrusted network
- Hosts **hp600a (DMZ Proxmox host)**
- DMZ workloads run as VMs under Proxmox — not directly on host OS
- Public-facing workloads only
- No direct LAN or storage access
- Proxmox management interface NOT exposed on this network — Tailscale only

### 3. Tailscale VPN (Private Mesh)
- Encrypted overlay network
- Internal-only access
- Strong identity-based access
- Only management path to hp600a Proxmox host

### 4. Internal Infrastructure
- Proxmox cluster (hp800)
- NAS systems
- Observability stack
- Never exposed publicly

---

## Node Responsibilities

### hp600a — DMZ Proxmox Host (Guest WiFi / DMZ)
- **Proxmox VE** on bare metal (replaced bare metal Ubuntu Server)
- DMZ workloads run as VMs under Proxmox
- Physical uplink: guest WiFi (192.168.101.0/24 DHCP)
- Management: Tailscale only (tag:dmz, tag:ssh)
- Proxmox web UI accessible only via Tailscale alias `{{DMZ_BOX_URL}}`
- SSH via Tailscale SSH; restricted to `{{SSH_USER}}` user, `tag:sudoer` devices only
- VT-x / VT-d enabled for full KVM support
- No-subscription Proxmox repositories configured
- Cloudflared tunnel client (in DMZ services VM — to be deployed)
- Only outbound internet via guest WiFi uplink

### DNS VM (Internal Proxmox / hp800)
- CoreDNS
- Caddy reverse proxy
- Central traffic broker
- Entry point for Cloudflare tunnel traffic and VPN user traffic

### CoreNAS
- TrueNAS
- 5×8 TB Z2 + 2 TB NVMe
- Primary storage
- Log sink for backups
- No public exposure

### MiniNAS
- Windows-based NAS
- SATA passthrough
- Backup target and orchestrator
- Forwards offsite backups to Backblaze

### Observability Stack
- Metrics
- Logs
- Monitoring
- Internal only

### VPN User
- Authenticated Tailscale user
- Accesses internal services only via DNS VM

---

## Topology Diagram

```mermaid
graph TB
    subgraph "Guest WiFi (DMZ)"
        hp600a[hp600a — Proxmox VE<br/>DMZ workloads as VMs<br/>{{DMZ_BOX_URL}} — Tailscale only]
    end

    subgraph "Tailscale VPN"
        VPNuser[VPN user]
        subgraph "Internal Proxmox (hp800)"
            DNS[DNS VM<br/>CoreDNS + Caddy]
            MININAS[MiniNAS<br/>Win + SATA passthrough]
            OBS[Observability Stack]
        end
        CORENAS[CoreNAS<br/>TrueNAS 5×8TB Z-2 + 2TB NVMe]
    end

    WEB[Internet Users] -->|443| CF[Cloudflare]
    CF -->|tunnel| hp600a

    hp600a -->|Tailscale only| DNS
    VPNuser -->|internal access| DNS

    DNS -->|resolve / proxy| CORENAS
    DNS -->|resolve / proxy| MININAS
    DNS -->|resolve / proxy| OBS

    CORENAS -->|Robosync| MININAS
    MININAS -.->|logs| MININAS
    MININAS -->|robosync logs| CORENAS
    MININAS -->|Backblaze| CLOUD[Backblaze Cloud]
```

---

## Communication Matrix

### Public to DMZ
| Source | Destination | Method | Allowed |
|------|------------|--------|--------|
| Internet | Cloudflare | HTTPS 443 | Yes |
| Cloudflare | hp600a DMZ VM | Tunnel | Yes |

### DMZ to Internal
| Destination | Purpose | Allowed |
|------------|--------|--------|
| DNS VM (via Tailscale) | Reverse proxy, routing | Yes |
| Any other internal node | Direct access | No |

### DMZ Management
| Source | Destination | Purpose | Allowed |
|------|------------|--------|--------|
| tag:sudoer device | hp600a Proxmox web UI | Management via Tailscale | Yes |
| Any device on guest WiFi | hp600a Proxmox web UI | Direct access | No |

### VPN User to Internal
| Destination | Purpose | Allowed |
|------------|--------|--------|
| DNS VM | Entry point | Yes |
| Other nodes | Direct | No |

### Internal Flows
| Source | Destination | Purpose | Allowed |
|------|------------|--------|--------|
| DNS VM | CoreNAS | Service proxy | Yes |
| DNS VM | MiniNAS | Service proxy | Yes |
| DNS VM | Observability | Metrics and UI | Yes |
| CoreNAS | MiniNAS | Backup (Robosync) | Yes |
| MiniNAS | CoreNAS | Backup logs | Yes |
| MiniNAS | Backblaze | Offsite backup | Yes |

---

## Network Rules Summary

### hp600a (DMZ Proxmox host)
- No LAN trust
- No NAS access
- No lateral movement
- Single narrow Tailscale tunnel to internal services only
- Proxmox management NOT exposed on guest WiFi — Tailscale only

### DNS VM
- Central choke point
- All ingress funnels through it
- No public exposure

### Internal Nodes
- No public ingress
- No DMZ trust
- Mesh-only access

---

## Architectural Outcome

- Single controlled ingress point
- Minimal blast radius
- Clear separation of ingress, routing, storage, and backups
- Strong zero-trust posture without over-complexity
- DMZ compute is VM-isolated under Proxmox, consistent with internal compute model

---

## Future Improvements

- **Proxmox cluster (hp800 + hp600a):** Both nodes run Proxmox VE 9.x. Unified management view deferred until network topology allows low-latency direct connectivity between nodes.

---

## Changelog

### v4 – 2026-04-19
- Added {{DMZ_BOX_URL}} URL to topology diagram

### v3 – 2026-04-19
- Replaced bare metal Ubuntu Server DMZ with hp600a running Proxmox VE
- DMZ workloads now run as VMs under Proxmox, not directly on host OS
- Documented Tailscale-only management access model for hp600a
- Updated topology diagram to reflect hp600a
- Added DMZ management communication rules
- Added future improvement: Proxmox cluster

### v2 – 2025-12-23
- Replaced Core/Infra split with DNS VM as central ingress choke point
- Introduced Guest WiFi DMZ with narrow tunnel model
- Clarified VPN user access path restrictions
- Split primary storage and backup responsibilities across CoreNAS and MiniNAS

### v1 – Initial
- Original Core / Infra / DMZ topology
