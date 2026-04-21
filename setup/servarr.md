# Servarr (Media VM) Setup

## Overview

The `servarr` VM runs the complete media stack: Plex and Jellyfin for streaming, the *arr suite for automated media management, and qBittorrent routed through Gluetun VPN for downloads. All services run via Docker Compose. The VM lives on Proxmox in VLAN 1020 (Homelab).

## VM Spec

| Property | Value |
|----------|-------|
| Hostname | servarr |
| VLAN | 1020 (Homelab) |
| Cores | 4 |
| RAM | 8GB |
| OS | Debian 12 |
| Nesting | ✓ |

## Services

| Service | Purpose |
|---------|---------|
| Plex | Media streaming (hardware transcoding) |
| Jellyfin | Open-source media streaming alternative |
| Sonarr | TV show management |
| Radarr | Movie management |
| Lidarr | Music management |
| Prowlarr | Indexer aggregation (feeds Sonarr/Radarr/Lidarr) |
| Bazarr | Subtitle management |
| qBittorrent | Downloads — routed through Gluetun VPN container |
| Calibre-Web Automated | Book library with auto-ingest |
| Kavita | E-reader / comic reader |

## Prerequisites

- VM created in VLAN 1020
- Gluetun-compatible VPN credentials (stored in Vaultwarden)
- Media storage mounted (NFS or local disk)
- Caddy routing configured for any public-facing services

## Installation

```bash
apt update && apt upgrade -y
apt install -y curl nano
timedatectl set-timezone <your/timezone>
curl -fsSL https://get.docker.com | sh
systemctl enable docker
```

## Directory Structure

```
/opt/docker/servarr/
├── docker-compose.yml
├── .env
└── config/
    ├── plex/
    ├── jellyfin/
    ├── sonarr/
    ├── radarr/
    ├── lidarr/
    ├── prowlarr/
    ├── bazarr/
    ├── qbittorrent/
    ├── calibre/
    └── kavita/

/media/
├── tv/
├── movies/
├── music/
├── books/
└── downloads/
    ├── complete/
    └── incomplete/
```

## qBittorrent + Gluetun (VPN-gated downloads)

qBittorrent runs inside the Gluetun network namespace. All download traffic exits through the VPN — no VPN = no download traffic.

```yaml
# docker-compose.yml excerpt
services:
  gluetun:
    image: qmcgaw/gluetun:latest
    container_name: gluetun
    cap_add:
      - NET_ADMIN
    environment:
      - VPN_SERVICE_PROVIDER=<provider>
      - VPN_TYPE=wireguard
      - WIREGUARD_PRIVATE_KEY=${WIREGUARD_PRIVATE_KEY}
      - WIREGUARD_ADDRESSES=${WIREGUARD_ADDRESSES}
      - SERVER_COUNTRIES=${SERVER_COUNTRIES}
    ports:
      - "8080:8080"   # qBittorrent WebUI via Gluetun

  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    network_mode: "service:gluetun"   # all traffic through VPN
    environment:
      - WEBUI_PORT=8080
    volumes:
      - ./config/qbittorrent:/config
      - /media/downloads:/downloads
```

## *arr Suite Configuration

Prowlarr is the central indexer — configure it first, then connect Sonarr/Radarr/Lidarr to it. All *arr services connect to qBittorrent as the download client (pointing to Gluetun's exposed port).

## Caddy Configuration

Add to Caddyfile on infra LXC for any *arr services you want accessible via HTTPS. Replace `<servarr-ip>` with the VM's IP.

```caddyfile
# Example — Plex handles its own auth, no forward auth needed
plex.lerkolabs.com {
    reverse_proxy <servarr-ip>:32400
}

# *arr services — protect with Authentik forward auth
sonarr.lerkolabs.com {
    import authentik_forward_auth
    reverse_proxy <servarr-ip>:8989
}
```

## Verification

```bash
# All containers running
docker ps

# Gluetun VPN tunnel active
docker exec gluetun wget -qO- https://api.ipinfo.io/ip
# Should return VPN provider IP, not home WAN IP

# qBittorrent accessible
curl -I http://localhost:8080
```
