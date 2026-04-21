# WireGuard Setup

## Overview

WireGuard VPN is configured directly in pfSense. It runs on UDP port 51820 — the only inbound port on the WAN interface. VPN clients get IPs in the 10.200.0.0/24 subnet and receive the same network access as LAN (Homelab + MGMT web GUI + Pi-hole DNS). No external software needed — pfSense handles it natively.

| Property | Value |
|----------|-------|
| Listen Port | 51820 UDP |
| VPN Subnet | 10.200.0.0/24 |
| Access granted | Homelab (10.2.0.0/24) + MGMT web GUI + Pi-hole DNS |
| Access blocked | Guest, IoT, WFH VLANs |

## Prerequisites

- pfSense running and accessible
- WireGuard package installed (System → Package Manager → Available Packages → WireGuard)
- Port 51820 UDP forwarded/open on WAN if behind NAT (not needed with IP Passthrough — pfSense has the public IP directly)
- DDNS client configured on pfSense if WAN IP is dynamic

## Installation

### 1. Install WireGuard Package

Navigate to: **System → Package Manager → Available Packages**

Search "WireGuard" → Install.

### 2. Create WireGuard Tunnel

Navigate to: **VPN → WireGuard → Tunnels → Add Tunnel**

| Setting | Value |
|---------|-------|
| Enabled | ✓ |
| Description | HomeVPN |
| Listen Port | 51820 |
| Interface Keys | Click "Generate" |
| Interface Addresses | 10.200.0.1/24 |

Save. Note the **server public key** — you'll need it in peer configs.

### 3. Add Peers (Clients)

Navigate to: **VPN → WireGuard → Peers → Add Peer**

For each client device:

| Setting | Value |
|---------|-------|
| Tunnel | HomeVPN |
| Description | e.g., iPhone |
| Public Key | (generate on client, paste here) |
| Allowed IPs | 10.200.0.X/32 (unique per peer) |

### 4. Create WireGuard Interface

Navigate to: **Interfaces → Assignments**

Assign the WireGuard tunnel as a new interface (e.g., `OPT1`). Rename it to `WG` or `VPN`.

Enable the interface: Interfaces → WG → Enable ✓

### 5. Firewall Rules

#### WAN — allow inbound WireGuard

Navigate to: **Firewall → Rules → WAN → Add**

| Setting | Value |
|---------|-------|
| Action | Pass |
| Protocol | UDP |
| Destination | WAN address |
| Destination Port | 51820 |
| Description | WireGuard VPN |

#### WG interface — allow VPN clients same access as LAN

Navigate to: **Firewall → Rules → WG → Add**

```
Pass | IPv4 | Source: WG net | Destination: 10.2.0.0/24 | any | Homelab access
Pass | IPv4 | Source: WG net | Destination: 10.0.0.0/24 | 443 | MGMT web GUI
Pass | IPv4 | Source: WG net | Destination: 10.2.0.11 | 53 | Pi-hole DNS
```

### 6. DNS for VPN Clients

In WireGuard peer config, set DNS to 10.2.0.11 (Pi-hole) so VPN clients get ad blocking and local name resolution.

## Client Configuration

Generate on each client device. Structure:

```ini
[Interface]
PrivateKey = <client private key>
Address = 10.200.0.X/24
DNS = 10.2.0.11

[Peer]
PublicKey = <server public key from pfSense>
Endpoint = <WAN IP or DDNS hostname>:51820
AllowedIPs = 10.0.0.0/8   # route all RFC1918 through VPN, or use split tunnel
PersistentKeepalive = 25
```

In pfSense you can generate QR codes for mobile clients: VPN → WireGuard → Peers → (peer) → QR code icon.

## Key Rotation

When rotating keys or adding/removing peers:

1. Generate new key pair on client
2. Update peer's public key in pfSense: VPN → WireGuard → Peers → Edit
3. Update client config with new private key
4. Apply changes in pfSense

## Verification

```bash
# From a mobile device on cellular (not home WiFi):
# 1. Connect WireGuard
# 2. curl https://outline.lerkolabs.com → should load with Authentik login
# 3. curl http://10.2.0.11/admin → Pi-hole admin should be reachable

# On pfSense shell:
wg show  # should show peer with recent handshake
```
