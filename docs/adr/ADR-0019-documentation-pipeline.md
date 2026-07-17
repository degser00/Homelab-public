# ADR-0019: Homelab Documentation Pipeline — Private Authoring to Public Publishing
**Date:** 2026-04-06
**Status:** Implemented
**Deciders:** Owner

---

## Context

Homelab documentation was maintained in a single public GitHub repository. This created a recurring risk of accidentally publishing sensitive information — domain URLs, local IPs, Tailscale node names, port numbers, service credentials, and API keys — embedded in markdown files.

The risk was not theoretical: PII had been committed and pushed to the public repo before this ADR was acted upon.

A solution was needed that:
- Allows free-form documentation authoring without constant manual redaction discipline
- Automatically strips or replaces sensitive values before anything reaches a public repository
- Is low-maintenance — adding a new secret should not require editing the pipeline itself
- Keeps the public repo as a clean, community-readable mirror of the private source

---

## Decision

### Repository Structure

Two GitHub repositories:

| Repo | Visibility | Purpose |
|------|-----------|---------|
| `{{SUDO_USER}}/Homelab-private` | Private | Source of truth; real values; free authoring |
| `{{SUDO_USER}}/Homelab-public` | Public | Sanitized mirror; placeholders only; community-readable |

### Secrets Management

A local `secrets.map` file (gitignored, never committed) serves as the single source of truth for all sensitive values:

```
KEY=real_value
MY_DOMAIN={{TEST_SECRET_ROOT_DOMAIN}}
TS_NAS_NAME=my-node
...
```

The entire map is stored as a single GitHub Actions secret (`SECRETS_MAP`) in the private repo. To add or update a secret, only two steps are needed:
1. Update `secrets.map` locally
2. Run: `gh secret set SECRETS_MAP --repo {{SUDO_USER}}/Homelab-private --body (Get-Content "secrets.map" -Raw)`

The pipeline yaml never requires editing when secrets change.

### Publishing Pipeline

A GitHub Actions workflow (`.github/workflows/publish.yml`) triggers on every push to `main` of the private repo:

1. **Checkout** private repo
2. **Stage** all `.md` files to a temp directory
3. **Substitute** — parse `SECRETS_MAP` dynamically; replace every real value with `{{KEY}}` placeholder via `sed`; order in secrets.map controls substitution sequence (specific values before broad ones, e.g. subdomains before root domain)
4. **Push** sanitized copy to public repo via HTTPS using a scoped PAT token

The substitution step is fully dynamic — it iterates over every key=value pair in `SECRETS_MAP` at runtime, with no hardcoded variable names in the yaml.

### Editor

**Zed** is the primary authoring and publishing tool:
- Native Git integration (commit, push, pull within the editor)
- Native Ollama integration for inline AI assist
- Compatible with Claude Code for agentic documentation workflows

---

## Alternatives Considered

| Option | Description | Decision |
|--------|-------------|----------|
| **Pre-commit hook only (gitleaks)** | Block commits containing secrets locally | ❌ Rejected as sole solution — reactive, not preventive; requires perfect discipline; no public mirror |
| **Single repo, manual redaction** | Author carefully, never write real values | ❌ Rejected — not sustainable; human error inevitable |
| **SOPS / git-crypt** | Encrypt secret values in-file | ❌ Rejected — encrypted blobs in public repo are unreadable; defeats purpose of public documentation |
| **Self-hosted Gitea as middle layer** | Local git remote with CI before GitHub | ❌ Rejected — overkill for solo use; adds operational overhead with no meaningful benefit |
| **Private → public via SSH deploy key** | Use SSH keypair for Action push access | ❌ Rejected in practice — multiline private key handling in GitHub Actions secrets proved unreliable across runners; replaced with PAT over HTTPS |
| **Hardcoded secrets in workflow yaml** | Each secret declared as named env var in yaml | ❌ Rejected — requires yaml edit on every secret addition; not maintainable |

---

## Stack Layout

```
Local machine (Windows, Zed)
├── secrets.map (gitignored)
└── Homelab-private/ (git clone)
        │
        └── .github/workflows/publish.yml
                │
                ▼
        GitHub Actions (on push to main)
        ├── Stage .md files
        ├── Parse SECRETS_MAP → sed substitution
        └── Push sanitized copy
                │
                ▼
        {{SUDO_USER}}/Homelab-public (clean mirror)
```

---

## Future Improvements

1. **Zed secrets preview** — investigate Zed extension or Templater-style plugin that substitutes `secrets.map` values inline during editing, so the author sees real values in preview but `{{PLACEHOLDER}}` is what gets committed.
2. **RAG integration** — connect local clone of private repo to AnythingLLM workspace for conversational querying and AI-assisted documentation authoring and expansion.
3. **gitleaks pre-commit hook** — add as a local backstop to catch any accidental real values that bypass the substitution pipeline.
4. **Automated secrets.map sync** — explore a local script or Zed task that runs the `gh secret set SECRETS_MAP` command automatically on save of `secrets.map`, eliminating the manual sync step entirely.

---

## Consequences

- All documentation can be authored freely in the private repo without redaction discipline.
- Public repo is always a clean, placeholder-only mirror updated on every push.
- Adding a new secret requires updating one local file and running one command — yaml never changes.
- PAT token requires periodic rotation (set a calendar reminder at token expiry).
- Public repo history will not contain real values; force-push on every Action run means public repo has no meaningful git history — acceptable for a documentation mirror.

## Test

TEST_SECRET_BASIC={{TEST_SECRET_BASIC}}
TEST_SECRET_SUBDOMAIN={{TEST_SECRET_SUBDOMAIN}}
TEST_SECRET_ROOT_DOMAIN={{TEST_SECRET_ROOT_DOMAIN}}
TEST_SECRET_URL={{TEST_SECRET_URL}}
TEST_SECRET_IP={{TEST_SECRET_IP}}
TEST_SECRET_EMAIL={{TEST_SECRET_EMAIL}}
TEST_SECRET_PASSWORD={{TEST_SECRET_PASSWORD}}
TEST_SECRET_QUERY_URL={{TEST_SECRET_QUERY_URL}}
TEST_SECRET_PATH={{TEST_SECRET_PATH}}
TEST_SECRET_REGEXLIKE={{TEST_SECRET_REGEXLIKE}}
