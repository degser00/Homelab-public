# ADR-0021: DMZ Node Operational Constraints

**Date:** 2026-04-19
**Status:** Accepted
**Deciders:** Owner
**Relates to:** ADR-0016 (Network Zones and Trust Boundaries), ADR-0017 (Container Management and Control Plane Selection)

---

## Context

The DMZ node (hp600a) runs Proxmox VE with DMZ workloads as VMs. Two operational constraints emerged during initial setup that are not covered by existing ADRs:

1. How container workloads on the DMZ VM are managed (no Portainer agent)
2. How NFS storage is accessed from hp600a for backups (Tailscale IP, manual ACL flow)

These constraints follow directly from the trust model established in ADR-0016 and the container management model in ADR-0017, but require explicit documentation as they deviate from the default approach used on internal nodes.

---

## Decisions

### 1. No Portainer Agent on DMZ VM

The DMZ services VM does not run a Portainer agent. Docker workloads are managed via Docker CLI only.

ADR-0017 designates Portainer CE as the central container management control plane, with agents on all Docker hosts. The DMZ VM is explicitly excluded from this model.

Connecting the DMZ VM to the internal Portainer instance — even via Edge Agent — creates an outbound tunnel from the DMZ into internal infrastructure. If the DMZ VM is compromised, this tunnel becomes an attack vector into the control plane. The DMZ VM is treated as stateless and disposable; Docker CLI is sufficient for its operational needs.

### 2. NFS Storage Access via Tailscale IP with Manual ACL Flow

NFS shares on TrueNAS (CoreNAS) are accessed from hp600a using the TrueNAS Tailscale IP (`{{TS_NAS_IP}}`) rather than the hostname `{{TS_NAS_NAME}}`.

NFS storage access from hp600a requires temporary ACL changes on TrueNAS to authorize the hp600a Tailscale IP. This is a manual, event-driven operation performed only when taking backups. It is not a persistent open connection.

The hostname `{{TS_NAS_NAME}}` only resolves over Tailscale, but Proxmox's storage health check daemon (`pvestatd`) performs DNS resolution independently of the Tailscale overlay, causing storage to appear inactive when using the hostname. Using the direct Tailscale IP resolves this.

NFS shares cannot be added via the Proxmox web UI when the target directory structure does not already exist on the share — the UI attempts to create subdirectories and fails if the share is read-only or the directory is pre-existing. Shares must be added via CLI using `pvesm add` with `--mkdir 0`.

---

## Backup Procedure (Current)

1. Open TrueNAS ACL for hp600a Tailscale IP (`{{TS_DMZ_IP}}`) on the relevant NFS share
2. Run backup job from Proxmox UI or CLI
3. Close TrueNAS ACL after backup completes
4. Verify backup file present on TrueNAS

**Future consideration:** Replace NFS-over-Tailscale backup with a directly attached USB drive on hp600a to eliminate the ACL open/close requirement and Tailscale dependency for backups.

---

## Consequences

### Positive
- DMZ VM compromise cannot pivot into internal Portainer or internal infrastructure via agent tunnel
- NFS backup storage is not persistently accessible from the DMZ — access is explicitly granted and revoked per backup event
- Storage health checks work reliably using Tailscale IP

### Negative
- DMZ Docker workloads require SSH + CLI for management — no GUI
- Backup requires a manual ACL change on TrueNAS before and after each run
- NFS shares on hp600a must be added via CLI, not the Proxmox web UI

---

## Alternatives Considered

| Option | Reason Rejected |
|--------|----------------|
| Portainer Edge Agent on DMZ VM | Creates outbound tunnel from DMZ into internal infrastructure; unacceptable blast radius if DMZ is compromised |
| Persistent NFS mount with open ACL | DMZ having persistent storage access contradicts the DMZ → Management default-deny rule in ADR-0016 |
| Hostname-based NFS server config | `pvestatd` health checks fail to resolve hostname independently of Tailscale overlay, causing storage to appear inactive |
| Adding NFS shares via Proxmox web UI | UI creates subdirectory structure on mount; fails on pre-existing or constrained shares; CLI with `--mkdir 0` is the correct approach |
