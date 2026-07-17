# Open WebUI Module

## Overview
This module provides a **standalone deployment of Open WebUI**, a feature-rich web interface for interacting with Ollama and other LLM providers.  
It is **self-contained and replaceable**, in line with the modular architecture ADR.

---

## Dependencies
- **Docker Engine** (compose v2 or later).  
- **Ollama service** running on the same Docker network (ai-net).  
- **Local persistent storage** for user data, chats, and settings.  
- **Cloudflare Tunnel** for secure external access (optional).  
- **Internal Docker network** (`ai-net`) for connection to Ollama.

---

## Folder Structure
```
/modules/open-webui/
│
├── docker-compose.yml    # Deployment definition for Open WebUI
├── .env                  # Environment variables (not committed)
├── .env.example          # Template for environment variables
├── README.md             # This file
└── data/                 # Persistent storage (auto-created by volume)
```

---

## Deployment

1. Copy `.env.example` → `.env` and configure:
   - `OLLAMA_BASE_URL` (default: `http://ollama:{{OLLAMA_AMD_PORT}}`)
   - `WEBUI_PORT` (default: `3000`)
   - `WEBUI_SECRET_KEY` (generate with: `openssl rand -hex 32`)

2. Start the service:
   ```bash
   docker compose up -d
   ```

3. Access locally via:
   ```
   http://localhost:3000
   ```

4. Access remotely via Cloudflare tunnel (if configured):
   ```
   https://lll.mydomain.com
   ```

> The `docker-compose.yml` defines a single `open-webui` container with persistent storage and connection to the `ai-net` network.

---

## Networking & Access

### Local Access
- Default port: `3000` (configurable via `WEBUI_PORT`)  
- Direct access: `http://localhost:3000`

### Remote Access
- Secured via **Cloudflare Access** tunnel
- Domain: `lll.mydomain.com` (or your configured domain)
- Cloudflare acts as authentication gateway
- Users login with Cloudflare credentials, then with Open WebUI accounts

### Internal Connection
- Connects to Ollama via: `http://ollama:{{OLLAMA_AMD_PORT}}`
- Both services must be on the same Docker network (`ai-net`)

---

## Environment Variables

Defined in `.env`:

| Variable | Description | Example |
|----------|-------------|---------|
| `OLLAMA_BASE_URL` | URL to Ollama service | `http://ollama:{{OLLAMA_AMD_PORT}}` |
| `WEBUI_PORT` | Host port for local access | `3000` |
| `WEBUI_SECRET_KEY` | Session encryption key | Generate with `openssl rand -hex 32` |
| `ENABLE_SIGNUP` | Allow public signups | `false` (admin-only user creation) |
| `DEFAULT_USER_ROLE` | Role for new users | `user` |

> Additional variables can be added for OAuth, model access control, or feature flags.

---

## Authentication & User Management

### User Roles
- **Admin**: Full access to all models, settings, and user management
- **User**: Limited access based on permissions (configure in Admin Panel)

### Creating Users
1. Login as admin
2. Navigate to **Admin Panel** → **Users**
3. Click **Add User** and create accounts
4. Assign appropriate roles and model permissions

### Model Access Control
By default, only admin accounts can see all models. To grant model access to regular users:
1. **Admin Panel** → **Settings** → **Models** (or similar)
2. Configure model visibility/permissions
3. OR edit individual users and grant model access

> Note: User role determines model visibility. If users can't see models, check their role and permissions.

---

## Persistence

- **User data, chats, and settings** stored in the `open-webui-data` Docker volume
- Maps to `/app/backend/data` inside the container
- When migrating or upgrading, preserve this volume to retain all user data

### Backup
```bash
# Backup volume
docker run --rm -v open-webui-data:/data -v $(pwd):/backup alpine tar czf /backup/open-webui-backup.tar.gz /data

# Restore volume
docker run --rm -v open-webui-data:/data -v $(pwd):/backup alpine tar xzf /backup/open-webui-backup.tar.gz -C /
```

---

## Version Pinning

Currently using `ghcr.io/open-webui/open-webui:main`.  

Future policy (recommended):
- Pin a tested version tag (e.g. `ghcr.io/open-webui/open-webui:v0.1.123`)
- Update manually only after validation in staging
- Check release notes at: https://github.com/open-webui/open-webui/releases

---

## Cloudflare Tunnel Configuration

### Tunnel Setup
Your Cloudflare tunnel should point to:
```yaml
ingress:
  - hostname: lll.mydomain.com
    service: http://localhost:3000
  - service: http_status:404
```

### Cloudflare Access Application
- **Application domain**: `lll.mydomain.com`
- **Private hostname**: `open-webui:{{OLLAMA_B50_PORT}}`
- **Authentication**: Configure email/domain allow list in Access policies
- Open WebUI handles its own authentication after Cloudflare gateway

---

## Troubleshooting

### Users can't see models
- Check user role (Admin vs User)
- Verify model permissions in Admin Panel → Settings
- Ensure user has been granted access to specific models
- Admin role has full access; User role requires explicit permissions

### Port already in use
- Change `WEBUI_PORT` in `.env` to an available port
- Restart: `docker compose down && docker compose up -d`

### Can't connect to Ollama
- Verify both containers are on `ai-net` network: `docker network inspect ai-net`
- Check `OLLAMA_BASE_URL` is set to `http://ollama:{{OLLAMA_AMD_PORT}}`
- Verify Ollama is running: `docker ps | grep ollama`

### Cloudflare tunnel not working
- Verify tunnel is running and configured for the correct port
- Check tunnel status: `cloudflared tunnel info`
- Ensure domain matches in both tunnel config and Access application

---

## Migration & Portability

To migrate this module, copy:
- `/modules/open-webui/docker-compose.yml`
- `/modules/open-webui/.env`
- Backup the `open-webui-data` volume (see Persistence section)

Re-deploy on the new host:
```bash
# Restore volume backup first
docker volume create open-webui-data
docker run --rm -v open-webui-data:/data -v $(pwd):/backup alpine tar xzf /backup/open-webui-backup.tar.gz -C /

# Start service
docker compose up -d
```

No external dependencies beyond this folder and the `ai-net` network.

---

## Security Notes

- **Secret Key**: Never commit `WEBUI_SECRET_KEY` to version control
- **User Signups**: Keep `ENABLE_SIGNUP=false` in production to prevent unauthorized access
- **Cloudflare Access**: Acts as the first layer of authentication for external access
- **HTTPS**: All external traffic secured via Cloudflare tunnel (automatic TLS)
- **Local Access**: No authentication at network level; Open WebUI login required

---

## Useful Commands

```bash
# View logs
docker logs -f open-webui

# Restart service
docker compose restart

# Stop service
docker compose down

# Rebuild and restart
docker compose up -d --force-recreate

# Check connected networks
docker network inspect ai-net
```

---

## Additional Resources

- Open WebUI Documentation: https://docs.openwebui.com
- GitHub Repository: https://github.com/open-webui/open-webui
- Cloudflare Tunnel Docs: https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/
