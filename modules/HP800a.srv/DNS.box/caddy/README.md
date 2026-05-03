# Caddy (Internal Reverse Proxy) – Ingress Node Deployment

This directory contains the Caddy configuration used for internal HTTPS termination and hostname-based routing inside the private network.
Caddy runs on the ingress node and serves as the single entry point for all internal services under `mydomain.com`.

## Overview

Caddy provides:

- Automatic internal TLS using its built-in CA
- Reverse proxy routing based on hostname
- Centralized ingress for all internal services
- A stable, trusted HTTPS layer for Tailscale-connected clients

Caddy listens on the ingress node (`<reverse-proxy-ip>`) and forwards requests to internal workloads running on service hosts (`<service-host-ip>`).

## Folder Structure

```
caddy/
 ├─ Caddyfile
 └─ README.md
```

## Caddyfile Example

```
{
    admin off
}

n8n.mydomain.com {
    tls internal
    reverse_proxy <service-host-ip>:<service-port>
}

llm.mydomain.com {
    tls internal
    reverse_proxy <service-host-ip>:<service-port>
}

obs.mydomain.com {
    tls internal
    reverse_proxy <service-host-ip>:<service-port>
}
```

## How It Works

1. Client resolves `service.mydomain.com` → `<reverse-proxy-ip>` via CoreDNS.
2. Client connects to Caddy using HTTPS.
3. Caddy uses its internal CA to present a trusted certificate.
4. Caddy proxies the request to the appropriate backend service:
   `<service-host-ip>:<service-port>`

## Internal CA

Caddy's internal CA root certificate is stored at:

```
/var/lib/caddy/.local/share/caddy/pki/authorities/local/root.crt
```

Clients must install this root certificate into their system trust store to avoid HTTPS warnings.

## Managing Caddy

### Validate config
```
sudo caddy validate --config /etc/caddy/Caddyfile
```

### Restart
```
sudo systemctl restart caddy
```

### Logs
```
journalctl -u caddy -f
```

### Status
```
systemctl status caddy
```

## Updating Config

1. Edit `Caddyfile`
2. Validate with `caddy validate`
3. Restart Caddy

## Notes

- Internal-only deployment (not exposed publicly).
- TLS is terminated at the ingress node.
- Backends communicate over HTTP.
- Works together with CoreDNS for split-DNS.

## Future Improvements

- Containerize Caddy
- Add Prometheus metrics
- Forward logs into observability stack
