# Modules

This folder contains the configuration and operational documentation for every service running in the homelab. It is the source of truth for how services are deployed, configured, and operated.

---

## Organization Logic

Modules are organized by **host**, then by **box** within that host, then by **service** within that box.

A *host* is a physical machine. A *box* is a logical unit running on that host — typically a VM or a container group with a distinct role. A *service* is an individual application running inside a box.

This hierarchy reflects the physical and logical topology of the infrastructure. If you know which host a service runs on, you know where to find its documentation and configuration.

---

## What Lives Here

Each service folder contains what is needed to understand, deploy, and operate that service:

- **Compose file** — the container definition
- **Configuration files** — service-specific config, colocated with the compose file that references them
- **README** — what the service does, how it is accessed, and day-to-day operational notes
- **Architecture doc** — where the service warrants deeper design explanation beyond the README
- **Tech debt doc** — known issues and deferred work, kept visible rather than buried in issue trackers

Runbooks live at the host level rather than the service level where procedures span multiple services on the same host. Service-specific operational steps (restart, config reload, log inspection) belong in the service README.

---

## Relationship to Other Documentation

- **Why** a service is deployed or configured a certain way → `docs/adr/`
- **What** problem a service is intended to solve → `docs/oc/`
- **How** the network layer connects services → `docs/networking/`
- **Platform direction** → `docs/platform-roadmap.md`

Configuration and operation live here. Architecture and decisions live in `docs/`.
