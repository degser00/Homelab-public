# Technical Debt — Ghost Infrastructure

This document tracks known technical debt, security shortcuts, and
unresolved issues introduced during the initial Ghost-Core provisioning
on the `web` VM. Items should be resolved before considering the
infrastructure production-ready.

---

## Security

### 1. Staff Device Verification Disabled
**Severity:** High
**Location:** `docker-compose.yml` — `security__staffDeviceVerification: false`

Ghost's 2FA-style device verification was disabled because the admin
login session was dropping instantly due to a `url` vs `admin__url`
conflict. The email sending was confirmed working (code received) but
the session was already dead by the time the code was entered.

**Resolution required:**
- Fix the admin session/URL conflict first (see item below)
- Re-enable `security__staffDeviceVerification` by removing the override
- Validate full login + device verification flow end-to-end

---

### 2. Ghost Admin Session / URL Conflict Unresolved
**Severity:** High
**Location:** `docker-compose.yml` — `url` vs `admin__url`

Ghost is configured with `url: https://www.{{MY_DOMAIN}}` (public domain)
and `admin__url: http://{{TS_WEB_IP}}:{{PORT}}` (Tailscale IP). The session
cookie domain mismatch between the two causes the admin to log out
instantly after authentication. A workaround of temporarily setting
`url` to the Tailscale IP was used to complete initial setup, then
reverted.

**Resolution path decided:**
- Add `{{GHOST_ADMIN_URL}}` to split DNS on DNS VM — resolves to DNS VM internally, no public record
- Add Caddy upstream on DNS VM: `{{GHOST_ADMIN_URL}}` → `http://{{TS_WEB_IP}}:{{PORT}}`
- Set `admin__url: https://{{GHOST_ADMIN_URL}}` in Ghost-Core compose
- Cookie domain will be consistent — session conflict resolved
- Re-enable `security__staffDeviceVerification` after validating login

**Status:** `www.{{MY_DOMAIN}}` local routing implemented and working ✅
`{{GHOST_ADMIN_URL}}` split DNS + Caddy entry not yet configured ⏳

---

### 3. No Firewall on VM
**Severity:** Medium
**Location:** `web` VM — UFW not installed

UFW was intentionally skipped in favour of Tailscale ACL-based access
control. This is acceptable as long as Tailscale ACLs are actively
maintained and Ghost is not accidentally re-bound to `0.0.0.0`.

**Resolution required:**
- Document Tailscale ACL rules that restrict access to the `web` VM
- Confirm Ghost is bound only to `{{TS_WEB_IP}}` and not `0.0.0.0`
- Consider adding UFW as a secondary layer — defence in depth

---

## Infrastructure

### 4. Ghost Admin API Keys Not Generated
**Severity:** High — blocks DMZ sync
**Location:** Ghost Admin → Settings → Integrations

API keys for the `DMZ Sync` integration have not been created. Without
these, the content sync pipeline cannot be built or tested.

**Resolution required:**
- Resolve admin login issue first
- Generate Content API Key and Admin API Key
- Store securely in a secrets manager or encrypted file

---

### 5. No DMZ Sync Pipeline
**Severity:** High — site not publicly live
**Location:** `web` VM — sync script not created

Content sync from Ghost-Core to Ghost-DMZ via Ghost Admin API has not
been implemented. The Ghost-DMZ instance has not been provisioned yet.
Until this is done, the public site at `www.{{MY_DOMAIN}}` is not
serving Ghost content.

**Resolution required:**
- Provision Ghost-DMZ instance (see `issue-ghost-dmz.md`)
- Write and test API sync script on `web` VM
- Schedule via cron, then migrate to webhook-triggered on publish

---

## Operations

### 6. SMTP Not Fully Validated End-to-End
**Severity:** Medium
**Location:** Migadu SMTP — `docker-compose.yml`

A verification email was successfully received during setup, confirming
basic SMTP send works. However a full end-to-end flow (password reset,
member magic link, newsletter) has not been tested.

**Resolution required:**
- Test password reset email flow
- Test member signup magic link
- Confirm sender display name is correct (currently raw email address)
- Set a proper display name via `mail__from` in `.env`

---

## Labels

`security` `infrastructure` `ghost` `tech-debt` `self-hosted`
