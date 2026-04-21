# Inventory

Hardware inventory — make/model, role, specs. See [README](../README.md) for how everything fits together.

## Active Hardware

| Device | Role | Model | Notes |
|--------|------|-------|-------|
| Proxmox host | Hypervisor | Dual-socket Xeon rackmount | 32c / 128GB RAM |
| Proxmox backup server | Backup server | HP desktop | Backup all LXCs + VMs |
| pfSense router | Firewall / VPN / DHCP / routing | Netgate 1100 | ~6W idle, handles 2–3Gbps routing + 600Mbps WireGuard |
| Managed switch | VLAN switching | TP-Link Omada L2+ managed | All port VLANs managed via Omada Controller |
| Access points | WiFi | TP-Omada WiFi 6 | Auto-adopted by Omada Controller |
| AT&T Gateway | ISP ONT | AT&T-issued | IP Passthrough |

## pfSense Box Detail

| Property | Value |
|----------|-------|
| CPU | Intel N100 (4-core, 3.4GHz) |
| Idle power | ~6W |
| Routing throughput | 2–3Gbps |
| WireGuard throughput | ~600Mbps |
| pfSense version | pinned |

## Proxmox Host Detail

| Property | Value |
|----------|-------|
| CPU | Intel Xeon E5-2683 v4 (32-core, 2.10GHz) |
| RAM | 128 GB |
| Boot drive | 128 GB |
| Storage | 1500 GB |
| Proxmox version | pinned |

## Proxmox Backup Detail |

| Property | Value |
|----------|-------|
| CPU | Intel i5-4590T (4-core, 2.00GHz) |
| RAM | 16 GB |
| Boot drive | 256 GB |
| Storage | 2 TB |
| PBS version | pinned |

## Licensing / Subscriptions

| Service | Type | Notes |
|---------|------|-------|
| Cloudflare | Free | lerkolabs.com DNS + DNS-01 challenge |
| Let's Encrypt | Free | Via Caddy — auto-renewal |
| AT&T Fiber | Monthly | 1Gbps symmetric |
| Domain Name | Annually | `lerkolabs.com` |

## Backup (PBS)

Nightly backups + weekly GC for all LXCs + VMs, managed by Proxmox Backup Server.

**Covered:** pihole, auth, infra, monitor, apps, vault, servarr VM, haos VM

### Retention

| Keep | Amount |
|------|--------|
| Last | 3 |
| Daily | 13 |
| Weekly | 8 |
| Monthly | 11 |
| Yearly | 2 |
