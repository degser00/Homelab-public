# ADR-0020: Agentic News Intelligence Pipeline

**Status:** Accepted  
**Date:** 2026-04-18  
**Deciders:** Mark Segalis  

---

## Context

The goal is a personal news intelligence pipeline that monitors a configurable set of
sources (LinkedIn feed, RSS, newsletters), understands what is relevant to defined
topic interests, normalises findings into structured data, and surfaces them for
triage and deeper research — ultimately feeding a Ghost blog authoring workflow.

The pipeline must be:
- Fully self-hosted and privacy-preserving
- Able to browse authenticated sources (LinkedIn) without brittle credential management
- Orchestrated centrally through the existing n8n instance
- Low-maintenance for personal, low-volume use

---

## Decision Drivers

- Existing infrastructure: TrueNAS Scale 25 (Core NAS, Minisforum N5 Pro), n8n,
  Baserow, Ghost, Kasm VNC ({{KASM_CDP_HOSTNAME}}), Tailscale mesh network
- Two local LLM endpoints already operational:
  - `{{OLLAMA_AMD_HOSTNAME}}` — 890M iGPU (ROCm), Ollama, large-model lane
  - `{{OLLAMA_B50_HOSTNAME}}` — Intel Arc Pro B50 16GB (llama.cpp Vulkan), fast lane
- Claude Pro subscription available but rate-limited; preferred for interactive use
  not automated pipelines
- Preference for no-code orchestration (n8n) over custom agent code
- Security posture: personal homelab, DMZ-segmented, Tailscale-gated

---

## Decisions

### 1. Agent Framework: Hermes Agent (Nous Research)

**Chosen over:** OpenClaw, CoWork-OS, Claude Dispatch, n8n-native AI agents

**Rationale:**
- Zero CVEs vs OpenClaw's 5 formal CVEs (CVSS up to 9.9), 137 advisories in 63 days,
  and an actively poisoned ClawHub skills marketplace (824+ malicious skills confirmed)
- OpenClaw's skills are manually written and maintained; Hermes auto-generates and
  self-improves skills from task experience — compound value for recurring workflows
- Cleaner headless Docker deployment — Docker backend is the recommended default,
  not an afterthought
- Native webhook adapter (port {{HERMES_WEBHOOK_PORT}}, HMAC-signed) allows n8n to trigger Hermes tasks
  directly — fits the "n8n owns all cron" architectural principle
- CDP browser integration supports pointing built-in browser tools at an existing
  authenticated session (see §4)
- Python-based — easier to write custom REST skills for Baserow, Paperless-ngx,
  OpenArchive if needed later
- Built-in cron scheduler kept as fallback only; n8n is the authoritative scheduler
- 16 messaging platforms including Telegram for manual ad-hoc task dispatch
- Self-improving memory compounds value for a recurring daily research workflow

**Rejected alternatives:**
- *OpenClaw:* CVE record and supply chain attack make it unsuitable for a machine
  with shell access to NAS infrastructure
- *CoWork-OS:* More feature-complete but more complex; CDP browser path less
  documented; Node.js makes custom REST skills harder
- *Claude Dispatch:* Requires Claude Pro subscription runtime, Windows/Mac desktop
  app, not headless-server-native, Anthropic blocked third-party harnesses April 2026
- *n8n AI Agent node alone:* Lacks persistent memory, self-improving skills,
  and browser automation depth needed for feed reading

---

### 2. Agent Host: {{KARAKURI_HOSTNAME}} (isolated VM, DMZ box)

**Rationale:**
- Named for traditional Japanese mechanical puppets — autonomous, deterministic,
  constrained — appropriate for an agent with shell access
- Runs as an isolated VM on the DMZ box, not on Core NAS, to limit blast radius
  if the agent is compromised or misbehaves
- Connected to other services exclusively via Tailscale with explicit allowlist:
  - `{{OLLAMA_AMD_HOSTNAME}}` (Core NAS, port {{OLLAMA_AMD_PORT}})
  - `{{OLLAMA_B50_HOSTNAME}}` (Core NAS, port {{OLLAMA_B50_PORT}})
  - `{{KASM_CDP_HOSTNAME}}` (Core NAS, Kasm CDP port — see §4)
  - n8n (inbound webhook trigger + outbound result POST)
  - Baserow (via n8n only — Hermes does not write directly, see §6)
