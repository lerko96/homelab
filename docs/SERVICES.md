# Services

## Identity & access

| Service | What it does |
|---|---|
| Authentik | SSO for internal services, OIDC where supported + caddy forward auth otherwis |
| Pi-hole | LAN DNS, ad blocking + source of truth for internal hostnames |
| WireGuard | remote access |

## Reverse proxy & TLS

Two Caddy instances:

- **Internal Caddy** fronts everything internal. LAN or VPN only.
- **DMZ Caddy** fronts the public services. Lives on its own VLAN with a firewall-enforced allowlist into internal.

Both use Cloudflare DNS-01 for ACME, which lets internal-only services get valid public certs without being exposed for issuance.

## Productivity & knowledge

| Service | What it replaces |
|---|---|
| Outline | notion |
| Vikunja | todoist / asana |
| Hoarder | pocket / raindrop |
| Memos | apple nnotes |
| FreshRSS | feedly |
| Bytestash | gist / pastebin |
| Filebrowser | dropbox |
| Baikal | iCloud calendar/contacts (CalDAV / CardDAV) |

## Money

| Service | What it replaces |
|---|---|
| Actual Budget | YNAB |
| Ghostfolio | personal capital |

## Operations

| Service | What it does |
|---|---|
| Grist | lightweight excel type |
| Glance | personal homepage |
| Traggo | time tracking |

## Media

| Service | What it does |
|---|---|
| Plex | mdia library (legacy clients) |
| Jellyfin | media library (primary) |
| *arr stack | library automation |
| qBittorrent | Downloads |
| Immich | photo backup and viewing |

## Home / IoT

| Service | What it does |
|---|---|
| Home Assistant OS | home automation hub |

## Secrets

| Service | What it does |
|---|---|
| Vaultwarden | bitwarden-compatible password manager *Planned, not deployed yet |

## Bots & automation

| Service | What it does |
|---|---|
| Vocard | discord music bot |
| MonitorRSS | rss-to-discord feed |
| ntfy | push notifications for ops alerts |

## Monitoring

| Service | What it does |
|---|---|
| Victoria Metrics | time-series store |
| Grafana | dashboards |
| Beszel | host metrics |
| Uptime Kuma | uptime checks |

## Public services

A small set behind the DMZ reverse proxy on a VLAN with no inbound to internal.

| Service | Why it's public |
|---|---|
| Portfolio | it's a portfolio |
| Self-hosted Git | so you can read this |
| SSO endpoint | required for the OIDC flow on the Discord bot dashboard. the firewall is enabled so that the public proxy can only reach this one internal backend |
| Discord bot dashboard | so my friends can use pick tunes. authentik forward auth gates it |

