# homelab

Self-hosted services on a single Proxmox host. Segmented network, runs 24/7.

## Why I built this

I started this while studying for CompTIA and the plan was a small router, a few VLANs, and maybe two or three services and then got carried away.

## What's running

| Layer | Tool |
|---|---|
| Hypervisor | Proxmox VE |
| Firewall | pfSense (low-power x86) |
| Switching | TP-Link Omada (managed VLANs) |
| Reverse proxy | Caddy (Cloudflare DNS-01) |
| Identity | Authentik (OIDC + forward auth) |
| DNS | Pi-hole → Unbound → Cloudflare |
| Remote access | WireGuard |
| Monitoring | Victoria Metrics + Grafana + Beszel |
| Backups | Proxmox Backup Server |

## Scope

Around 10 LXCs and a couple of VMs running about 20 services across 7 VLANs.

## Design choices

- VLANs are organized by trust tier. Management is its own tier because a compromise there would be no bueno
- Internal services sit behind Authentik. OIDC where the app supports it and then Caddy forward auth where it doesn't
- Public surface is small. A handful of services, behind a DMZ-isolated reverse proxy with firewall rules backing up the proxy config
- Admin surfaces are only available from Management tier and VPN.

## Documented here

| Doc | About |
|---|---|
| [Services](docs/SERVICES.md) | What's deployed, grouped by what it does |
| [Network](docs/NETWORK.md) | Segmentation, firewall posture, DNS |
| [Security](docs/SECURITY.md) | Layered controls, threat model, limitations |

The IP plan, hardware inventory, ADRs, rebuild runbook, and retention policies are in a private repo.