- No direct internet access from karakuri; all external browsing goes through the
  Kasm Firefox session on Core NAS

---

### 3. LLM Endpoints

**Primary (scraping + evaluation):** `qwen3.5:27b` via `{{OLLAMA_AMD_HOSTNAME}}:{{OLLAMA_AMD_PORT}}`  
**Secondary (JSON normalisation):** `qwen3:14b-Q4_0` via `{{OLLAMA_B50_HOSTNAME}}:{{OLLAMA_B50_PORT}}`

**Rationale:**
- 890M iGPU with 96GB unified RAM handles large models; qwen3.5:27b provides the
  reasoning depth needed for relevance judgement and content understanding
- B50 16GB VRAM handles qwen3:14b at Q4_0 comfortably; fast dedicated VRAM for
  structured output tasks with predictable schema
- Both endpoints OpenAI-compatible; Hermes uses `provider: custom` for each

**Model review cadence:** Models should be reviewed quarterly or when a clearly
superior alternative becomes available. The Baserow sources table (see §5) and
Hermes config are the two places requiring updates on model change.

**Hermes config (abbreviated):**

```yaml
model:
  default: qwen3.5:27b
  provider: custom
  base_url: http://{{OLLAMA_AMD_HOSTNAME}}:{{OLLAMA_AMD_PORT}}/v1
  context_length: 32768   # must be set server-side via OLLAMA_CONTEXT_LENGTH

custom_providers:
  - name: b50
    base_url: http://{{OLLAMA_B50_HOSTNAME}}:{{OLLAMA_B50_PORT}}/v1
    api_key: none
```

**Important Ollama context length note:** Ollama defaults to 4,096 tokens under
certain VRAM conditions. For agent use with tool schemas, minimum 16k–32k is
required. Set server-side:

```bash
OLLAMA_CONTEXT_LENGTH=32768 ollama serve
# or via systemd override:
Environment="OLLAMA_CONTEXT_LENGTH=32768"
```

This cannot be set from the client — it must be configured on the Ollama server.

---

### 4. Browser: Kasm Firefox CDP via {{KASM_CDP_HOSTNAME}}

**Chosen over:** Headless Playwright with cookie persistence, cloud browser services

**Rationale:**
- {{KASM_CDP_HOSTNAME}} (Kasm VNC on Core NAS) is always-on and already authenticated
  into LinkedIn and other sources
- Avoids the LinkedIn session management problem entirely: no repeated logins,
  no li_at/JSESSIONID rotation, no CAPTCHA handling
- Agent connects to the existing authenticated session via CDP — sees exactly what
  the logged-in user sees
- Side benefit: agent activity is visible in real time by opening {{KASM_CDP_HOSTNAME}}
  — free monitoring interface

**CDP setup required (not yet done):**
- Kasm Firefox workspace needs `--remote-debugging-port=9222` added to `APP_ARGS`
- nginx proxy inside Kasm container forwards external port `42###` → internal `9222`
  (Chrome/Firefox only accept CDP from localhost; nginx works around this)
- Exact port TBD; document as `KASM_CDP_PORT` environment variable in Hermes config
- Kasm Docker Run Config Override:

```json
{
  "environment": {
    "APP_ARGS": "--remote-debugging-port=9222"
  },
  "custom_port_map": {
    "firefox_debug": {
      "container_port": "42###/tcp/http/basic"
    }
  }
}
```

- nginx config inside container proxies `42### → 9222` following Kasm CDP guide
- Hermes browser config points at CDP endpoint:

```yaml
browser:
  headless: true
  noSandbox: true
  defaultProfile: openclaw     # managed Playwright profile
  cdpEndpoint: "ws://{{KASM_CDP_HOSTNAME}}:42###"
```

- Firefox CDP support is stable for read/navigate/snapshot operations
- Limitation: very advanced DOM manipulation not supported in Firefox CDP;
  not relevant for feed reading use case

---

### 5. Source Configuration: Baserow Sources Table

A dedicated `agent_sources` table in Baserow allows dynamic addition/removal of
sources without redeploying or reconfiguring Hermes.

**Minimum schema:**

