# Network

## Trust tiers

| Tier | What's on it | Posture |
|---|---|---|
| Management | Hypervisor, firewall, backup server, network controllers | Most trusted. VPN-only. No outbound unless required. |
| Internal services | LXCs and VMs running the app stack | Trusted. Serves clients in adjacent tiers per policy. |
| LAN | Personal devices on home WiFi/Ethernet | Trusted. Consumes internal services. |
| Work-from-home | Employer-owned laptop | Untrusted lateral. Internet only. Blocked from everything internal, including DNS. |
| IoT | Smart devices, cloud-managed appliances | Untrusted. Internet only. Isolated from internal. |
| Guest | Visitor WiFi | Untrusted. Internet only. |
| DMZ | Internet-facing services | Treated as compromised by default. Tight inbound allowlist to internal. |
| VPN (WireGuard) | Authenticated remote clients | Same posture as LAN, plus admin-tier visibility. |

## Policy

- Default deny inter-VLAN. Every cross-tier flow is an explicit allow rule with a reason.
- WFH and IoT are restricted to internet only. Nothing internal, including DNS for local hostnames.
- Management is kept minimal. Only what runs the lab lives there.
- DMZ is one-way. Public services in there can only initiate inward through a firewall-enforced allowlist by source IP + destination port with reverse proxy reinforcing.
- Admin only accessible via Management + VPN

## DNS

Three layers:

1. **Pi-hole** — first hop for clients on most VLANs. Filters ad/tracker domains and holds local A records. Not used by Management hosts or by the WFH VLAN.
2. **Unbound on the firewall** — Pi-hole's upstream. Recursive resolver, validates DNSSEC.
3. **Cloudflare** — Unbound's upstream when needed.

The hypervisor (which is the box Pi-hole runs on) statically resolves through the firewall, not Pi-hole. If it didn't, there'd be a circular dependency at boot.

## Internet exposure

Three ports forwarded from WAN:

- HTTP and HTTPS to the DMZ reverse proxy.
- WireGuard to the firewall.
