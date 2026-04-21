# Pi-hole Setup

## Overview

Pi-hole runs in the `pihole` LXC (10.2.0.11) in VLAN 1020 (Homelab). It is the primary DNS server for all VLANs, providing ad/tracker blocking, local DNS records, and query logging. All `*.lerkolabs.com` subdomains resolve to 10.2.0.20 (Caddy). Upstream resolver is pfSense Unbound → Cloudflare 1.1.1.1.

## Prerequisites

- LXC created in VLAN 1020 with static IP 10.2.0.11
- Debian 12 template
- pfSense DHCP reservations updated to point VLANs at 10.2.0.11 for DNS

## LXC Spec

| Property | Value |
|----------|-------|
| Hostname | pihole |
| IP | 10.2.0.11/24 |
| Gateway | 10.2.0.1 |
| Cores | 1 |
| RAM | 512MB |
| Template | debian-12-standard |

## Installation

```bash
apt update && apt upgrade -y
curl -sSL https://install.pi-hole.net | bash
```

Installer prompts:
- Upstream DNS: Custom (set to pfSense: 10.2.0.1)
- Blocklists: Default (customize later)
- Admin Web Interface: Yes
- Web Server: lighttpd
- Query Logging: Yes
- Privacy Mode: Show everything (0)

## Configuration

### Local DNS Records

Add all internal domains via **Local DNS → DNS Records**. Every entry points to 10.2.0.20 (Caddy), not the service directly.

Key records to add:

| Domain | IP |
|--------|----|
| pihole.lerkolabs.com | 10.2.0.20 |
| auth.lerkolabs.com | 10.2.0.20 |
| outline.lerkolabs.com | 10.2.0.20 |
| gitea.lerkolabs.com | 10.2.0.20 |
| tasks.lerkolabs.com | 10.2.0.20 |
| finance.lerkolabs.com | 10.2.0.20 |
| grafana.lerkolabs.com | 10.2.0.20 |
| proxmox.lerkolabs.com | 10.2.0.20 |
| vault.lerkolabs.com | 10.2.0.20 |

Add remaining services from [SERVICES.md](../docs/SERVICES.md) following the same pattern.

### Upstream DNS

Settings → DNS → Custom upstream: `10.2.0.1` (pfSense Unbound)

Uncheck all other upstream providers.

### pfSense DHCP Integration

In pfSense: set DNS server for each VLAN's DHCP scope to 10.2.0.11. The WFH VLAN (1050) is the exception — it uses pfSense DNS only (Pi-hole unreachable by design).

## Backup / Restore

Use Teleporter for full config export: Settings → Teleporter → Backup. Store the teleporter zip in Vaultwarden or PBS.

On restore: Settings → Teleporter → Restore. All DNS records, blocklists, and settings are included.

## Verification

```bash
# DNS resolves internal names
nslookup outline.lerkolabs.com 10.2.0.11
# Expected: 10.2.0.20

# Ad blocking active
nslookup doubleclick.net 10.2.0.11
# Expected: 0.0.0.0

# Admin interface
curl -s http://10.2.0.11/admin | grep -i pi-hole
```

## Updates

```bash
pihole -up
```
