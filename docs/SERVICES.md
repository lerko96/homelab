# Services

Full registry of what's running, where it lives, and how to reach it. See [README](../README.md) for compute layout and [Network](NETWORK.md) for VLAN/IP context.

## Status Key

| Symbol | Meaning |
|--------|---------|
| ✅ | Running, healthy |
| ⚠️ | Running, needs attention |
| 🔴 | Down / broken |
| 🚧 | In progress |
| ➖ | Decommissioned |

## Core Network (VLAN 1000/1010/1020)

Admin consoles at <service>.lerkolabs.com, VPN-gated.

| Service | IP | Port | VLAN | Status | Notes |
|---------|----|------|------|--------|-------|
| pfSense | 10.1.0.1 / 10.0.0.1 | 443 | LAN/MGMT | ✅ | Firewall, DHCP, WireGuard VPN |
| Omada Switch | 10.0.0.2 | 443 | MGMT | ✅ | Managed switch, VLAN config |
| AT&T Gateway | 192.168.1.254 | 80 | — | ✅ | IP Passthrough only, WiFi disabled |
| Pi-hole | 10.2.0.11 | 80/53 | 1020 | ✅ | Primary DNS, ad blocking |
| Caddy (infra) | 10.2.0.20 | 80/443 | 1020 | ✅ | Reverse proxy, wildcard SSL via Cloudflare DNS-01 |
| ntfy | 10.2.0.20 | — | 1020 | ✅ | Push notifications (infra LXC) |
| Authentik | 10.2.0.25 | 9000 | 1020 | ✅ | SSO — OIDC + forward auth |
| Proxmox | 10.2.0.10 | 8006 | 1020 | ✅ | Hypervisor |

## Observability (monitor LXC — 10.2.0.51)

Observability at <service>.lerkolabs.com, VPN-gated.

| Service | Notes |
|---------|-------|
| Grafana | Dashboards, alerting |
| Victoria Metrics | Metrics storage |
| Beszel | Container + host monitoring |

## Productivity Apps (apps LXC — 10.2.0.60)

All apps served at <service>.lerkolabs.com behind Authentik.

| Service | Auth | Purpose |
|---------|------|---------|
| Outline | OIDC | Team wiki |
| Vikunja | OIDC | Task management |
| Ghostfolio | Forward auth | Portfolio tracking |
| Hoarder | Forward auth | Bookmark manager |
| Grist | Forward auth | Spreadsheets / data |
| Actual Budget | Forward auth | Personal budgeting |
| FreshRSS | Forward auth | RSS reader |
| Memos | Forward auth | Quick notes |
| Traggo | Forward auth | Time tracking |
| Baikal | Forward auth | CalDAV / CardDAV |
| Glance | Forward auth | Homepage dashboard |
| Filebrowser | Forward auth | File management |
| Bytestash | Forward auth | Snippet storage |

Shared infrastructure in apps LXC: single Postgres instance (multi-DB) + Redis. See [D004](DECISIONS.md#d004--shared-postgres--redis-in-apps-lxc).

## Secrets (vault LXC — 10.2.0.21)

Served at <service>.lerkolabs.com, VPN-gated.

| Service | Notes |
|---------|-------|
| Vaultwarden | Isolated LXC — not shared with apps |

## Media (servarr VM)

| Service | Purpose |
|---------|---------|
| Plex + Jellyfin | Media streaming |
| Sonarr / Radarr / Lidarr | Automated media management |
| Prowlarr + Bazarr | Indexer aggregation + subtitles |
| qBittorrent (via Gluetun) | Downloads — VPN-gated |
| Calibre-Web Automated | Book library with auto-ingest |
| Kavita | E-reader |

## DMZ (VLAN 1099 — 10.99.0.0/24)

| Service | URL | Status | Notes |
|---------|-----|--------|-------|
| Caddy (DMZ) | — | ✅ | Public reverse proxy |
| Gitea | https://gitea.lerkolabs.com | ✅ | Public Git |
| Portfolio | https://lerkolabs.com | ✅ | Personal site |

## Access Matrix

| Service | LAN | Homelab | Guest | IoT | WFH | VPN |
|---------|-----|---------|-------|-----|-----|-----|
| pfSense Web GUI | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ |
| Pi-hole Admin | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ |
| All *.lerkolabs.com | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ |
| Proxmox | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ |
| Internet | ✅ | limited | ✅ | ✅ | ✅ | optional |
