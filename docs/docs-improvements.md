# Documentation Improvements — What Could Be Added

This document captures opinionated suggestions for expanding the documentation structure over time. Each entry covers purpose, value, trade-offs, and relevant best practices.

---

## 1. `docs/adr/index.md` — ADR Index

**Purpose**
A single navigable index of all ADRs with status (`active`, `superseded`, `deprecated`), a one-line summary, and links. Currently the ADR folder requires reading filenames to understand scope and status.

**Value**
- Makes the ADR corpus navigable without opening individual files
- Immediately useful as a portfolio artifact — a reader can scan 21 decisions in 30 seconds
- Surfaces superseded decisions clearly without hunting through filenames

**Trade-offs**
- Requires manual maintenance on each new ADR
- Can become stale if not updated consistently

**Best practices**
- Keep it as a simple markdown table: number, title, status, one-line summary
- Update it as part of the same commit that adds the ADR — treat it as part of the ADR authoring workflow
- Status values should be constrained: `active`, `superseded by ADR-XXXX`, `deprecated`

---

## 2. Runbook Coverage Per Host

**Purpose**
Ensure every host has a host-level runbook covering procedures that span multiple services: node recovery, Tailscale re-registration, storage failure response, full service restart sequence. `hp600a.runbook.md` already exists — the pattern should be consistent across all hosts.

**Current state**
- `HP600a.srv/hp600a.runbook.md` — exists
- `HP800a.srv/` — no host-level runbook visible
- `Core.srv/` — no host-level runbook visible

**Value**
- Recovery procedures for high-stress scenarios should not rely on memory
- Particularly important for CoreNAS (N5Pro) given it runs the most critical workloads: ZFS pools, Ollama, Immich, ComfyUI
- Demonstrates operational maturity in a portfolio context

**Trade-offs**
- Runbooks require maintenance as infrastructure changes — a stale runbook is worse than none
- Risk of duplication with service-level README operational notes — scope needs to be clear

**Best practices**
- Host-level runbook covers cross-service and host-recovery procedures only
- Service-level operational notes (restart, config reload, log inspection) stay in the service README
- Each runbook starts with a trigger condition ("when to use this") and prerequisites
- Written as numbered steps, not prose
- Include expected output for key commands so the operator knows if something worked
- Link to relevant ADRs for design context
- Reviewed after any significant infrastructure change to that host

---

## 3. `docs/glossary.md` — Terms and Conventions

**Purpose**
A reference for naming conventions, abbreviations, and homelab-specific terminology. Covers node names (`CoreNAS`, `web`, `karakuri`), network segments, tagging conventions, and recurring acronyms.

**Value**
- Useful after a long break — removes the mental overhead of remembering why something is named what it is
- Essential for the portfolio angle — external readers have no context for homelab-specific naming
- Prevents naming drift over time by making conventions explicit

**Trade-offs**
- Low priority while the homelab is small and familiar
- Easily neglected; becomes outdated if not maintained alongside infrastructure changes

**Best practices**
- Keep it flat — a simple alphabetical list, not categories
- Include the *reason* for a name where non-obvious
- Link from the root `README.md`

---

## 4. `docs/postmortems/` — Incident Post-Mortems

**Purpose**
Short structured write-ups of significant failures or outages: what broke, timeline, root cause, what was learned, what was changed as a result.

**Value**
- Strongest portfolio signal of operational maturity — anyone can design a system, post-mortems show you operate one
- Captures learnings that would otherwise be lost (e.g. the CoreDNS wildcard regex breaking ACME challenge resolution, TrueNAS app unresponsive after update pattern)
- Several incidents already exist in memory and ADR context — retroactive write-up is low effort now, impossible later

**Trade-offs**
- Requires discipline to write when the instinct after fixing something is to move on
- Some incidents are too minor to warrant a full write-up — judgment required

**Best practices**
- Blameless by convention — focus on system and process, not operator error
- Strict structure: summary, timeline, root cause, contributing factors, resolution, follow-up actions
- Follow-up actions link to GitHub issues
- Even a 200-word post-mortem is better than none
- Start retroactively with known incidents before the context fades

---

## Priority Order

Suggested sequence based on value-to-effort ratio:

1. **ADR index** — 30 minutes, immediately improves navigability and portfolio readability
2. **Host runbooks** — fill the gap for HP800a and Core.srv; hp600a pattern already exists to follow
3. **Post-mortems** — retroactively document known incidents while context is still fresh
4. **Glossary** — one sitting, low ongoing cost, high portfolio value
