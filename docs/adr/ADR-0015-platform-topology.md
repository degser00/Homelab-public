# ADR-0015: Homelab Platform Topology and Device Classes

**Date:** 2026-01-11  
**Status:** Accepted  
**Last updated:** 2026-07-11  
**Deciders:** Owner  
**Relates to:** ADR-0011 (Backup Infrastructure Strategy), ADR-0012 (Compute-Centric Data Ownership with Cold-Archive NAS), ADR-0006 (Observability Core)

---

## Context

The homelab has grown from a single box hosting storage and compute into a multi-node environment. The current state still includes coupled storage and compute, increasing failure blast radius and reducing clarity around trust boundaries.

A refined target landscape is needed to:
- Separate persistent user data from stateless services and from AI compute
- Standardize how workloads are deployed (Proxmox vs TrueNAS)
- Standardize remote access approach (Tailscale) across device types
- Make replacement and rebuild routine, not exceptional

---

## Decision

Adopt a topology with explicit device roles and three device classes.

### Current (Existing)

- **TrueNAS Core (main backup)**
  - Currently also running user compute and AI compute (temporary coupling)

- **HP800 Proxmox (infra VMs)**
  - Windows VM: secondary backup and bridge to cloud backup
  - Ubuntu Server VM: DNS
  - Ubuntu Server VM: Observability
  - Ubuntu Server VM: Control

- **HP600 (DMZ) — as of 2026-04-19**
  - Proxmox VE on bare metal (replacing previous bare metal Ubuntu Server)
  - DMZ workloads run as VMs under Proxmox — not directly on the host OS
  - Management access exclusively via Tailscale (no management interface exposed on DMZ network)
  - WiFi uplink to guest network (physically isolated from internal LAN)
  - VT-x / VT-d enabled in BIOS for full KVM virtualisation support
  - Aligns with hp800 pattern: same hypervisor, same backup/restore strategy, same operational model

### In the works

- **HP600 DMZ VMs**
  - ~~Migrate current DMZ workloads (Ghost, etc.) from bare metal to VMs under Proxmox~~ — **Done (2026-07-11):** DMZ workloads run as VMs under Proxmox
  - n8n webhook ingress is **not** a DMZ workload: its tunnel is co-located with n8n per ADR-0003/ADR-0004 placement rule
  - Optional future: Home Assistant OS VM, ARR-related VMs

### Target (Future)

- **User compute node**
  - TrueNAS on bare metal
  - Stores and processes user data

- **AI compute node** *(revised 2026-07-11 — was: Proxmox host running Ubuntu Server VM(s))*
  - Ubuntu Server on bare metal (Docker + Portainer agent, Tailscale at OS level)
  - AI workloads separated from user-data storage
  - Rationale for bare metal: the AI backend is stateless by definition — no
    authoritative data, rebuildable from configuration — so VM isolation adds no
    recovery value here, while GPU passthrough adds a performance tax and
    operational complexity on a single-purpose inference box. Rebuild procedure
    is documented at the host level instead (same guarantee, different mechanism)
  - This is a deliberate, scoped exception to the "Stateless runs on Proxmox"
    device-class default below; it does not apply to any other node

---

## Key Properties

### Device classes

1. **Stateless (ephemeral service compute)**
   - Runs on Proxmox as Ubuntu Server VMs
   - **Exception:** the dedicated AI compute node runs Ubuntu Server on bare
     metal (see Target section) — GPU workloads justify skipping the hypervisor;
     rebuildability is preserved via documented host-level rebuild procedure
   - Does not store authoritative user data
   - Rebuildable from configuration
   - Tailscale runs on the OS (host or VM, depending on deployment)

2. **User data storing (authoritative storage + user-data processing)**
   - Runs on TrueNAS on bare metal
   - Stores and processes user data
   - Prioritizes integrity, backups, and controlled change
   - Tailscale runs as a native app

3. **Windows devices**
   - Used where Windows is required (backup bridge, tooling, clients)
   - Tailscale for Windows

### Invariants

- Authoritative user data lives only on **User data storing** class devices.
- **Stateless** devices are replaceable and must not be required for data availability.
- **AI compute** must not become a hard dependency for core user-data access and backups.
- **AI compute access model**
  - AI compute interacts with user data exclusively via scoped APIs
  - No direct filesystem, dataset, or backup-archive access from AI compute
  - AI nodes may only access data actively being processed
  - Compromise of AI compute must not expose historical archives or full datasets
- Remote access is standardized on **Tailscale** across all classes.
- **DMZ Proxmox hosts** follow the same operational model as internal Proxmox hosts (hp800): same hypervisor, same backup strategy, same rebuild procedure. Bare metal OS on DMZ nodes is not used for workloads directly.

---

## Communication Principles (High-Level)

This ADR defines directionality and intent, not ports/IPs.

### Allowed

- **Stateless → User data storing**
  - Allowed for service access (read/write as required by the service)

- **User data storing → Stateless**
  - Allowed for telemetry, control callbacks, and integrations where needed

- **Control/Observability (Stateless) → All classes**
  - Allowed for management and monitoring access as required

- **Windows → User data storing**
  - Allowed for backup workflows and administrative access where appropriate

### Disallowed / Avoided

- **User data storing depending on Stateless for core availability**
  - No "data is only reachable if service VM exists" designs

- **DMZ → Internal by default**
  - DMZ access must be explicitly scoped (principle only; details in network ADR)

- **DMZ Proxmox management interface exposed on DMZ network**
  - Proxmox web UI (port 8006) on hp600 must only be reachable via Tailscale

---

## Consequences

### Benefits

- Reduced blast radius by separating storage, user compute, and AI compute
- Clear trust boundaries for security and compliance reasoning
- Easier rebuild and migration of stateless services
- Standardized remote access model (Tailscale) by device class
- DMZ on Proxmox enables unified backup/restore strategy across all compute nodes
- DMZ workloads are now VM-isolated, not running directly on host OS

### Costs / Tradeoffs

- More nodes to manage and monitor
- Requires explicit planning of network boundaries and allowed traffic
- Migration effort from current coupled state to target topology

---

## Alternatives Considered

- **Single "everything" node**
  - Rejected due to coupling, large blast radius, and unclear trust boundaries

- **Run TrueNAS virtualized under Proxmox for all storage**
  - Not chosen as the default due to complexity and higher risk for authoritative storage role

- **Put Tailscale only on a gateway**
  - Rejected because per-device access control and visibility are preferred

- **Proxmox host for the AI compute node** *(added 2026-07-11)*
  - Originally the target; rejected in favour of bare metal Ubuntu Server
  - GPU passthrough imposes a performance tax and configuration complexity
  - The node is stateless and single-purpose — VM isolation and unified
    Proxmox backup add no value over a documented bare-metal rebuild

- **Bare metal Ubuntu Server on DMZ node**
  - Previously the plan; rejected in favour of Proxmox VM model
  - Bare metal Ubuntu does not align with standard rebuild/restore procedures
  - No VM isolation between workloads
  - No unified backup strategy with hp800

---

## Future Improvements

- **Proxmox cluster across hp800 and hp600**: Both nodes run Proxmox VE 9.x. Clustering would allow unified management view from a single web UI. Deferred because hp600 is on guest WiFi (DMZ) and hp800 is on internal LAN — Proxmox clustering requires low-latency direct connectivity between nodes. Revisit when network topology allows it.

---

## Follow-ups

- Create a separate ADR for **Network Zones and Trust Boundaries**
  - DMZ vs internal vs management
  - Traffic rules, allowed paths, and enforcement approach
  - Tailscale ACL model and identity approach
