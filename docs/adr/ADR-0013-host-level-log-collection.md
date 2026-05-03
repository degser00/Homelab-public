# ADR-0013: Host-Level Log Collection (Docker + System)

**Date:** 2026-01-10  
**Status:** Accepted  
**Deciders:** Owner  
**Relates to:** ADR-0006 (Observability Core), ADR-0007 (Telemetry & Action Flow), ADR-0002 (Compliance Logging)

---

## Context

The platform operates across multiple Docker-based environments, including:

- TrueNAS SCALE Apps (Docker backend)
- Standalone Docker hosts and VMs
- System-level services on each host

Earlier approaches relied on **per-application log sidecars** or compose-level configuration, increasing operational complexity and coupling observability to application deployment.

This approach does not scale and complicates maintenance, upgrades, and compliance guarantees.

---

## Decision

Adopt a **host-level log collection model** using **Grafana Alloy** as a single agent per node.

### Key principles
- One Alloy instance per host (VM, NAS, server)
- No per-application or per-compose log sidecars
- No application-level log shipping configuration
- Logs collected centrally from:
  - Docker engine (container stdout/stderr)
  - Host system logs (journald / syslog)

---

## Scope

### Collected logs

**System**
- OS logs (journald / syslog)
- Service lifecycle events
- Hardware and storage related messages

**Containers**
- All Docker containers on the host
- Includes TrueNAS SCALE Apps containers
- Includes standalone Docker and Compose stacks

### Excluded
- No raw AI content
- No application payloads
- No deep application introspection beyond stdout/stderr

---

## Architecture

```
[ Host / VM / TrueNAS ]
|
| Docker logs + system logs
v
[ Grafana Alloy (single agent) ]
|
v
[ Loki ] ---> [ Grafana ]
```


- Alloy tails Docker log files and system logs locally
- Logs are labeled with host, container, image, and stack identifiers
- Loki is the single source of truth for logs

---

## Security & Compliance

- Alloy runs with read-only access to logs
- No credentials stored in application containers
- Log access is governed centrally in Grafana
- Compliance filtering rules remain enforced downstream (ADR-0002)

---

## Consequences

✅ Simplified operational model  
✅ Zero per-app configuration  
✅ Works uniformly across NAS, VMs, and servers  
✅ Easier upgrades and migrations  

⚠️ Requires host-level access for Alloy  
⚠️ Label discipline is required for long-term query hygiene  

---

## Rollout

1. Deploy one Grafana Alloy instance per host
2. Enable Docker + system log scraping
3. Decommission any existing log sidecars
4. Standardize Grafana dashboards on host/container labels

---

## Status

This ADR supersedes any prior sidecar-based log collection patterns.
