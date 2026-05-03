# ADR-0012: Compute-Centric Data Ownership with Cold-Archive NAS

**Date:** 2026-01-08  
**Status:** Accepted 
**Deciders:** Owner  
**Relates to:** ADR-0011 (Backup Infrastructure Strategy), ADR-0010 (Email Archiving), ADR-0006 (Observability Core), ADR-0002 (AI Compliance Logging)

---

## Context

The current setup colocates:
- Primary data storage
- User-facing application compute
- AI compute workloads

This simplifies local access but creates tight coupling between storage and compute, increases the failure blast radius, and limits future scalability.

The target architecture separates the system into three dedicated roles:
- **NAS** for cold storage and backups
- **User compute** for applications and live user data
- **AI compute** for model execution and derived AI state

A clear definition of **data ownership** and **system of record** is required to avoid ambiguity, reduce complexity, and support independent scaling and recovery.
This ADR defines the **target architecture** for a future state in which user compute, AI compute, and NAS roles are deployed on dedicated nodes. At present, these roles may be colocated on shared infrastructure. This document describes intended responsibilities and data ownership boundaries once separation is complete.

---

## Decision

The following decision applies to the **intended steady-state architecture** and does not imply immediate changes to the current deployment topology.
Adopt a **compute-centric data ownership model**.

### Deployment Clarification (Finalized)

The finalized deployment model places **User Compute directly on TrueNAS SCALE (bare metal)**.

User-facing applications are deployed as **standard Docker/Compose stacks** and managed centrally via **Portainer**. This avoids dependency on the TrueNAS Apps catalog and its opinionated lifecycle.

- TrueNAS SCALE runs on **bare metal**
- User-facing applications run as **Docker/Compose stacks**
- Portainer is the central management interface for these stacks
- **Proxmox is reserved exclusively for infrastructure and control-plane VMs** (DNS, observability, management)

### User Compute
- Acts as the **system of record** for:
  - All live user documents
  - Application data and databases
  - Version history (for example via Nextcloud)
  - Derived data (thumbnails, previews, transcodes, indexes)
- Stores all data **locally** on attached storage
- Does **not** depend on NAS-mounted filesystems for runtime operation

### NAS
- Acts as **cold archive and backup target only**
- Stores:
  - Full resolution photos (cold storage)
  - Backup copies of documents
  - VM and system backups with retention
- Does **not** participate in live application workflows
- Does **not** serve filesystems to applications

### AI Compute
- Stores:
  - AI models
  - embeddings
  - vector databases
- Accesses user data **exclusively via APIs**
- Has no direct filesystem access to user compute or NAS data

---

## Data Flow (Target State)

```text
User Interaction
  |
  v
User Compute
  - Apps
  - Live documents
  - Versioning
  - Derived data
  |
  |  (scheduled backup)
  v
NAS
  - Cold archive
  - Long-term retention
  - VM backups

AI Compute
  |
  |  (API access only)
  v
User Compute Services
```
### Key properties
- No shared or mounted filesystems between systems  
- Backup is asynchronous and batch-oriented  
- APIs are the only integration surface between compute roles  

---

## Scope & Placement

The scope described below reflects the desired end-state role separation. During transitional phases, multiple roles may coexist on the same physical or virtual host without violating this model.

- **User compute**
  - Runs all user-facing applications
  - Owns document lifecycle and versioning

- **NAS**
  - Provides storage durability and recovery depth
  - Handles snapshots and long-term retention

- **AI compute**
  - Performs inference and analysis
  - Maintains only reproducible or derived state

Infrastructure and control-plane systems are deployed as VMs on a hypervisor.

User Compute runs on **TrueNAS SCALE (bare metal)** and hosts user-facing applications as **Docker/Compose stacks** managed via **Portainer**.

---

## Security

- No live data access from NAS to applications
- No direct filesystem access from AI compute to user data
- Reduced attack surface by minimizing cross-system trust
- Backup and archive access is write-restricted and automated
- Observability and audit logs collected centrally

---

## Consequences

### Positive
- ✔ Clear ownership and source-of-truth boundaries
- ✔ Simplified application design (local storage only)
- ✔ Reduced coupling between compute and storage
- ✔ Easier hardware replacement and scaling
- ✔ Smaller failure blast radius

### Negative
- ⚠ Higher local storage requirements on compute nodes
- ⚠ Strong dependence on backup correctness
- ⚠ Restore operations require disciplined procedures

---

## Alternatives Considered

### 1. NAS-centric data ownership  
Rejected — couples compute and storage and increases blast radius.

### 2. Shared filesystem (NFS) between NAS and compute  
Rejected — introduces operational complexity, locking, and failure ambiguity.

### 3. Distributed filesystem  
Rejected — unnecessary complexity for the scale and use case.

### 4. TrueNAS Apps catalog as the application runtime  
Rejected — the Apps ecosystem is opinionated and couples application lifecycle to TrueNAS-specific mechanisms. Standard Docker/Compose stacks managed via Portainer provide consistent control across environments and reduce platform lock-in.

---

## Future Extensions

- Tiered retention policies per data class
- Automated integrity verification of backups
- Faster rebuild workflows for compute nodes
- Policy-driven lifecycle management for cold archives
- Formal API contracts between user compute and AI compute

## Changelog

| Date       | Change Type | Description | Decider |
|------------|-------------|-------------|---------|
| 2026-01-08 | Created | Initial definition of compute-centric data ownership and separation between user compute, NAS, and AI compute roles. | Owner |
| 2026-01-10 | Updated | Finalized deployment model placing User Compute on TrueNAS SCALE (bare metal) while running user-facing apps as standard Docker/Compose stacks managed centrally via Portainer; reserved Proxmox for infrastructure and control-plane VMs only. | Owner |
