# ADR-0026: Role-Based Development Pipeline with Human Gates

**Status:** Accepted
**Date:** 2026-07-12
**Last updated:** 2026-07-13

## Context

Development work in this repository — implementing issues, reviewing PRs, expanding test
coverage — needs a repeatable process that scales beyond one owner manually doing every
step, without collapsing the human review that keeps changes to a self-hosted platform
safe. ADR-0025 established the mechanical interchange for turning conversation into a PR
(issues in, PRs out, human gates between each stage) but deliberately left the internal
shape of that work — who does planning, who does implementation, who reviews — for this
ADR.

The owner is not a full-time developer and reviews every PR by hand (per the standing rule
in `CLAUDE.md`). A single monolithic agent taking an issue straight to a merged PR would
remove the intermediate artifacts (a task breakdown, a technical approach, a diff, a test
report) that make that review tractable, and would collapse every gate into one
after-the-fact approval. The pipeline defined here exists to keep those artifacts and gates
distinct while allowing the mechanism performing each role to change over time —
today a human-and-Claude.ai collapsed process, later distinct agents against owned
inference endpoints.

## Decision

Adopt a role-based pipeline with mandatory human approval gates at fixed points. Roles are
defined by function, not by implementation — each role's output is a GitHub-native
artifact (issue, PR, review comment), which is what makes the implementation behind any
given role swappable without changing the pipeline itself.

### Pipeline

```
Issue
  ↓
Planner        — understands the feature request; proposes task decomposition;
                 identifies affected modules; estimates effort
  ↓
🧑 approve
  ↓
Architect      — proposes technical approach; identifies interfaces;
                 drafts ADR changes if needed
  ↓
🧑 approve
  ↓
Developer      — implements ONE approved task
  ↓
Reviewer       — reviews correctness, style, security, maintainability
  ↓
Test Engineer  — expands test coverage; verifies acceptance criteria;
                 regression checks
  ↓
UI Reviewer    — only if applicable
  ↓
🧑 final approval
  ↓
Merge
```

### Principles

- **One task per Developer run.** A single Developer execution implements exactly one
  approved task, never a batch. This is what keeps PRs small, single-purpose, and
  reviewable, consistent with the "Keep PRs small and single-purpose" rule in
  `CLAUDE.md`.
- **Human gates are mandatory** at the three marked points (Planner → Architect,
  Architect → Developer, final approval before merge). No role may skip its gate or
  proceed on an assumed approval.
- **Deployment is entirely outside this pipeline.** The pipeline's terminal state is
  "merged to `main`." Deployment to live infrastructure remains manual via Portainer in
  every case, per ADR-0017 and the standing rule at the top of `CLAUDE.md`. No phase of
  this pipeline, present or future, gains a path to deployment.
- **Planner's decomposition materializes as GitHub sub-issues under an epic.** Epic #124
  is the reference example of this pattern.
- **The interchange format between all roles is GitHub-native**: issues in, PRs out,
  review comments between roles. This is the same contract ADR-0025 established for
  Stage 1 → Stage 2; this ADR extends it to every role boundary, which is what makes each
  role's implementation independently swappable.
- **Reviewer/Test findings route back to Developer with bounded retries, not open-ended
  loops.** A fixed, small number of rework cycles is expected and normal. If Reviewer or
  Test Engineer and Developer reach fundamental disagreement within that bound, the
  correct outcome is to close the PR without merging, fix the issue spec (via Planner/
  Architect, or the owner directly), and re-run the pipeline — not to keep iterating
  indefinitely on the same PR.

### Phase 1 — current implementation (collapsed roles)

The roles above are not yet distinct systems; several are collapsed into the two-stage
mechanism ADR-0025 defines:

- **Planner + Architect** ≈ a Claude.ai project conversation with the owner (per
  ADR-0025 Stage 1). Owner approval for both gates is expressed as the issue being
  created, and later as the issue or PR being `@claude`-triggered.
- **Developer** = one Claude Code GitHub Actions run against an approved issue (ADR-0025
  Stage 2).
- **Reviewer / Test Engineer** = additional `@claude`-triggered runs against the open PR,
  and/or the owner reviewing directly.
- **UI Reviewer** = not currently instantiated; this repository has no UI surface today.
- **All human gates** = the owner, across every step above.

Phase 1 satisfies every principle in this ADR (one task per run, mandatory gates, GitHub-
native interchange, bounded retries via a small number of `@claude` re-runs) using tooling
that already exists per ADR-0025. It does not yet have distinct agents per role; that is
Phase 2.

### Phase 2 — target implementation (not yet committed to a date)

Roles become self-hosted daemon agents (Hermes-class) running against owned inference
endpoints, replacing the collapsed Phase 1 mechanism role by role:

