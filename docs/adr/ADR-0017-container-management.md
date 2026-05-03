# ADR: Container Management and Control Plane Selection

## Status

Accepted (updated 2026-04-19 to reflect DMZ Proxmox rollout)

## Context

The platform is evolving from a single-node TrueNAS-hosted setup to a multi-node architecture with distinct roles:

* dns
* svcs
* ai
* obs
* dmz
* ctrl

Key characteristics of the environment:

* Services increasingly depend on cross-node connectivity (e.g. SVCS to AI)
* DNS entries, network access, and observability should ideally be automated or at least consistent
* Native TrueNAS SCALE Apps are largely not viable due to lack of control over the internal Docker networking

  * Internal networks are hidden and non-deterministic
  * Service-to-service connectivity is brittle
  * n8n, intended as a glue/orchestration service, is effectively isolated and requires complex workarounds to reach other services
* Plain Docker usage (WSL and DNS node) has already proven to be operationally painful and does not scale

The decision focuses on how to manage Docker-based services across multiple nodes with minimal manual glue and acceptable cognitive overhead.

## Decision Drivers

* Reduce day-to-day operational pain (SSH, manual wiring, drift)
* Enable cross-node service awareness
* Provide a single source of truth for running services
* Support future automation around DNS, networking, and observability
* Avoid Kubernetes or Kubernetes-like complexity

## Options Considered

### Option 1: Plain Docker + Docker Compose

* Used previously on WSL and DNS node
* Baseline reference only

### Option 2: Dockge

* Single-node Docker Compose UI
* Compose-first, file-based model

### Option 3: Portainer CE (Server + Agents)

* Central control plane with node agents
* Stack-based service management

## Decision

Adopt **Portainer CE** as the central container management and control plane, running on a dedicated **ctrl** node, with Portainer agents deployed on all Docker hosts (dns, svcs, ai, dmz, obs, TrueNAS host).

Dockge is **not adopted** at this stage.

## Rationale

### Why Portainer

* Provides shared awareness of nodes, services, and stacks across the system
* Establishes a single, queryable source of truth for "what is running where"
* Preserves and enforces service metadata (labels) that can later drive:

  * Observability auto-discovery
  * DNS automation
  * Network and access policy decisions
* Reduces reliance on SSH and node-local state
* Makes additional nodes (e.g. DMZ) easier to introduce and rebuild

### Why not Plain Docker

* Proven to be a significant operational burden
* High manual effort for networking, DNS, and lifecycle management
* No global visibility or coordination

### Why not Dockge

* Single-node only, no native cross-node awareness
* No shared system state or service inventory
* Would require manual glue for:

  * Cross-node access control
  * DNS registration
  * Observability onboarding
* Using Dockge would effectively require the operator to act as the control plane

### Why not TrueNAS SCALE Native Apps

* Internal Docker networking is abstracted and not operator-controlled
* Breaks predictable service-to-service communication
* Prevents n8n from functioning as a system glue layer without brittle workarounds
* Incompatible with a composition-first, multi-node service model

**Dockge and TrueNAS SCALE Native Apps rejected due to lack of cross-node awareness and networking control.**

## Consequences

### Positive

* Centralized visibility and control
* Reduced operational friction
* Clear foundation for future automation
* DMZ and AI nodes become easier to manage and reason about

### Negative / Trade-offs

* Introduces a central dependency (ctrl node)
* Portainer adds its own abstraction layer and metadata
* Risk of drifting toward "Kubernetes-lite" if scope is not kept in check

## Guardrails

* Portainer is used strictly for lifecycle management and visibility, not orchestration
* Docker Compose remains the deployment primitive
* No Swarm, no Kubernetes
* Services remain rebuildable and node roles explicit
* Portainer must not become a single point of failure

### Control Plane Resilience

* Portainer Server runs only on the ctrl node
* Each Docker host may optionally have a **local Portainer UI installed but disabled by default**
* In the event of ctrl node failure, a local Portainer UI can be enabled per node for emergency management
* Docker CLI and Compose on each node remain fully functional regardless of Portainer availability

This ensures ctrl node loss results in inconvenience, not outage.

## DMZ Node — Current State (2026-04-19)

The DMZ node (hp600a) is now operational as a **Proxmox VE host**, not a bare metal Ubuntu Server. This changes the DMZ workload model:

* DMZ workloads run as **VMs under Proxmox**, not directly on the host OS
* Docker containers running DMZ services will run inside dedicated VMs (e.g. a `dmz-svcs` Ubuntu Server VM)
* The Portainer agent for the DMZ node will be deployed inside the relevant VM, not on the Proxmox host itself
* This aligns the DMZ node with the same operational model as hp800: Proxmox host + VM-based workloads
* Backup and restore procedures are consistent across both Proxmox nodes

**Implication for this ADR:** When deploying the Portainer agent on the DMZ node, target the DMZ services VM, not the Proxmox host OS.

## Follow-ups

* ~~Revisit this ADR after DMZ node is operational~~ — DMZ node (hp600a) is now running Proxmox VE. ADR updated accordingly.
* Deploy DMZ services VM on hp600a and install Portainer agent inside it
* Evaluate whether any node-local UI (e.g. Dockge) provides sufficient incremental value to justify adoption
