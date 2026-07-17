# Ghost-Core — `web` VM

## Overview

Ghost-Core is the trusted backend for {{MY_DOMAIN}}. It runs on VM 101 (`web`)
on Proxmox (NODE), is internal-only, and serves as the single source of
truth for all content, members, and authentication. It is never publicly
exposed. Content is pushed to the DMZ frontend via Ghost Admin API on publish.

Ghost runs as **two coordinated instances**:

### Ghost-Core (this machine — trusted)
- Full backend with admin panel
- Authoring, drafts, publishing
- Members (free + paid)
- Stripe integration + webhook handling
- Magic link authentication
- Newsletter sending
- Backups (DB + images)
- Internal-only, never publicly exposed

### Ghost-DMZ (untrusted — separate machine)
- Public-facing replica
- Serves posts and assets
- Forwards signup/login/payment requests to Ghost-Core
- Validates JWT tokens to show premium content
- Stateless and disposable
- No secrets, no webhooks, no admin access

---

## Infrastructure

| Property        | Value                          |
|---|---|
| Proxmox Node    | NODE (HP EliteDesk 800 G3)     |
| VM ID           | 101                            |
| VM Name         | web                            |
| OS              | Ubuntu Server 24.04.4 LTS      |
| CPU             | 2 vCPU (host passthrough)      |
| RAM             | 6 GB                           |
| Disk            | 48 GB (local-lvm, VirtIO SCSI) |
| Network         | virtio, bridge {{BRIDGE_NAME}}           |
| Tailscale IP    | {{TS_WEB_IP}}                  |
| Tailscale name  | web                            |
| Hostname        | web                            |

---

## Users

| Username   | sudo | docker | SSH | Notes                     |
|---|---|---|---|---|
| {{SUDO_USER}}   | ✅   | ✅     | ❌  | Primary admin user        |
| {{SSH_USER}} | ❌   | ✅     | ✅  | SSH user, no sudo         |

---

## Networking

- Interface: `{{INTERFACE}}` — DHCP via netplan
- Tailscale: connected, MagicDNS enabled, device name `web`
- No UFW — firewall managed at network/Tailscale ACL level
- Ghost bound to Tailscale IP `{{TS_WEB_IP}}` — not reachable from public internet

### Local DNS routing (split DNS)

`www.{{MY_DOMAIN}}` resolves locally via CoreDNS on the DNS VM:
- CoreDNS `{{MY_DOMAIN}}` zone template returns `100.94.170.14` (DNS VM) for all `*.{{MY_DOMAIN}}`
- AAAA records for `www.{{MY_DOMAIN}}` suppressed (NXDOMAIN) to prevent IPv6 bypass to Cloudflare
- Caddy on DNS VM proxies `www.{{MY_DOMAIN}}` → `http://web:{{PORT}}`
- Clients must use Tailscale MagicDNS (`100.100.100.100`) as DNS to resolve locally

---

## Storage

| Mount Point  | Source                          | Purpose                  |
|---|---|---|
| `{{PATH}}` | `{{TS_NAS_NAME}}:{{PATH}}` | Ghost persistent content |

- NFS mount configured in `/etc/fstab` with `_netdev,nofail`
- Survives reboot ✅
- TrueNAS dataset: `{{PATH}}` (on CORE-NAS)
- Nightly snapshot in place ✅
- Dataset owner: `ghost:ghost` (UID/GID 1000)
- Ghost Docker container runs as UID 1000 — matches dataset ownership

---

## Docker

- Docker version: 29.4.0
- Compose file: `{{PATH}}docker-compose.yml`
- Credentials: `{{PATH}}.env` (not committed)
- Container name: `ghost`
- Restart policy: `unless-stopped`

### Compose environment

| Variable                            | Value / Source                        |
|---|---|
| `NODE_ENV`                          | production                            |
| `url`                               | https://www.{{MY_DOMAIN}}               |
| `admin__url`                        | http://{{TS_WEB_IP}}:{{PORT}}             |
| `database__client`                  | sqlite3                               |
| `database__connection__filename`    | /var/lib/ghost/content/data/ghost.db  |
| `mail__transport`                   | SMTP                                  |
| `mail__options__host`               | {{SMTP}}                       |
| `mail__options__port`               | {{PORT}}                                   |
| `mail__options__secureConnection`   | true                                  |
| `mail__options__auth__user`         | ${MAIL_USER}                          |
| `mail__options__auth__pass`         | ${MAIL_PASSWORD}                      |
| `mail__from`                        | ${MAIL_USER}                          |
| `security__staffDeviceVerification` | false ⚠️ temporary — see tech-debt.md |

### Volume mapping

```
{{PATH}}/content → /var/lib/ghost/content
```

---

## Ghost

- Version: 6.30.0
- Site URL: https://www.{{MY_DOMAIN}}
- Admin URL: http://{{TS_WEB_IP}}:{{PORT}}/ghost
- Database: SQLite, stored on NFS share (CORE-NAS)
- Mail: Migadu SMTP — sending confirmed ✅

---

## Known Issues / Pending

See `issue-ghost-core-followup.md` for full task list. Key items:

- Ghost Admin login session conflict (`url` vs `admin__url`) — not yet resolved (see tech-debt.md)
- `{{GHOST_ADMIN_URL}}` split DNS + Caddy entry not yet configured
- API keys not yet generated
- DMZ sync not yet configured
- Reboot validation ✅
- TrueNAS snapshot schedule ✅ nightly
- Staff device verification disabled — temporary, must be re-enabled
- `www.{{MY_DOMAIN}}` accessible locally via split DNS + Caddy ✅

---

## Related

- `issue-ghost-core-followup.md` — remaining tasks
- `issue-ghost-dmz.md` — DMZ frontend setup (next)
- `architecture.md` — full system architecture
- `tech-debt.md` — known technical debt
- TrueNAS dataset: `{{PATH}}` (CORE-NAS)
