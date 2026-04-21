# Security

Security posture — what's exposed, how auth works, update cadence, known debt. See [Network](NETWORK.md) for VLAN isolation details.

## Internet-Exposed Ports

| Port | Protocol | Destination | Purpose |
|------|----------|-------------|---------|
| 51820 | UDP | pfSense WAN | WireGuard VPN |

No management ports (22, 8006, 443) exposed to the internet. All admin access requires an active WireGuard connection first. Cloudflare DNS-01 challenge handles TLS — no port 80/443 needed on WAN.

## Authentication Layers

| Layer | Mechanism | Coverage |
|-------|-----------|----------|
| All web services | Authentik SSO (OIDC or forward auth) | 100% of `*.lerkolabs.com` |
| VPN | WireGuard pre-shared keys | Required for all remote access |
| pfSense | Web GUI + SSH key | VPN-only access |
| Proxmox | Web GUI + SSH key | VPN-only access |
| Secrets | Vaultwarden (isolated LXC) | All credentials |

No service is accessible anonymously. Guests and IoT have zero access to any internal service.

## Secrets Policy

- No plaintext secrets in any config file committed to the repo
- All secrets referenced by Vaultwarden entry name (e.g., `homelab/pfsense`)
- `.env` files in `.gitignore`
- Vaultwarden lives in its own isolated LXC — no shared container

## Certificate Management

| Domain | Provider | Method | Renewal |
|--------|----------|--------|---------|
| `*.lerkolabs.com` | Let's Encrypt via Cloudflare | DNS-01 challenge | Automatic (Caddy) |

Caddy handles all cert issuance and renewal automatically. No manual action unless Cloudflare API token expires.

## Update Cadence

| System | Frequency | Method |
|--------|-----------|--------|
| pfSense | Monthly | Manual — System → Update |
| Proxmox | Monthly | `apt update && apt dist-upgrade` |
| Pi-hole | Monthly | `pihole -up` |
| Docker services | Weekly | `docker compose pull && docker compose up -d` |
| Omada firmware | Quarterly | Omada Controller → Devices |
| AT&T Gateway | Automatic | AT&T pushes updates |
| WireGuard keys | Annually (or on peer change) | Rotate in pfSense VPN config |

## Known Technical Debt

Known gaps tracked privately.
