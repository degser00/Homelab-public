# CoreDNS (Internal DNS) – Ingress Node Deployment

This directory contains the CoreDNS configuration used for internal split-DNS resolution inside the private network. CoreDNS runs on the ingress node and provides DNS answers for all internal service hostnames under `mydomain.com`.

## Overview

CoreDNS serves as the internal DNS authority for the cluster.

It resolves internal service hostnames to the **reverse proxy ingress node** (`<reverse-proxy-ip>`), where Caddy terminates TLS and routes traffic.  
All other DNS queries are forwarded upstream.

### Key Responsibilities
- Authoritative DNS for `*.mydomain.com`
- Split DNS for Tailscale clients
- Forwarding for external domains
- Simple, file-based configuration

## Folder Structure

```
coredns/
 ├─ Corefile        # Main CoreDNS configuration
 └─ README.md       # This document
```

## Corefile Explanation

Example:

```
. {
    forward . 1.1.1.1 9.9.9.9
    log
    errors
}

mydomain.com {
    hosts {
        <reverse-proxy-ip> service1.mydomain.com
        <reverse-proxy-ip> service2.mydomain.com
        <reverse-proxy-ip> n8n.mydomain.com
        <reverse-proxy-ip> llm.mydomain.com
        <reverse-proxy-ip> obs.mydomain.com
        fallthrough
    }
    log
    errors
}
```

## How This Works

1. A Tailscale client queries `service.mydomain.com`.
2. CoreDNS returns the ingress node IP (`<reverse-proxy-ip>`).
3. The client connects to the ingress node.
4. **Caddy** terminates TLS using its internal CA.
5. Caddy proxies to the correct backend service:

```
<reverse-proxy-ip> → <service-host-ip>:<service-port>
```

## Running CoreDNS

### Start
```
sudo systemctl start coredns
```

### Stop
```
sudo systemctl stop coredns
```

### Restart
```
sudo systemctl restart coredns
```

### Status
```
systemctl status coredns
```

## Logs

```
journalctl -u coredns -f
```

## Updating Configuration

1. Edit `Corefile`
2. Optional syntax validation:
```
coredns -conf Corefile
```
3. Reload:
```
sudo systemctl restart coredns
```

## Notes

- All internal hostnames resolve to `<reverse-proxy-ip>`.
- Internal-only DNS (no public queries).
- Minimal configuration for reliability and auditability.

## Future Improvements
- Containerize CoreDNS.
- Add Prometheus metrics.
- Integrate logs with observability stack.
