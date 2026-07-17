# ADR-0014: Dedicated Management VM for Control Plane Tooling

**Date:** 2026-01-10  
**Status:** Accepted  
**Deciders:** Owner  
**Relates to:** ADR-0006 (Observability Core), ADR-0007 (Telemetry & Action Flow), ADR-0013 (Host-Level Log Collection)

---

## Context

The homelab operates multiple infrastructure layers, including:

- Proxmox hypervisors
- Docker-based workloads and VMs
- TrueNAS SCALE
- Observability and logging stack

Several tools require **high-privilege, control-plane access**, such as:
- Docker host management (Portainer)
- Proxmox administration and orchestration
- Future automation or inventory tooling

Colocating such tools with workloads, observability, or DNS increases blast radius and weakens separation of duties.

At the same time, the homelab favors **simplicity and low operational churn**, avoiding unnecessary VM sprawl.

---

## Decision

Introduce a **single, dedicated Management VM** acting as the **control plane** for infrastructure administration.

This VM centralizes all high-privilege management tooling while remaining isolated from workloads and observability components.

---

## Scope

### Runs on the Management VM
- **Portainer** (Docker control plane UI)
- Proxmox management tooling (API access, scripts, hooks)
- Future infrastructure orchestration tools (if needed)

### Explicitly does NOT run
- Observability stack (Grafana, Loki, Prometheus, Alloy)
- DNS services
- Application workloads
- Data plane services

---

## Access Model

### Primary access
- **SSH via Tailscale only**
- No password-based SSH login
- No public network exposure
- No inbound access outside the Tailnet

This SSH access model is uniform across all infrastructure VMs (DNS, observability, management).

### Fallback access
- **Proxmox console access**
- Used only if:
  - Tailscale is unavailable
  - Network configuration is broken
- Considered acceptable due to:
  - Low frequency of change
  - Intentionally boring, stable configuration

This trade-off is explicitly accepted to reduce complexity.

---

## Security Model

- Management VM treated as **high-trust**
- Minimal installed services
- No application data stored
- Credentials scoped to infrastructure APIs only
- Compromise impact limited to control plane, not data plane

---

## Consequences

✅ Clear separation of control plane vs workloads  
✅ Reduced blast radius compared to colocation  
✅ Avoids unnecessary VM sprawl  
✅ Consistent access model across all infrastructure  

⚠️ Requires Proxmox console access in rare recovery scenarios  
⚠️ Management VM becomes a critical asset and must be backed up  

---

## Alternatives Considered

| Option | Reason Rejected |
|------|----------------|
| Portainer colocated with Observability VM | Violates separation of duties; control-plane tooling would share a trust boundary with read-only telemetry infrastructure. |
| Portainer colocated with DNS VM | Couples always-on core infrastructure with administrative tooling; increases blast radius during Docker restarts or management actions. |
| Portainer on each Docker host | Fragmented control plane; no single source of administration; higher operational overhead. |
| Coolify / CapRover | Introduces a PaaS abstraction that conflicts with explicit, infrastructure-first design; assumes Git-driven continuous deployment and ownership of ingress and lifecycle. |
| CasaOS / app-store style UI | Appliance-oriented and opinionated; unsuitable for multi-host, infrastructure-grade management. |
| Kubernetes + Helm | Excessive complexity and operational churn for homelab scale; rejected in favor of explicit Docker-based management. |
| One VM per management tool | Overengineering for homelab scale; unnecessary fragmentation of control plane. |
| No central management UI (CLI only) | Functional but reduces visibility and ergonomics; Portainer chosen as a lightweight, non-opinionated control plane. |

---

## Status

This ADR defines the long-term control-plane layout for the homelab.
All future management tooling must be evaluated against this separation model.

## Changelog

| Date       | Change Type | Description | Decider |
|------------|-------------|-------------|---------|
| 2026-01-10 | Created | Initial definition of a dedicated Management VM for control-plane tooling, including Portainer and Proxmox administration, with Tailscale-only SSH access and Proxmox console as fallback. | Owner |
| 2026-01-11 | Updated | Added and expanded “Alternatives Considered” section to document rejected Portainer placement options and platform-level management tools. | Owner |
