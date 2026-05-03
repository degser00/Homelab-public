# Documentation

This folder contains the architectural and operational documentation for my homelab infrastructure. It is organized into three main areas — architectural decisions, opportunity canvases, and networking reference — alongside a top-level platform roadmap.

See `modules/` at the repository root for per-host, per-service configuration and operational documentation.

---

## Structure

### `adr/` — Architecture Decision Records

Architectural Decision Records capture significant infrastructure choices: what was decided, why, what alternatives were considered, and what consequences follow. Each ADR is numbered sequentially and named to reflect its subject.

ADRs are written at decision time and treated as immutable history. If a decision is reversed or superseded, a new ADR is written rather than editing the original. This preserves the reasoning trail over time.

Current ADRs span topics including zero-trust networking, observability, container management, backup infrastructure, VPN stack, and the agentic news intelligence pipeline.

### `oc/` — Opportunity Canvases

Opportunity Canvases describe problems worth solving before committing to implementation. Each canvas covers the problem, who it affects, the value of solving it, the proposed solution, key assumptions, dependencies, and open questions.

They serve as a lightweight design gate — a way to think a feature through end to end before it becomes a GitHub issue or ADR.

### `networking/` — Networking Reference

Networking documentation covering the physical and logical topology of the homelab, port assignments, and network-specific notes. Folders may contain their own readme explaining their specific scope.

### `platform-roadmap.md`

A top-level document tracking the overall platform direction. It sits at the root of `docs/` intentionally — it spans all areas and does not belong to any single category.

---

## Conventions

- ADRs are numbered with zero-padded four-digit prefixes: `ADR-NNNN-subject.md`
- Opportunity Canvases follow the same pattern: `OC-NNNN-subject.md`
- Decisions are never edited retroactively — superseded ADRs are noted and a new record is created
- Documentation commits accompany infrastructure changes, not after
