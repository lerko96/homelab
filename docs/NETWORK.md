# Network

VLAN map, firewall policy, DNS architecture, and physical topology. See [README](../README.md) for the big picture and [Services](SERVICES.md) for what lives where.

## VLAN Map

| VLAN ID | Name | Subnet | Gateway | DHCP Range | DNS |
|---------|------|--------|---------|------------|-----|
| 1000 | MGMT | 10.0.0.0/24 | 10.0.0.1 | 10.0.0.100–150 | pfSense only |
| 1010 | LAN | 10.1.0.0/24 | 10.1.0.1 | 10.1.0.100–200 | Pi-hole → pfSense |
| 1020 | Homelab | 10.2.0.0/24 | 10.2.0.1 | 10.2.0.100–200 | Pi-hole → pfSense |
| 1030 | Guests | 10.3.0.0/24 | 10.3.0.1 | 10.3.0.100–250 | Pi-hole → pfSense |
| 1040 | IoT | 10.4.0.0/24 | 10.4.0.1 | 10.4.0.100–250 | Pi-hole → pfSense |
| 1050 | WFH | 10.5.0.0/24 | 10.5.0.1 | 10.5.0.100–200 | pfSense only |
| 1099 | DMZ | 10.99.0.0/24 | 10.99.0.1 | static only | pfSense only |
| — | VPN | 10.200.0.0/24 | pfSense | assigned by WG | Pi-hole → pfSense |

## Firewall Policy

Default: **deny all inter-VLAN unless explicitly allowed.**

| VLAN | Policy Summary |
|------|---------------|
| LAN (1010) | Full internet; can reach Homelab + MGMT; blocked from Guest/IoT/WFH |
| Homelab (1020) | Internet for updates (HTTP/S, SSH, NTP); cannot initiate to other VLANs |
| Guests (1030) | Internet only — hard block on all RFC1918 |
| IoT (1040) | Internet + Home Assistant (explicit rule); blocked from LAN |
| WFH (1050) | Internet only; pfSense DNS only; no personal network access |
| MGMT (1000) | Updates + NTP outbound; inbound from LAN + VPN only |
| DMZ (1099) | HTTP/S + NTP outbound; hard-blocked from all internal VLANs |
| VPN (10.200.0.0/24) | Same as LAN: Homelab + MGMT web GUI + Pi-hole DNS |

## Static IP Reservations

### VLAN 1000 — MGMT

| IP | Device |
|----|--------|
| 10.0.0.1 | pfSense MGMT |
| 10.0.0.2 | Omada Switch |
| 10.0.0.3 | Guest AP |
| 10.0.0.4 | IoT AP |

### VLAN 1010 — LAN

| IP | Device |
|----|--------|
| 10.1.0.1 | pfSense LAN gateway |

### VLAN 1020 — Homelab

| IP | Device |
|----|--------|
| 10.2.0.1 | pfSense Homelab gateway |
| 10.2.0.10 | Proxmox |
| 10.2.0.11 | Pi-hole |
| 10.2.0.20 | Caddy (infra LXC) |
| 10.2.0.21 | Vaultwarden (vault LXC) |
| 10.2.0.25 | Authentik (auth LXC) |
| 10.2.0.51 | Monitor LXC |
| 10.2.0.60 | Apps LXC |

### VLAN 1099 — DMZ

| IP | Device |
|----|--------|
| 10.99.0.1 | pfSense DMZ gateway |
| 10.99.0.20 | Public Service A |
| 10.99.0.22 | Public Service B |
| 10.99.0.23 | Public Service C |

## IP Block Allocation (VLAN 1020)

| Block | Purpose |
|-------|---------|
| 10.2.0.1–9 | Infrastructure (gateway, pfSense interfaces) |
| 10.2.0.10–19 | Network critical (Proxmox, Pi-hole) |
| 10.2.0.20–29 | Auth / Proxy (Caddy, Authentik, Vaultwarden) |
| 10.2.0.30–39 | Observability |
| 10.2.0.40–49 | Dev tools |
| 10.2.0.50–59 | Data |
| 10.2.0.60–69 | Apps |
| 10.2.0.70–79 | Files |
| 10.2.0.80–99 | Media |
| 10.2.0.100+ | DHCP pool (dynamic) |

## DNS Architecture

```
Device → Pi-hole (10.2.0.11)
           ↓
    pfSense Unbound (10.x.0.1) — local records + DHCP hostnames
           ↓
    Cloudflare 1.1.1.1 (upstream)
```

- Pi-hole: ad/tracker blocking, local DNS records (all `*.lerkolabs.com` → 10.2.0.20 Caddy), query logging
- pfSense Unbound: DHCP hostname registration, backup resolver if Pi-hole is down
- WFH VLAN: pfSense DNS only — Pi-hole unreachable by design

## Physical Topology

```
AT&T Fiber ONT
  |
AT&T BGW320 (IP Passthrough)
  |
pfSense N100 (WAN/LAN)
  |
Omada Managed Switch
  ├── Trunk port → pfSense (all VLANs tagged)
  ├── VLAN 1000 — MGMT devices
  ├── VLAN 1010 — Desktop / LAN
  ├── VLAN 1020 — Proxmox / Homelab servers
  ├── VLAN 1030 — Guest WiFi AP
  ├── VLAN 1040 — IoT WiFi AP
  ├── VLAN 1050 — Work laptop
  └── VLAN 1099 — DMZ
```

## WireGuard VPN

| Property | Value |
|----------|-------|
| Listen Port | 51820 UDP |
| VPN Subnet | 10.200.0.0/24 |
| Access granted | Homelab + MGMT web GUI + Pi-hole DNS |
| Access blocked | Guest, IoT, WFH |

No management ports (22, 8006, 443) exposed to the internet. WireGuard is the only inbound port on the WAN interface (aside from Cloudflare DNS-01 challenge traffic, which uses no inbound ports).
