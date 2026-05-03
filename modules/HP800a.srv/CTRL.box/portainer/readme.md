# Portainer Control Plane

## Overview

The **ctrl** VM runs a Portainer instance that acts as a centralised control plane for managing all Docker environments across the infrastructure. Rather than SSH-ing into individual hosts to inspect or modify containers, everything is visible and manageable from a single Portainer UI.

---

## What It Does

- **Centralised visibility** — see all containers, stacks, images, volumes, and networks across every connected Docker host in one place
- **Stack management** — deploy, edit, and redeploy Docker Compose stacks without touching the command line
- **Real-time monitoring** — container status, resource usage (CPU, memory), and logs accessible from the browser
- **Multi-environment support** — connects to remote Docker hosts via Portainer Agent or Docker socket, grouping them under a single dashboard

---

## Architecture

```
┌─────────────────────────────────┐
│          ctrl VM                │
│  ┌───────────────────────────┐  │
│  │  Portainer (Control Plane)│  │
│  │  Port: {{SRV_IMMICH_PORT}} (HTTPS)       │  │
│  └───────────┬───────────────┘  │
└─────────────-│──────────────────┘
               │
       Portainer Agent
               │
    ┌──────────┼──────────┐
    │          │          │
  Host A     Host B     Host C
 (Docker)  (Docker)   (Docker)
```

---

## Connecting a New Docker Host

1. Install the Portainer Agent on the target host:
   ```bash
   docker run -d \
     -p 9001:9001 \
     --name portainer_agent \
     --restart=always \
     -v /var/run/docker.sock:/var/run/docker.sock \
     -v /var/lib/docker/volumes:/var/lib/docker/volumes \
     portainer/agent:latest
   ```

2. In the Portainer UI on **ctrl**: go to **Environments** → **Add environment** → **Docker Standalone** → enter the agent IP and port `9001`

---

## Managing Stacks (Compose Files)

### Viewing / Editing via UI
1. Go to the target environment
2. Navigate to **Stacks**
3. Click the stack name → **Editor** tab
4. Make changes → **Update the stack**

### Editing on Disk (file-based stacks)
Compose files are stored at:
```
/appliance/<stack-name>/compose.yml
```

SSH into the relevant host, edit the file, then redeploy:
```bash
nano /appliance/<stack-name>/compose.yml
docker compose -f /appliance/<stack-name>/compose.yml up -d
```

---

## Accessing Portainer

| URL | Notes |
|-----|-------|
| `https://ctrl:{{SRV_IMMICH_PORT}}` | Main UI (HTTPS) |

---

## Notes

- Portainer data (stack definitions, users, settings) is persisted in the `portainer_data` Docker volume on the ctrl VM — back this up if the VM is recreated
- Agent communication runs on port `9001` — ensure firewall rules allow traffic from ctrl to each host on this port
- Stacks deployed via the Portainer UI are tracked internally; stacks deployed manually with `docker compose` need to be imported or re-created in Portainer to appear under **Stacks**
