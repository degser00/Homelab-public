# ADR-0025: Claude.ai ↔ GitHub Integration for Docs/Config Authoring

**Status:** Adopted
**Date:** 2026-07-12
**Last updated:** 2026-07-13

## Context

The owner authors architecture decisions, documentation, and Compose config for this
repository conversationally, mobile-first, and is not a developer. Execution — creating
issues, writing code, opening PRs — must stay gated behind explicit human review at each
step, and deployment to live infrastructure must always remain a manual, human-performed
action (per the standing rule at the top of `CLAUDE.md`).

The owner already maintains architectural and project context in a Claude.ai project
("Homelab services") through ongoing conversation. This repository is the source of
truth and audit layer: every decision that matters must land here as a durable,
reviewable artifact (an ADR, an issue, a PR), not remain only in chat history.

The need is a way to move from "thinking out loud in a Claude.ai conversation" to
"a PR a human can review and merge" without collapsing the human gates in between, and
without requiring the owner to manually copy-paste file contents or context on a phone.

## Decision

Integrate Claude.ai and Claude Code with this repository in two stages, connected by a
stable interchange format (GitHub issues in, GitHub PRs out) rather than a single
monolithic tool.

### Stage 1 — thinking → issue (Claude.ai project)

The Claude.ai "Homelab services" project is connected to this repository via Anthropic's
**Claude GitHub MCP Connector** (a GitHub App).

- Claude drafts issues — including epics with sub-issues — directly in this repo during
  conversation, but only creates them on the owner's explicit go-ahead per issue. Drafting
  is unrestricted; creation is gated.
- Read access lets Claude reference existing content under `docs/adr/` (and the rest of
  the repo) directly during conversation, removing the need to upload files manually to
  keep context current.
- This stage produces no code changes and touches no live infrastructure — its only
  write action is issue/sub-issue creation.

### Stage 2 — issue → PR (Claude Code GitHub Actions)

Claude Code runs as a GitHub Action (`.github/workflows/claude.yml`, using
`anthropics/claude-code-action@v1`) on GitHub-hosted runners (`ubuntu-latest`; no
self-hosted runner).

- Triggered only by explicit `@claude` mentions: issue comments, PR review comments, PR
  reviews, or issues opened/assigned whose title or body contains `@claude`. No trigger
  fires on a plain issue creation or an unrelated comment.
- The action checks out the repo, commits to a branch, and stops there — it does not
  open, merge, or otherwise dispose of a PR on its own. PR creation, review, rework
  iteration, and merge stay owner-controlled.
- Repository conventions (ADR structure, Compose rules, "never automate deployment," etc.)
  are enforced by the root `CLAUDE.md`, which the action reads as project instructions on
  every run.
- Workflow permissions are scoped to `contents: write`, `pull-requests: write`,
  `issues: write`, `id-token: write`, `actions: read` — no deployment-target credentials,
  no infrastructure access.

## Key Sub-Decisions

### Auth: subscription OAuth token, not API-key billing

`.github/workflows/claude.yml` authenticates via `CLAUDE_CODE_OAUTH_TOKEN`, a repo secret
generated with `claude setup-token`, rather than `ANTHROPIC_API_KEY`. This runs Actions
usage against the owner's existing Claude subscription at zero marginal cost, instead of
metered API billing.

**Accepted risk:** Anthropic announced, then paused (June 2026), a billing change
affecting subscription-token usage in Actions. This ADR accepts that risk rather than
pre-emptively switching to API billing, with three mitigations in place if the change
lands:

1. The issue → PR interchange format is vendor-neutral — it does not depend on
   Anthropic-specific tooling beyond the Action itself.
2. GitHub Copilot coding agent has been identified as a drop-in fallback for Stage 2 if
   subscription-token billing becomes unworkable.
3. Prepaid API credits with auto-reload disabled are available as a capped-cost fallback
   if a switch to `ANTHROPIC_API_KEY` billing becomes necessary.

