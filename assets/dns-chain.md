# DNS Resolution

Two flows, one resolver chain.

## External resolution

Client asks for a public domain.

```mermaid
graph LR
  CLIENT[Client<br/>most VLANs] --> PIHOLE[Pi-hole<br/>filtering + cache]
  PIHOLE -->|miss| UNBOUND[Unbound on firewall<br/>recursive + DNSSEC]
  UNBOUND --> UPSTREAM[Cloudflare<br/>fallback only]

  PIHOLE -.->|blocked| BLOCKED[Ad/tracker<br/>domains]

  classDef client fill:#1f2f3a,stroke:#3a6b8b,color:#d0e0f0
  classDef resolver fill:#1f3a2f,stroke:#3a8b6b,color:#d0f0e0
  classDef upstream fill:#3a2f1f,stroke:#8b6b3a,color:#f0e0d0
  classDef blocked fill:#3a1f1f,stroke:#8b3a3a,color:#f0d0d0

  class CLIENT client
  class PIHOLE,UNBOUND resolver
  class UPSTREAM upstream
  class BLOCKED blocked
```

## Local hostname resolution (split-horizon)

Client asks for an internal hostname. The query stays on the LAN. Pi-hole answers from local A records and the client connects to the internal reverse proxy.

```mermaid
graph LR
  CLIENT[Client] -->|asks for<br/>app.lerkolabs.com| PIHOLE[Pi-hole<br/>local A records]
  PIHOLE -->|returns<br/>internal IP| CLIENT
  CLIENT -->|HTTPS<br/>valid public cert| CADDY[Internal Caddy<br/>reverse proxy]
  CADDY --> SVC[Internal service]

  classDef client fill:#1f2f3a,stroke:#3a6b8b,color:#d0e0f0
  classDef resolver fill:#1f3a2f,stroke:#3a8b6b,color:#d0f0e0
  classDef edge fill:#2f1f3a,stroke:#6b3a8b,color:#e0d0f0

  class CLIENT client
  class PIHOLE resolver
  class CADDY,SVC edge
```

