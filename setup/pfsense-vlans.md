# pfSense VLAN Setup

## Overview

pfSense (Intel N100 mini PC at 10.0.0.1 / 10.1.0.1) handles firewall, routing, DHCP, DNS resolution, and WireGuard VPN for all 8 VLANs. See [Network](../docs/NETWORK.md) for the full VLAN map and firewall policy. See [Decisions](../docs/DECISIONS.md) D005 for the AT&T IP Passthrough rationale.

## Prerequisites

- pfSense installed on Intel N100 mini PC
- AT&T BGW320 in IP Passthrough mode (pfSense WAN gets public IP)
- Omada managed switch connected to pfSense
- Trunk port between pfSense and switch carrying all VLANs tagged

## VLAN Configuration

### 1. Create VLAN Interfaces

Navigate to: **Interfaces → VLANs → Add**

Create one entry per VLAN:

| VLAN Tag | Parent | Description |
|----------|--------|-------------|
| 1000 | (WAN NIC or LAN NIC) | MGMT |
| 1010 | (LAN NIC) | LAN |
| 1020 | (LAN NIC) | Homelab |
| 1030 | (LAN NIC) | Guests |
| 1040 | (LAN NIC) | IoT |
| 1050 | (LAN NIC) | WFH |
| 1099 | (LAN NIC) | DMZ |

### 2. Assign VLAN Interfaces

Navigate to: **Interfaces → Assignments**

Add each VLAN as a new interface. Enable and configure each:

| Interface | IP | Subnet |
|-----------|-----|--------|
| MGMT (1000) | 10.0.0.1 | /24 |
| LAN (1010) | 10.1.0.1 | /24 |
| Homelab (1020) | 10.2.0.1 | /24 |
| Guests (1030) | 10.3.0.1 | /24 |
| IoT (1040) | 10.4.0.1 | /24 |
| WFH (1050) | 10.5.0.1 | /24 |
| DMZ (1099) | 10.99.0.1 | /24 |

### 3. DHCP Servers

Navigate to: **Services → DHCP Server** — configure one per VLAN:

| VLAN | DHCP Range | DNS |
|------|------------|-----|
| MGMT | 10.0.0.100–150 | pfSense (10.0.0.1) |
| LAN | 10.1.0.100–200 | Pi-hole (10.2.0.11) |
| Homelab | 10.2.0.100–200 | Pi-hole (10.2.0.11) |
| Guests | 10.3.0.100–250 | Pi-hole (10.2.0.11) |
| IoT | 10.4.0.100–250 | Pi-hole (10.2.0.11) |
| WFH | 10.5.0.100–200 | pfSense (10.5.0.1) — Pi-hole intentionally excluded |
| DMZ | static only | pfSense (10.99.0.1) |

### 4. Firewall Rules

Navigate to: **Firewall → Rules** — configure per-interface rules following the policy in [NETWORK.md](../docs/NETWORK.md#firewall-policy).

Key rules:

- Default deny all inter-VLAN (floating rule or per-interface block at end)
- LAN → Homelab: allow (LAN users reach services)
- LAN → MGMT: allow (admin access from home devices)
- Homelab → internet: HTTP/S, SSH, NTP only (for updates)
- Guests → internet only: block all RFC1918
- IoT → internet + Home Assistant: block everything else
- WFH → internet only: block all RFC1918, pfSense DNS only
- MGMT → internet: NTP + updates only; inbound from LAN + VPN only
- DMZ → internet: HTTP/S + NTP; block all internal VLANs

### 5. DNS Resolver (Unbound)

Navigate to: **Services → DNS Resolver**

- Enable: ✓
- Listen on: all interfaces
- Upstream DNS: Cloudflare 1.1.1.1
- DNSSEC: ✓ (optional)

Pi-hole (10.2.0.11) uses pfSense Unbound as its upstream. WFH VLAN devices use pfSense Unbound directly — Pi-hole is unreachable from WFH by firewall rule.

### 6. Static DHCP Reservations

Navigate to: **Services → DHCP Server → [interface] → DHCP Static Mappings**

Add reservations for all homelab hosts from [NETWORK.md](../docs/NETWORK.md#static-ip-reservations).

## Configuration Backup

Navigate to: **Diagnostics → Backup & Restore → Backup Configuration**

Download `config.xml`. Store in Vaultwarden or PBS. This is the single file needed to restore pfSense from scratch.

## Verification

```bash
# From a LAN device:
# 1. Gets IP from DHCP in 10.1.0.100–200 range
ip addr

# 2. DNS resolves via Pi-hole
nslookup google.com  # should show answer from 10.2.0.11

# 3. Internal service resolves
nslookup outline.lerkolabs.com  # should return 10.2.0.20

# 4. Internet access works
curl -I https://google.com
```