### Human gates preserved end to end

- Issue creation (including sub-issues of an epic) requires explicit owner go-ahead in
  chat before Claude creates anything.
- Stage 2 execution only starts on an explicit `@claude` mention — nothing runs
  automatically off issue creation alone.
- PR merge is always a manual owner action.
- Deployment to live infrastructure is always manual via Portainer (per ADR-0017) —
  this integration has no path to deployment and never will, per the standing rule in
  `CLAUDE.md`.

### Publish-pipeline interaction (ADR-0019)

Claude-authored `.md` files are ordinary content to the ADR-0019 publish pipeline: they
flow into the sanitized public mirror (`{{SUDO_USER}}/Homelab-public`) on every push to `main`,
same as human-authored docs. Two things follow from this:

- `CLAUDE.md` requires that any new sensitive or infrastructure-identifying value
  introduced in a doc be flagged in the PR description as "needs SECRETS_MAP entry before
  merge," so the owner can add it to `secrets.map` before the value reaches `main`.
- `.github/workflows/publish.yml` has a "Verify no mapped value remains" step that fails
  the publish run if any `SECRETS_MAP` value is still present in the sanitized output —
  a backstop against an unflagged value slipping through, whether introduced by Claude or
  by hand.

## Operational Notes

- The GitHub App behind the Claude GitHub MCP Connector must be **installed** on this
  repository (repository access explicitly granted in the GitHub App settings), not
  merely *authorized* at the account level. Authorized-only access returns 404 against
  private repositories, even though it works against public ones — this is easy to miss
  since the failure mode only appears on a private repo.
- Compliance logging of this integration's own LLM traffic (Claude.ai project chat,
  Claude Code Action runs) is explicitly out of scope. ADR-0002 (Proxy-based compliance
  logging) remains on hold and unimplemented; this integration is not gated on it.

## Alternatives Considered

**API-key billing (`ANTHROPIC_API_KEY`) from the start** — Rejected for now. Metered
billing on top of an existing subscription is unnecessary marginal cost; kept as the
capped-cost fallback if the paused June 2026 billing change is reinstated.

**GitHub Copilot coding agent instead of Claude Code Action** — Rejected for Stage 2 at
this time; the owner's existing conversational context and ADR-authoring workflow already
lives in Claude.ai, and Claude Code Action shares that model family. Identified as the
drop-in fallback if Claude Code Action billing/availability becomes unworkable.

**Self-hosted Actions runner on the Ctrl VM** — Rejected. GitHub-hosted runners are
zero-infrastructure for this workload (Stage 2 is bursty, human-triggered, and has no
requirement for internal network access); a self-hosted runner would add operational
surface (patching, availability, the runner itself becoming a target) for no offsetting
benefit at current scale.

**Pure manual copy-paste (no GitHub integration)** — Rejected as the status quo being
replaced. Copy-pasting file contents and diffs between a mobile chat session and GitHub
does not scale for epics with sub-issues and does not keep the repo current with
in-conversation architectural context.

## Consequences

**Positive**
- Conversational, mobile-first authoring reaches the repo directly as issues and PRs,
  without manual file transfer, while every write action (issue creation, PR merge,
  deployment) stays a distinct human-gated step.
- Claude's read access to `docs/adr/` keeps in-conversation architectural reasoning
  consistent with what is actually accepted in the repo, reducing drift between the
  Claude.ai project's working context and the source of truth.
- Zero marginal cost for Stage 2 execution under current subscription billing.
- The issue → PR interchange format is a stable contract independent of which tool
  performs either stage, easing a future swap (Copilot, self-hosted daemon agents) without
  changing how the owner works.

**Negative / risks**
- Dependent on a paused-not-cancelled Anthropic billing change for Actions usage; if
  reinstated, requires an unplanned switch to API-key billing or a fallback tool with no
  fixed timeline.