| Field | Type | Notes |
|---|---|---|
| `name` | Text | Human-readable label |
| `url` | URL | Entry point for the source |
| `type` | Select | `linkedin_feed`, `rss`, `newsletter`, `web` |
| `enabled` | Boolean | Toggle without deleting |
| `topics` | Long text | Comma-separated interest keywords for relevance hint |
| `last_scraped` | Date | Updated by agent on completion |
| `notes` | Long text | Any special instructions for the agent |

Hermes scraping task reads this table at the start of each run via Baserow REST API.
New sources are added by updating the table — no code changes required.

---

### 6. Data Flow: Hermes → n8n → Baserow (not Hermes → Baserow directly)

**Rationale:**
- Minimises Hermes's network access surface; karakuri only needs to reach n8n,
  not Baserow directly
- n8n acts as the write gateway: validates, enriches if needed, and writes to Baserow
- Keeps all data pipeline logic in n8n where it is visible and maintainable
- Allows n8n to add retry logic, error handling, and downstream triggers
  (e.g. notify on high-relevance items) without touching Hermes config

**Flow:**

```
n8n cron (3am)
  → POST to karakuri:{{HERMES_WEBHOOK_PORT}}/webhooks/scrape-task    (trigger scraping job)
      Hermes scrapes sources from Baserow sources table
      Hermes POSTs raw results JSON to n8n webhook
  → n8n receives raw results
  → POST to karakuri:{{HERMES_WEBHOOK_PORT}}/webhooks/evaluate-task  (trigger evaluation job)
      Hermes scores, tags, normalises to schema
      Hermes POSTs structured JSON to n8n webhook
  → n8n writes rows to Baserow daily_feed table
  → n8n triggers downstream (notifications, flagging logic, etc.)
```

**Hermes output payload (minimum):**

```json
{
  "items": [
    {
      "title": "string",
      "source_url": "string",
      "source_name": "string",
      "summary": "string",
      "tags": ["string"],
      "relevance_score": 0.0,
      "scraped_at": "ISO8601"
    }
  ]
}
```

---

### 7. Orchestration: n8n Owns All Cron

All scheduled triggers — agent and non-agent — are managed in n8n.
Hermes's built-in cron is disabled. This gives a single place to:
- View scheduled job history
- Pause/resume jobs during maintenance
- Adjust timing without SSH into karakuri
- Chain agent tasks with deterministic n8n steps in one workflow view

---

## Pipeline Architecture Summary

```
n8n (scheduler + orchestrator)
  │
  ├─ TRIGGER → {{KARAKURI_HOSTNAME}} (Hermes Agent, DMZ VM)
  │              │
  │              ├─ CDP → {{KASM_CDP_HOSTNAME}}:42### (Kasm Firefox, Core NAS)
  │              │         LinkedIn feed, RSS, other sources
  │              │         Already authenticated — no login needed
  │              │
  │              ├─ LLM → {{OLLAMA_AMD_HOSTNAME}}:{{OLLAMA_AMD_PORT}} (890M, qwen3.5:27b)
  │              │         Browsing, reading, relevance judgement
  │              │
  │              └─ LLM → {{OLLAMA_B50_HOSTNAME}}:{{OLLAMA_B50_PORT}} (B50, qwen3:14b-Q4_0)
  │                        JSON normalisation, scoring, tagging
  │
  └─ RECEIVE ← Hermes POSTs structured results back to n8n
       │
       └─ WRITE → Baserow (daily_feed table)
                   → Metabase reads Baserow Postgres directly for dashboards
```

---

## Risks and Mitigations

### R1 — Hermes shell access on karakuri can affect NAS infrastructure
**Severity:** High  
**Likelihood:** Low (nightly batch, no real-time interaction)  
**Mitigation:**
- karakuri is an isolated VM, not running on Core NAS directly
- Tailscale allowlist restricts karakuri to specific ports on specific hosts only
- Hermes terminal backend set to `docker` (sandboxed container execution),
  not `local`
- No write access granted to NAS storage paths from karakuri
- Review Hermes logs periodically for unexpected tool calls

---

