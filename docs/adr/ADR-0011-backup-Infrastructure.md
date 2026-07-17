# ADR-0011: Backup Infrastructure and Data Protection Strategy

**Date:** 2026-01-07  
**Status:** Accepted  
**Deciders:** Owner  
**Relates to:** ADR-0006 (Observability Core), ADR-0007 (Telemetry & Action Flow)

---

## Context

The system requires a backup strategy that balances:
- Performance for applications
- Long-term archive durability
- Incremental growth over time
- Operational simplicity for a personal environment

Primary storage is provided by **CoreNAS**, which hosts both application data and long-term archives.  
Backups must protect against:
- Disk failure
- Data corruption
- Accidental deletion
- Local infrastructure loss (via offsite/cloud backup)

The solution must be evolvable without disruptive migrations.

---

## Current Storage Architecture

### CoreNAS
- **Fast storage**
  - Single M.2 NVMe (non-redundant)
  - Used for:
    - Application data
    - High-IO workloads
- **Archive storage**
  - ZFS pool with 5 drives
  - 2-drive failure tolerance
  - Used for:
    - Main data archive
    - Backups of fast storage

Current capacity planning:
- Archive pool sufficient for at least 5 years
- Future scaling by swapping drives for larger capacity
- NVMe tier planned to be upgraded to:
  - 2x larger NVMe drives
  - Added redundancy
  - Hosting all application data (including Immich thumbnails, caches, etc.)

---

## Backup and Replication Flow

### Local backups (on CoreNAS)
- NVMe fast storage is backed up **nightly** to the ZFS archive pool
- ZFS provides redundancy and snapshot-based protection

### Off-host replication
- Archive data and application backups from CoreNAS are synchronized to:
  - A smaller Windows VM
  - No local redundancy
- This VM acts as a **staging node** for cloud backups

### Offsite backup
- Windows VM backs up synchronized data to cloud storage
- Acts as protection against full site loss

---

## Synchronization Technology Decision

### Current choice: Syncthing
- Used for CoreNAS → Windows VM synchronization
- Chosen because:
  - Fewer copy and consistency errors than Robocopy
  - More resilient to partial failures
  - Acceptable performance for current data volumes

### Robocopy
- Tested previously
- Faster, but produced significantly more copy errors
- Deferred until:
  - Data ingestion pipelines are stabilized
  - Application exports are normalized
  - Backups become filesystem-agnostic and incremental

---

## Decision

Adopt a **tiered backup architecture**:

- CoreNAS as the primary storage system
- ZFS archive pool as local backup and durability layer
- Windows VM as offsite synchronization and cloud-backup bridge
- Syncthing as the current synchronization mechanism

This architecture favors **reliability and correctness over raw speed**.

---

## Consequences

### Positive
- ✔ Clear separation between primary storage, local backup, and offsite backup
- ✔ ZFS provides strong local data protection
- ✔ Cloud backup protects against catastrophic loss
- ✔ Storage can be expanded incrementally
- ✔ Minimal operational complexity

### Negative
- ⚠ NVMe tier currently lacks redundancy
- ⚠ Windows VM is a single point in the offsite chain
- ⚠ Syncthing is slower than block-based replication
- ⚠ Requires monitoring of sync and backup health

---

## Future Extensions

- Upgrade NVMe tier to redundant high-capacity drives
- Move all application data fully onto NVMe tier
- Introduce normalized, application-native backup exports
- Re-evaluate Robocopy or alternative tools once data layouts stabilize
- Add redundancy to the Windows VM backup node
- Enhance observability around backup success and lag