- **Endpoint mapping** follows the natural weight of each role's task: Developer runs
  against the large-model endpoint (the dev-agent-class model defined in ADR-0024,
  e.g. the 32B path); Planner, Reviewer, and Test Engineer share the concurrent
  general-purpose endpoint(s) (ADR-0024's `vllm-b-n5`/B50 class). This mapping is only
  viable because ADR-0024 makes every endpoint OpenAI-compatible and concurrent-safe —
  a prerequisite this pipeline depends on but does not itself define.
- **Trigger plumbing** reuses existing automation patterns rather than inventing new
  ones: n8n label/webhook dispatch, per ADR-0004, extended to drive pipeline-stage
  transitions instead of (or alongside) its current integrations.
- **Compliance logging becomes relevant again.** ADR-0002 (Proxy-based compliance
  logging) is on hold today because no consumer's traffic is currently self-hosted
  end-to-end. Once both the agents and the endpoints they call are self-hosted, this
  pipeline's traffic is exactly what ADR-0002's Proxy was designed to capture. Phase 2
  planning must revisit ADR-0002 rather than leave it on hold by default.
- **Migration order:** Developer and Reviewer roles migrate first — their outputs (a
  diff, a review comment) are the most mechanically bounded and easiest to validate
  against Phase 1 behavior. Planner and Architect migrate last: they are the
  highest-judgment roles (task decomposition, scoping, effort estimation, technical
  approach) and the cheapest to keep human-assisted while the rest of the pipeline is
  validated against self-hosted endpoints — a sequencing choice, not a permanent
  exemption. Per ADR-0025's target-architecture amendment, the self-hosting target
  applies to the whole pipeline, including Planner and Architect; hosted assistants
  (Claude.ai, ChatGPT, or others) may remain available as a fallback/secondary channel
  for those roles even after self-hosted capacity exists for them, consistent with
  ADR-0025.
- No date is committed for Phase 2; it is contingent on ADR-0024's serving capacity
  landing successfully and on a future dedicated GPU-node ADR (referenced as out of
  scope in ADR-0024).

## Alternatives Considered

**Single monolithic agent, issue straight to merged PR** — Rejected. Collapses every
review point into one after-the-fact approval, with no reviewable intermediate artifact
(no separate plan, approach, diff, or test report to inspect). Makes partial trust
impossible: the owner would have to fully trust or fully reject the entire output, rather
than approving a plan before implementation starts.

**Fully manual process (status quo before ADR-0025)** — Rejected as the state being
improved. Does not scale past the owner personally writing every plan, diff, and test;
motivated ADR-0025 and this ADR in the first place.

**Vendor-hosted end-to-end pipeline (e.g. a single third-party "AI SDLC" product)** —
Rejected for sovereignty and migration reasons. This platform's standing preference
(ADR-0012, ADR-0017) is compute-centric ownership and avoiding vendor lock-in; a
vendor-hosted pipeline would make every role's implementation a single opaque unit rather
than the independently swappable, GitHub-native-interchange roles this ADR requires.
Phase 1 already uses a vendor model (Claude) behind specific roles, but the interchange
contract keeps that swappable — a genuinely end-to-end vendor pipeline would not.

## Consequences

**Positive**
- Distinct, reviewable artifacts at every stage (task list, technical approach, diff,
  review comments, test report) instead of one opaque end-to-end action.
- Human gates are structurally mandatory, not a matter of discipline — Phase 1's gates
  are literally "does the owner create/trigger the next step," so a skipped gate means
  nothing happens next.
- GitHub-native interchange means Phase 1 → Phase 2 migration can happen role by role
  (per the migration order above) without changing how the owner interacts with the
  pipeline at any gate.
- One-task-per-Developer-run keeps every PR small and single-purpose, aligning with the
  existing `CLAUDE.md` PR convention and making owner review tractable at scale.

**Negative / risks**
- Phase 1's role collapse (Planner+Architect as one conversational stage, Reviewer/Test
  as ad hoc re-runs) means the roles are conceptually distinct but not yet
  operationally independent — a process discipline requirement on the owner today,
  not a system-enforced one.
- Bounded retries require a concrete bound to be useful; this ADR states the principle
  but does not fix a number. An unbounded-in-practice retry count would silently
  reintroduce the open-ended-loop failure mode this ADR rejects.
- Phase 2 reintroduces the ADR-0002 compliance-logging question at exactly the moment
  the platform has the most autonomous LLM traffic to date; deferring that revisit past
  Phase 2's start would leave agent-to-agent traffic unlogged during the highest-volume
  period.
- Phase 2 is contingent on serving capacity (ADR-0024) and a not-yet-written GPU-node
  ADR; if either slips, Phase 2 has no fallback timeline, only "not yet committed."

**Neutral**
- This ADR governs pipeline shape and gate placement, not the specific tools performing
  each role in Phase 1 — those remain defined in ADR-0025 and are expected to be
  superseded by Phase 2 tooling without changing this ADR's pipeline structure.
- Epic #124 is the first exercise of this pipeline as an explicit, named process (rather
  than ad hoc owner-driven work), even though Phase 1 predates this ADR's formal write-up.

## Out of Scope

- The specific Phase 2 daemon agent implementation (Hermes-class) and its deployment —
  future ADR(s) once a dedicated GPU-node decision is made.
- Fixing a specific bounded-retry count — left as an operational parameter, not an
  architectural one, to be tuned once Phase 1 usage data exists.
- Revising ADR-0002 itself — this ADR only establishes that Phase 2 must revisit it, not
  what that revision should conclude.

## References

- ADR-0002 — AI compliance logging (on hold; must be revisited before/at Phase 2)
- ADR-0004 — Public webhook exposure (trigger plumbing pattern for Phase 2 dispatch)
- ADR-0012 — Compute-centric data ownership (sovereignty rationale against vendor lock-in)
- ADR-0017 — Container management (Portainer/Docker; deployment stays manual and outside
  this pipeline)
- ADR-0024 — LLM serving architecture on CORE-NAS (endpoint concurrency prerequisite for
  Phase 2 role-to-endpoint mapping)
- ADR-0025 — Claude.ai ↔ GitHub integration (defines the Phase 1 two-stage mechanism this
  ADR's pipeline roles map onto; this ADR is ADR-0025's named successor for Stage 2
  execution)
- Epic #124 and sub-issues — first exercise of this pipeline
