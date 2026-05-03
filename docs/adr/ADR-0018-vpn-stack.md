# ADR-0018: VPN Strategy for Download Stack
**Date:** 2026-04-04
**Status:** Accepted
**Deciders:** Owner

---

## Context

A torrent-based media download stack (qBittorrent + *arr apps) requires VPN protection to avoid exposing the host's real IP during download activity. Additionally, a VPN-protected browser is desirable for securely browsing torrent indexers and other sites without exposing the home IP.

Two sub-decisions were required:
1. **VPN gateway approach** — how to route container traffic through the VPN
2. **Browser selection** — which in-browser remote desktop solution to use for VPN-protected browsing

The stack runs on TrueNAS SCALE via a custom Docker Compose app (not native TrueNAS app train), as no suitable VPN gateway app exists in the Community train.

---

## Decision

### VPN Gateway: Gluetun as shared network gateway

Deploy **Gluetun** as a dedicated VPN gateway container. Both qBittorrent and the browser container share Gluetun's network stack via `network_mode: service:gluetun`. All outbound traffic from these containers is routed exclusively through the PIA VPN tunnel.

Provider: **Private Internet Access (PIA)** via OpenVPN.

Kill switch behaviour: if the VPN tunnel drops, Gluetun's built-in firewall (iptables) blocks all outbound traffic from dependent containers — no fallback to the real IP.

### Browser: kasmweb/firefox

Deploy **kasmweb/firefox** as a VPN-routed in-browser remote desktop, also sharing Gluetun's network stack. Accessible via Caddy reverse proxy at `browser.mydomain.com`.

Tag strategy: `1.18.0-rolling-daily` — nightly rolling builds within the current Kasm release line. When Kasm publishes a new major version, the tag must be manually bumped (e.g. `1.19.0-rolling-daily`).

---

## Alternatives Considered

### VPN Gateway

| Option | Description | Decision |
|--------|-------------|----------|
| **Gluetun (OpenVPN)** | Dedicated gateway container; all dependent containers share its network stack | ✅ Selected |
| **qBittorrent built-in SOCKS5 proxy** | PIA SOCKS5 configured directly in qBittorrent settings | ❌ Rejected — no kill switch; proxy drop silently exposes real IP; DNS leak risk; does not cover browser |
| **Gluetun (WireGuard)** | Same as selected option but using WireGuard protocol instead of OpenVPN | ⏭ Deferred — lower CPU overhead and faster handshake; requires generating WireGuard credentials from PIA portal; noted as future improvement |

### Browser

| Option | Description | Decision |
|--------|-------------|----------|
| **kasmweb/firefox** | Standalone Kasm Firefox image; good mobile touch support; no full Kasm install required | ✅ Selected |
| **linuxserver/firefox** | Lightweight containerised Firefox via KasmVNC | ❌ Rejected — functional but basic UI; poor mobile/touch experience |
| **Kasm Workspaces (full)** | Full remote desktop platform with session management and multiple browser options | ❌ Rejected — heavyweight; overkill for single-user VPN browser use case |
| **Neko** | Self-hosted virtual browser with multi-user screen sharing | ❌ Rejected — designed for shared/collaborative sessions; unnecessary complexity for solo use |

### Deployment Location

| Option | Description | Decision |
|--------|-------------|----------|
| **TrueNAS SCALE (NAS)** | Custom Compose app deployed on the existing NAS | ✅ Selected — consolidates infrastructure; avoids extra hardware; download storage is co-located |
| **Dedicated VM** | Separate VM running the download stack in isolation | ❌ Rejected — additional operational overhead; no meaningful security benefit for home use case |
| **Existing other VM** | Piggyback on another running VM | ❌ Rejected — mixing download workload with other services; storage I/O overhead |

---

## Stack Layout

```
Gluetun (PIA VPN, OpenVPN)
├── qBittorrent    → host port 42080 (WebUI)
└── kasmweb/firefox → host port 42901 (HTTPS)
```

Storage paths:
- Gluetun config: `/mnt/Storage/Jellyfin/VPN/gluetun`
- qBittorrent config: `/mnt/Storage/Jellyfin/VPN/qbittorrent/config`
- Downloads: `/mnt/Storage/Soft/Videos`
- Firefox profile: `/mnt/Storage/Jellyfin/VPN/firefox/config`

---

## Future Improvements

- Migrate from OpenVPN to **WireGuard** via Gluetun for lower latency and reduced CPU overhead.

---

## Consequences

- qBittorrent and browser traffic are always VPN-protected; real IP is never exposed during download or indexer browsing activity.
- Hard kill switch: VPN drop = no internet for dependent containers.
- Single PIA subscription covers both use cases.
- Compose stack is not managed via native TrueNAS app train — updates require manual image tag bumps for kasmweb/firefox.
- Browser is accessible on both desktop and mobile via `browser.mydomain.com` with reasonable touch support.
