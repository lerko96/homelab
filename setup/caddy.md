# Caddy (infra LXC) Setup

## Overview

The `infra` LXC (10.2.0.20) in VLAN 1020 runs Caddy as the internal reverse proxy. It handles TLS termination for all `*.lerkolabs.com` services using wildcard certs via Cloudflare DNS-01 challenge. Also hosts ntfy (push notifications) and Uptime Kuma (service monitoring) as co-located services.

| Service | Port | Domain |
|---------|------|--------|
| Caddy | 80/443 | reverse proxy — no direct domain |
| ntfy | 8090 | ntfy.lerkolabs.com |
| Uptime Kuma | 3001 | uptime.lerkolabs.com |

## Prerequisites

- LXC created at 10.2.0.20 in VLAN 1020
- Cloudflare API token with Zone → DNS → Edit permissions for lerkolabs.com (stored in Vaultwarden)
- Pi-hole DNS record: `*.lerkolabs.com → 10.2.0.20`

## LXC Spec

| Property | Value |
|----------|-------|
| Hostname | infra |
| IP | 10.2.0.20/24 |
| Gateway | 10.2.0.1 |
| Cores | 2 |
| RAM | 1GB |
| Template | debian-12-standard |
| Nesting | ✓ (required for Docker) |

## Installation

```bash
apt update && apt upgrade -y
apt install -y curl nano ufw
timedatectl set-timezone <your/timezone>
curl -fsSL https://get.docker.com | sh
systemctl enable docker
```

## Directory Structure

```
/opt/docker/
├── caddy/
│   ├── Caddyfile
│   ├── docker-compose.yml
│   ├── Dockerfile
│   ├── .env
│   ├── data/
│   ├── config/
│   └── logs/
└── infra/
    ├── uptimekuma/
    │   ├── docker-compose.yml
    │   └── data/
    └── ntfy/
        ├── docker-compose.yml
        ├── server.yml
        └── data/
```

## Caddy Deployment

### Dockerfile (custom build with Cloudflare DNS plugin)

```dockerfile
FROM caddy:<see private/configs/versions.md> AS builder
RUN xcaddy build \
    --with github.com/caddy-dns/cloudflare

FROM caddy:<see private/configs/versions.md>
COPY --from=builder /usr/bin/caddy /usr/bin/caddy
```

### .env

```bash
CLOUDFLARE_API_TOKEN=<token from Vaultwarden: homelab/cloudflare-api>
```

```bash
chmod 600 /opt/docker/caddy/.env
```

### docker-compose.yml

```yaml
services:
  caddy:
    build: .
    container_name: caddy
    restart: unless-stopped
    network_mode: host
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - ./data:/data
      - ./config:/config
      - ./logs:/logs
    env_file:
      - .env
```

### Caddyfile structure

```caddyfile
{
    email <your-acme-email>
    acme_dns cloudflare {env.CLOUDFLARE_API_TOKEN}
}

# Forward auth snippet — reuse in every protected service block
(authentik_forward_auth) {
    forward_auth 10.2.0.25:9000 {
        uri /outpost.goauthentik.io/auth/caddy
        copy_headers X-Authentik-Username X-Authentik-Groups X-Authentik-Email \
                     X-Authentik-Uid X-Authentik-Jwt X-Authentik-Meta-Jwks \
                     X-Authentik-Meta-Outpost X-Authentik-Meta-Provider \
                     X-Authentik-Meta-App X-Authentik-Meta-Version
        trusted_proxies private_ranges
    }
}

# Authentik
auth.lerkolabs.com {
    reverse_proxy 10.2.0.25:9000
}

# Pi-hole (forward auth)
pihole.lerkolabs.com {
    import authentik_forward_auth
    reverse_proxy 10.2.0.11:80
}

# Add remaining services following the same pattern
# Services with native OIDC (Outline, Gitea, Vikunja): no forward auth needed
# Services without OIDC: import authentik_forward_auth
```

### Build and start

```bash
cd /opt/docker/caddy
docker compose build
docker compose up -d
docker logs -f caddy
# Wait for: "certificate obtained successfully"
```

## ntfy Deployment

### server.yml

```yaml
base-url: https://ntfy.lerkolabs.com
listen-http: ":8090"
cache-file: /var/cache/ntfy/cache.db
auth-file: /var/lib/ntfy/auth.db
auth-default-access: deny-all
behind-proxy: true
```

### docker-compose.yml

```yaml
services:
  ntfy:
    image: binwiederhier/ntfy:latest
    container_name: ntfy
    restart: unless-stopped
    command: serve
    ports:
      - "8090:8090"
    volumes:
      - ./server.yml:/etc/ntfy/server.yml:ro
      - ./data:/var/cache/ntfy
      - ./data:/var/lib/ntfy
```

```bash
cd /opt/docker/infra/ntfy && docker compose up -d
docker exec -it ntfy ntfy user add --role=admin <username>
```

## Uptime Kuma Deployment

```yaml
services:
  uptime-kuma:
    image: louislam/uptime-kuma:latest
    container_name: uptime-kuma
    restart: unless-stopped
    ports:
      - "3001:3001"
    volumes:
      - ./data:/app/data
```

```bash
cd /opt/docker/infra/uptimekuma && docker compose up -d
```

## Caddy Reload

After editing the Caddyfile:

```bash
docker exec caddy caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile
docker exec caddy caddy reload --config /etc/caddy/Caddyfile --adapter caddyfile
```

## Verification

```bash
# Cert issued
docker logs caddy | grep "certificate obtained"

# Service reachable
curl -I https://pihole.lerkolabs.com
# Expected: HTTP/2 200

# Ports bound
ss -tlnp | grep -E "443|8090|3001"
```
