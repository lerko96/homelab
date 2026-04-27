# Network Topology

Two views of the same network.

## Trust tiers and policy

Seven VLANs grouped by trust level. Edges show allowed inter-tier flows; everything else is default-deny.

```mermaid
graph TB
  subgraph UNTRUSTED["Untrusted — internet only, no internal access"]
    GUEST[Guest WiFi]
    IOT[IoT]
    WFH[Work-from-home]
  end

  subgraph PUBLIC["Public-facing"]
    DMZ[DMZ<br/>reverse proxy + public services]
  end

  subgraph TRUSTED["Trusted"]
    LAN[LAN<br/>personal devices]
    INT[Internal services<br/>app stack]
  end

  subgraph MGMT["Management — VPN-only"]
    ADMIN[Hypervisor, firewall,<br/>backup, switches, APs]
  end

  subgraph REMOTE["Remote"]
    VPN[WireGuard clients]
  end

  INTERNET((Internet))

  UNTRUSTED -->|outbound only| INTERNET
  INTERNET -->|HTTP/HTTPS<br/>tight allowlist| DMZ
  INTERNET -->|WireGuard<br/>UDP| VPN

  DMZ -.->|narrow allowlist<br/>firewall-enforced| INT
  LAN -->|consume services| INT
  VPN -->|LAN-equivalent +<br/>admin access| INT
  VPN --> ADMIN

  classDef untrusted fill:#3a1f1f,stroke:#8b3a3a,color:#f0d0d0
  classDef public fill:#3a2f1f,stroke:#8b6b3a,color:#f0e0d0
  classDef trusted fill:#1f3a2f,stroke:#3a8b6b,color:#d0f0e0
  classDef mgmt fill:#1f2f3a,stroke:#3a6b8b,color:#d0e0f0
  classDef remote fill:#2f1f3a,stroke:#6b3a8b,color:#e0d0f0

  class GUEST,IOT,WFH untrusted
  class DMZ public
  class LAN,INT trusted
  class ADMIN mgmt
  class VPN remote
```

## Physical flow

What plugs into what. Tier labels, not addresses.

```mermaid
graph LR
  ISP[ISP] --> GW[Carrier gateway<br/>passthrough mode]
  GW --> FW[pfSense firewall]
  FW --> SW[Managed switch<br/>VLAN-aware]

  SW --> T_MGMT[MGMT tier]
  SW --> T_INT[Internal services tier]
  SW --> T_LAN[LAN tier]
  SW --> T_WFH[WFH tier]
  SW --> T_IOT[IoT tier]
  SW --> T_GUEST[Guest tier]
  SW --> T_DMZ[DMZ tier]

  FW -.->|VPN concentrator| VPN[WireGuard]
```

## Two reverse proxies

The DMZ-to-internal arrow above is by design. There are two Caddy instances:

- One in DMZ, internet-facing, fronting a small set of public services.
- One in internal services tier, LAN/VPN only, fronting everything else.

## Notes

- Inter-tier policy enforced at the firewall.