### R2 — LinkedIn session expires or gets flagged
**Severity:** Medium  
**Likelihood:** Medium (LinkedIn actively detects automation)  
**Mitigation:**
- {{KASM_CDP_HOSTNAME}} session is a real logged-in Firefox — not a headless scraper
- Agent reads feed at 3am, low frequency (once daily), human-like cadence
- If session expires: manual re-login in Kasm, session persists automatically
- Do not scrape at rates exceeding normal browsing — Hermes reads feed once per run,
  not bulk profile scraping
- Monitor for LinkedIn checkpoint/challenge pages in agent logs

---

### R3 — Hermes CVE or security regression
**Severity:** Medium  
**Likelihood:** Medium (fast release cadence, past webhook RCE in v0.8.0)  
**Mitigation:**
- Pin to a specific Hermes version in Docker image tag; update deliberately
- Subscribe to NousResearch/hermes-agent GitHub releases for security advisories
- Hermes webhook endpoint (port {{HERMES_WEBHOOK_PORT}}) is not exposed to internet — only reachable
  via Tailscale from n8n
- Rotate Hermes webhook HMAC secret periodically
- Do not install community skills from agentskills.io without auditing them

---

### R4 — LLM model quality degrades for relevance judgement
**Severity:** Low  
**Likelihood:** Medium (better models released regularly)  
**Mitigation:**
- Quarterly model review: compare current model output against a fixed set of
  test articles with known relevance
- Model and endpoint configuration is in one place (Hermes config.yaml);
  swap is a single-line change with no pipeline rebuild
- Baserow sources table `notes` field can carry per-source prompt hints to
  compensate for model weaknesses

---

### R5 — n8n → Hermes webhook is unauthenticated or misconfigured
**Severity:** High  
**Likelihood:** Low (Tailscale-gated, not internet-exposed)  
**Mitigation:**
- All traffic between n8n and karakuri travels over Tailscale — not routable from
  internet
- Hermes webhook routes must have HMAC secrets configured (not `INSECURE_NO_AUTH`)
- n8n sends signed payloads; Hermes rejects unsigned requests

---

### R6 — Kasm Firefox CDP port accidentally exposed to internet
**Severity:** Critical  
**Likelihood:** Low if configured correctly  
**Mitigation:**
- CDP port mapped in Kasm with `basic` auth type in port map config
- Double-check cloudflared tunnel does not accidentally expose the CDP port
- CDP endpoint accessible from karakuri only via Tailscale
- Verify with `curl http://{{KASM_CDP_HOSTNAME}}:42###/json` from outside Tailscale
  returns connection refused

---

### R7 — Baserow sources table becomes stale
**Severity:** Low  
**Likelihood:** Medium  
**Mitigation:**
- `last_scraped` field gives visibility into which sources are being hit
- `enabled` toggle allows quick deactivation without deletion
- Monthly review of sources table as part of pipeline health check

---

## What Is Not Decided Yet

The following items are deferred and should be addressed before go-live:

- [ ] Exact Kasm CDP port number (`42###` — TBD)
- [ ] nginx proxy config inside Kasm Firefox workspace container
- [ ] Hermes Docker Compose file for karakuri VM
- [ ] Tailscale ACL rules for karakuri node
- [ ] Baserow daily_feed table full schema (fields beyond the output payload above)
- [ ] n8n workflow design (webhook receiver, Baserow writer, downstream triggers)
- [ ] Metabase connection to Baserow Postgres (direct DB, not via n8n)
- [ ] Model review schedule and evaluation methodology
- [ ] Ghost publishing integration (future phase)
- [ ] Paperless-ngx / OpenArchive integration (future phase, optional)

---

## References

- Hermes Agent: https://github.com/NousResearch/hermes-agent
- Hermes docs: https://hermes-agent.nousresearch.com/docs
- Kasm CDP guide: https://kasmweb.atlassian.net/wiki/spaces/KCS/pages/241238017
- OpenClaw CVE tracker: https://github.com/jgamblin/OpenClawCVEs
- CVE-2026-25253 detail: https://nvd.nist.gov/vuln/detail/CVE-2026-25253
- llama.cpp B50 Vulkan confirmed: https://github.com/ggml-org/llama.cpp/issues/18808
- B50 TrueNAS setup guide: https://www.kevinoftech.com/Blog/Post/2025-10-21-intel-arc-b50-truenas
- Playwright CDP connect: https://playwright.dev/python/docs/api/class-browsertype
