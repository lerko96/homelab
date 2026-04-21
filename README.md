# homelab

Personal homelab running 24/7 on production-grade hardware. Domain: `lerkolabs.com`. Single Proxmox host running 9 LXC containers + 2 VMs across 8 isolated VLANs with 20+ self-hosted services.

## At a Glance

| Component | Technology |
|-----------|-----------|
| Hypervisor | Proxmox VE |
| Firewall | pfSense (Intel N100) |
| Switching | TP-Link Omada (managed VLANs) |
| Reverse Proxy | Caddy + Cloudflare DNS-01 |
| Auth | Authentik SSO (OIDC + forward auth) |
| DNS | Pi-hole → pfSense Unbound → Cloudflare |
| VPN | WireGuard, UDP 51820 |
| Monitoring | Victoria Metrics + Grafana + Beszel |
| Backups | Proxmox Backup Server (PBS) |

## Compute Layout

| Container | IP | Cores | RAM | What Runs |
|-----------|-----|-------|-----|-----------|
| `pihole` | 10.2.0.11 | 1 | 512MB | Pi-hole DNS + ad blocking |
| `auth` | 10.2.0.25 | 1 | 512MB | Authentik SSO |
| `infra` | 10.2.0.20 | 2 | 1GB | Caddy reverse proxy, ntfy |
| `monitor` | 10.2.0.51 | 4 | 4GB | Victoria Metrics, Grafana, Beszel |
| `apps` | 10.2.0.60 | 4 | 6GB | 15+ productivity apps (Docker Compose) |
| `vault` | 10.2.0.X | 1 | 256MB | Vaultwarden (isolated) |
| `servarr` (VM) | — | 4 | 8GB | Plex, Jellyfin, *arr stack, qBittorrent |
| `haos` (VM) | — | 2 | 4GB | Home Assistant OS |

## DMZ (Public-Facing)

| Container | IP | Service |
|-----------|-----|---------|
| `caddy-dmz` | 10.99.0.20 | Public reverse proxy |
| `gitea` | 10.99.0.22 | gitea.lerkolabs.com |
| `portfolio` | 10.99.0.23 | lerkolabs.com |

## Key Principles

- All services require Authentik authentication — no anonymous access
- No management ports exposed to internet — all admin access via WireGuard first
- Caddy handles TLS termination; internal services run plain HTTP
- Secrets never committed — all referenced by Vaultwarden entry name

## Navigation

- [Services](docs/SERVICES.md) — full service registry with URLs and access matrix
- [Network](docs/NETWORK.md) — VLANs, firewall policy, DNS architecture, physical topology
- [Decisions](docs/DECISIONS.md) — architecture decision records (D001–D010)
- [Security](docs/SECURITY.md) — security posture, auth layers, update cadence, known debt
- [Inventory](docs/INVENTORY.md) — hardware inventory
- [Rebuild](REBUILD.md) — disaster recovery sequence (8 phases)
- [Setup guides](setup/) — per-service installation and configuration
