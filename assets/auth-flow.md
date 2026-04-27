# Authentication Flow

Forward auth path for an internal service that doesn't speak OIDC natively. OIDC-native services skip the Caddy auth hop and go to Authentik directly.

```mermaid
sequenceDiagram
  participant U as User
  participant C as Caddy<br/>(reverse proxy)
  participant A as Authentik<br/>(IdP)
  participant S as Internal service

  U->>C: HTTPS request
  C->>A: Forward auth check
  A-->>C: 401 (no session)
  C-->>U: 302 → auth.lerkolabs.com

  U->>A: Login (OIDC or password)
  A-->>U: Set session cookie

  U->>C: HTTPS request + cookie
  C->>A: Forward auth check
  A-->>C: 200 OK + identity headers
  C->>S: Proxy request<br/>(plain HTTP, internal hop)
  S-->>U: Response
```


## Notes

- If Authentik is down, internal services are unreachable. This is an accepted tradeoff.
