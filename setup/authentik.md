# Authentik Setup

## Overview

Authentik is the centralized identity provider for the entire homelab. It runs in the `auth` LXC (10.2.0.25) in VLAN 1020. It provides two auth mechanisms:

1. **Forward auth** — Caddy asks Authentik "is this person logged in?" before proxying any request. Services that don't support SSO natively get a login wall this way.
2. **OIDC provider** — Services that support OAuth2/OIDC (Outline, Gitea, Vikunja, Grafana, Proxmox) get true single sign-on.

Stack: Postgres + Authentik server + Authentik worker (Redis removed as of 2025.10).

## Prerequisites

- LXC at 10.2.0.25 with nesting enabled (Docker requires it)
- Caddy already deployed at 10.2.0.20
- Pi-hole DNS record: `auth.lerkolabs.com → 10.2.0.20`

## LXC Spec

| Property | Value |
|----------|-------|
| Hostname | auth |
| IP | 10.2.0.25/24 |
| Gateway | 10.2.0.1 |
| Cores | 2 |
| RAM | 2GB |
| Disk | 10GB |
| Template | debian-12-standard |
| Nesting | ✓ |

## Installation

```bash
apt update && apt upgrade -y
apt install -y curl nano
timedatectl set-timezone <your/timezone>
curl -fsSL https://get.docker.com | sh
systemctl enable docker
mkdir -p /opt/docker/authentik/{data,certs,custom-templates}
```

## Configuration

### Generate secrets and create .env

```bash
echo "PG_PASS=$(openssl rand -base64 36 | tr -d '\n')" >> /opt/docker/authentik/.env
echo "AUTHENTIK_SECRET_KEY=$(openssl rand -base64 60 | tr -d '\n')" >> /opt/docker/authentik/.env
```

Add remaining config:

```bash
# PostgreSQL
PG_USER=authentik
PG_DB=authentik

# Pin to current version
AUTHENTIK_TAG=<see private/configs/versions.md>

AUTHENTIK_LOG_LEVEL=info
```

```bash
chmod 600 /opt/docker/authentik/.env
```

### docker-compose.yml

```yaml
services:
  postgresql:
    image: docker.io/library/postgres:<see private/configs/versions.md>
    container_name: authentik-db
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -d ${PG_DB} -U ${PG_USER}"]
      start_period: 20s
      interval: 30s
      retries: 5
      timeout: 5s
    volumes:
      - ./postgres:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: ${PG_PASS}
      POSTGRES_USER: ${PG_USER}
      POSTGRES_DB: ${PG_DB}

  server:
    image: ghcr.io/goauthentik/server:${AUTHENTIK_TAG}
    container_name: authentik-server
    restart: unless-stopped
    command: server
    environment:
      AUTHENTIK_SECRET_KEY: ${AUTHENTIK_SECRET_KEY}
      AUTHENTIK_POSTGRESQL__HOST: postgresql
      AUTHENTIK_POSTGRESQL__USER: ${PG_USER}
      AUTHENTIK_POSTGRESQL__PASSWORD: ${PG_PASS}
      AUTHENTIK_POSTGRESQL__NAME: ${PG_DB}
      AUTHENTIK_LOG_LEVEL: ${AUTHENTIK_LOG_LEVEL:-info}
    volumes:
      - ./data:/data
      - ./custom-templates:/templates
    ports:
      - "9000:9000"
      - "9443:9443"
    depends_on:
      postgresql:
        condition: service_healthy

  worker:
    image: ghcr.io/goauthentik/server:${AUTHENTIK_TAG}
    container_name: authentik-worker
    restart: unless-stopped
    command: worker
    environment:
      AUTHENTIK_SECRET_KEY: ${AUTHENTIK_SECRET_KEY}
      AUTHENTIK_POSTGRESQL__HOST: postgresql
      AUTHENTIK_POSTGRESQL__USER: ${PG_USER}
      AUTHENTIK_POSTGRESQL__PASSWORD: ${PG_PASS}
      AUTHENTIK_POSTGRESQL__NAME: ${PG_DB}
      AUTHENTIK_LOG_LEVEL: ${AUTHENTIK_LOG_LEVEL:-info}
    user: root
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./data:/data
      - ./certs:/certs
      - ./custom-templates:/templates
    depends_on:
      postgresql:
        condition: service_healthy
```

## Start Authentik

```bash
cd /opt/docker/authentik
docker compose up -d
docker logs -f authentik-server
# Wait for: "Everything is ready"
```

## Initial Admin Setup

Navigate to: `http://10.2.0.25:9000/if/flow/initial-setup/` (include trailing slash)

Set admin password for the `akadmin` account. Save in Vaultwarden.

## Caddy Integration

Add to Caddyfile on the infra LXC (see [caddy.md](caddy.md)):

```caddyfile
# Forward auth snippet
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

auth.lerkolabs.com {
    reverse_proxy 10.2.0.25:9000
}
```

## Configure Outpost

In Authentik admin: Applications → Outposts → authentik Embedded Outpost → Edit

Set both:
- `authentik Host`: `https://auth.lerkolabs.com`
- `authentik Host (browser)`: `https://auth.lerkolabs.com`

## Adding Applications

### Forward auth pattern (services without native OIDC)

1. Applications → Providers → Create → Proxy Provider
   - Mode: Forward auth (single application)
   - External host: `https://<service>.lerkolabs.com`
2. Applications → Applications → Create
   - Assign provider, set slug and launch URL
3. Outposts → Embedded Outpost → Edit → add application to Selected Applications

### OIDC pattern (Outline, Gitea, Vikunja)

1. Applications → Providers → Create → OAuth2/OpenID Provider
   - Client type: Confidential
   - Redirect URI: service-specific (see table below)
   - Scopes: openid, email, profile
2. Note the Client ID and Client Secret — configure in the service's .env

| Service | Redirect URI |
|---------|-------------|
| Outline | `https://outline.lerkolabs.com/auth/oidc.callback` |
| Gitea | `https://gitea.lerkolabs.com/user/oauth2/authentik/callback` |
| Vikunja | `https://tasks.lerkolabs.com/auth/openid/authentik` |
| Grafana | `https://grafana.lerkolabs.com/login/generic_oauth` |

OIDC discovery URL pattern: `https://auth.lerkolabs.com/application/o/<slug>/.well-known/openid-configuration`

## Verification

```bash
# All containers healthy
docker ps
# authentik-db      Up X minutes (healthy)
# authentik-server  Up X minutes
# authentik-worker  Up X minutes

# Accessible via Caddy
curl -I https://auth.lerkolabs.com
# Expected: HTTP/2 302 (redirect to login)
```

## Updates

```bash
# Edit .env, bump AUTHENTIK_TAG to new version
docker compose pull
docker compose up -d
```