- Subscription usage limits (the 5-hour rolling window) are already being hit under
  light, exploratory conversational use — well short of the production-scale agent
  traffic planned under ADR-0026 Phase 2 (Planner, Architect, Developer, Reviewer, Test
  Engineer roles running concurrently). This is direct evidence that any capped
  hosted-subscription model, not only pay-per-token API billing, is inadequate as the
  long-term primary substrate for either stage at planned agent-pipeline volume — see
  Migration Path below.
- Dependent on GitHub App installation state (not just authorization) on a private repo —
  a misconfiguration here fails silently as a 404 rather than an obvious permissions error.
- Two separate integration surfaces (Claude.ai MCP connector for Stage 1, GitHub Action
  for Stage 2) to keep working and re-authenticate over time, rather than one.
- No compliance logging of LLM traffic generated by this integration, consistent with the
  rest of the platform while ADR-0002 stays on hold.

**Neutral**
- This ADR governs the interchange mechanism, not the repo conventions Claude must
  follow when authoring content — those remain defined in `CLAUDE.md` and are enforced
  the same way for Claude-authored and human-authored PRs alike.
- First real use of this integration is epic #124 and its sub-issues, plus the issue that
  produced this ADR (#130).

## Migration Path

The interchange format — issues in, PRs out, human gates between each stage — is the
stable, durable contract of this decision. The concrete tooling behind each stage is not:
the roadmap target is full replacement of third-party paid/capped AI vendors with
self-hosted daemon agents (Hermes-class) running against self-hosted vLLM/Lemonade
endpoints, per ADR-0024 and a future dedicated GPU-node ADR. This applies to **both**
Stage 1 (thinking/planning) and Stage 2 (execution) — no stage is permanently exempt from
the self-hosting target. The Claude.ai MCP connector (Stage 1), and Claude Code Action on
subscription billing plus its Copilot/API-billed fallbacks (Stage 2), are transitional
risk mitigation, not the target architecture.

Migration is contingent on serving capacity from ADR-0024 and a future dedicated
GPU-node ADR, and on the role pipeline defined in ADR-0026 (Planner/Architect/Developer/
Reviewer/Test Engineer), which Stage 1/Stage 2 above map onto in its Phase 1. Stage 2 is
expected to migrate first; Stage 1 (the Claude.ai project connector) may migrate later, on
a separate timeline.

Hosted assistants (Claude.ai, ChatGPT, or others) may continue to be used **indefinitely
as a secondary/fallback channel** — for discussion, review, or execution — even after
self-hosted capacity exists. That continued availability is a deliberate convenience, not
evidence that self-hosting is incomplete or optional as the primary target: primary
reliance moves to self-hosted infrastructure, and hosted vendors drop to
opportunistic/fallback use, not the other way around.

This ADR is expected to be superseded once the self-hosted migration happens.

## Out of Scope

- Compliance logging of Claude.ai / Claude Code traffic (ADR-0002, on hold).
- Any deployment automation — this integration commits code to branches only; deployment
  remains manual via Portainer, per `CLAUDE.md` and ADR-0017.
- Self-hosted daemon agent execution (Hermes-class) for either stage — planned under
  ADR-0026, contingent on ADR-0024 and a future dedicated GPU-node ADR.

## References

- ADR-0002 — AI compliance logging (on hold, not implemented)
- ADR-0017 — Container management (Portainer/Docker, manual deployment)
- ADR-0019 — Documentation pipeline (private authoring → public publishing)
- ADR-0024 — LLM serving architecture on CORE-NAS (serving capacity for the self-hosted
  migration target)
- ADR-0026 — Role-Based Development Pipeline with Human Gates (Accepted). Defines the
  Planner/Architect/Developer/Reviewer/Test Engineer role pipeline that Stage 1/Stage 2
  above map onto in Phase 1, and the planned successor for both stages' execution.
- `.github/workflows/claude.yml`
- `.github/workflows/publish.yml`
- `CLAUDE.md`
- Epic #124 and sub-issues; issue #130 (this ADR)
