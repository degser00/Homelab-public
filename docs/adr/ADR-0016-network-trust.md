# ADR-0016: Network Zones and Trust Boundaries

**Date:** 2026-01-11  
**Status:** Accepted  
**Last updated:** 2026-04-19  
**Deciders:** Owner  
**Relates to:** ADR-0003 (Zero-Trust Cloudflare Access), ADR-0015 (Platform Topology and Device Classes), ADR-0012 (Compute-Centric Data Ownership)

---

## Context

As the homelab evolves into multiple physical nodes and device classes, a flat network model no longer provides sufficient isolation or clarity. Services now span internal infrastructure, user data storage, AI compute, and externally exposed components.

A clear model is required to:
- Define network zones and trust boundaries
- Prevent implicit trust between components
- Limit blast radius in case of compromise
- Align with zero-trust principles already established

---

## Decision

Adopt an explicit network zoning model with identity-based access as the primary control mechanism.

### Network Zones

1. **DMZ**
   - Hosts externally reachable services
   - Includes reverse proxies, webhook receivers, and internet-facing endpoints
   - Treated as untrusted by default
   - Physical uplink: guest WiFi network (isolated from internal LAN)
   - **hp600a** is the DMZ Proxmox host; DMZ workloads run as VMs under Proxmox
   - Proxmox management interface (port 8006) is NOT exposed on the DMZ network — accessible only via Tailscale

2. **Internal**
   - Hosts stateless service compute and internal application services
   - Includes observability, control, DNS, and AI compute
   - Not directly exposed to the public internet

3. **Management**
   - Hosts authoritative user-data storage and backup infrastructure
   - Includes TrueNAS systems and core backup services
   - Highest trust sensitivity

---

## Device Class to Zone Mapping

- **Stateless devices**
  - Internal zone
  - Exception: stateless services explicitly designed for exposure may live in DMZ

- **User data storing devices**
  - Management zone only

- **Windows devices**
  - Context-dependent
  - Treated as untrusted endpoints unless explicitly elevated

- **DMZ Proxmox host (hp600a)**
  - Physically in the DMZ zone (guest WiFi uplink)
  - Management plane accessible only via Tailscale overlay
  - VMs hosted on hp600a are DMZ-zone workloads

---

## Trust Principles

- Default deny between zones
- Explicit allow rules required for cross-zone communication
- No implicit trust based on network location alone
- Identity and service intent take precedence over IP-based trust
- **DMZ Proxmox management must never be reachable from the DMZ network directly** — Tailscale is the only management path

---

## Communication Rules (High-Level)

- **DMZ → Internal**
  - Allowed only for explicitly defined service flows

- **Internal → Management**
  - Allowed for scoped data access and backups

- **DMZ → Management**
  - Disallowed by default

- **Management → Internal**
  - Allowed for telemetry, control, and replication

- **Lateral movement within a zone**
  - Not implicitly trusted
  - Subject to identity-based controls

---

## Tailscale Role

- Provides identity-based overlay networking across all zones
- Used to:
  - Authenticate devices and services
  - Enforce least-privilege access
  - Provide the **only management path to DMZ Proxmox host** (hp600a)
- Does not collapse zones into a flat network
- Tailscale ACLs enforce that only `tag:sudoer` devices can SSH into `tag:dmz` nodes

---

## DMZ Proxmox Access Model (hp600a)

As of 2026-04-19, hp600a is configured as follows:

- **Physical network:** guest WiFi (192.168.101.0/24 via DHCP), isolated from internal LAN
- **Management access:** Tailscale only — Proxmox web UI accessible at `{{DMZ_BOX_URL}}` via Tailscale alias
- **SSH:** Tailscale SSH enabled; access restricted to `{{SSH_USER}}` user per Tailscale ACLs; `tag:sudoer` devices only
- **No management interface exposed on guest WiFi**
- **Outbound internet:** via guest WiFi uplink (for package updates, Tailscale relay if needed)

---

## Consequences

### Benefits

- Clear and auditable trust boundaries
- Reduced impact of compromised DMZ or stateless nodes
- Alignment with zero-trust and identity-first principles
- Easier reasoning about allowed communication paths
- DMZ Proxmox management is air-gapped from the DMZ network itself

### Costs / Tradeoffs

- Increased planning overhead for new services
- Requires discipline to avoid "temporary" trust exceptions

---

## Alternatives Considered

- **Flat network with VLANs only**
  - Rejected due to implicit trust and poor blast-radius control

- **Per-host firewall rules without zones**
  - Rejected due to lack of shared architectural intent

- **Exposing Proxmox web UI on DMZ network**
  - Rejected — management plane must not be reachable from untrusted network

---

## Future Improvements

- **Proxmox cluster across hp800 and hp600**: Deferred pending network topology change. Requires low-latency direct connectivity between nodes which is not available while hp600 is on guest WiFi.

---

## Notes

This ADR intentionally avoids implementation details (firewalls, ports, subnets).  
Those are handled in configuration and operational documentation.
