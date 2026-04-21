# Apps LXC Setup

## Overview

The `apps` LXC (10.2.0.60) in VLAN 1020 runs 15+ productivity apps via Docker Compose. All services run behind Authentik SSO (OIDC or forward auth). Shared infrastructure: single Postgres instance + single Redis instance, both local to the LXC. All services use `network_mode: host` to reach shared Postgres/Redis on localhost.

## LXC Spec

| Property | Value |
|----------|-------|
| Hostname | apps |
| IP | 10.2.0.60/24 |
| Gateway | 10.2.0.1 |
| DNS | 10.2.0.11 |
| Cores | 4 |
| RAM | 6GB |
| Disk | 80GB |
| Template | debian-12-standard |
| Nesting | ✓ (required for Docker) |
| Unprivileged | ✓ |

## Services

| Service | Port | Domain | DB | Auth |
|---------|------|--------|----|------|
| Outline | 3000 | outline.lerkolabs.com | Postgres | OIDC |
| Gitea | 3001 | gitea.lerkolabs.com | Postgres | OIDC |
| Vikunja | 3456 | tasks.lerkolabs.com | Postgres | OIDC |
| Ghostfolio | 3333 | finance.lerkolabs.com | Postgres | Forward auth |
| Hoarder | 3002 | hoarder.lerkolabs.com | Postgres | Forward auth |
| Grist | 8484 | grist.lerkolabs.com | SQLite | Forward auth |
| Glance | 8080 | glance.lerkolabs.com | — | Forward auth |
| Actual Budget | 5006 | budget.lerkolabs.com | File | Forward auth |
| FreshRSS | 8081 | rss.lerkolabs.com | SQLite | Forward auth |
| Memos | 5230 | memos.lerkolabs.com | SQLite | Forward auth |
| Traggo | 3030 | time.lerkolabs.com | SQLite | Forward auth |
| Baikal | 8082 | dav.lerkolabs.com | SQLite | Forward auth |
| Filebrowser | 8083 | files.lerkolabs.com | SQLite | Forward auth |
| Bytestash | 8084 | — | SQLite | Forward auth |

## Prerequisites

- LXC created with nesting enabled before first start
- Authentik OIDC providers created for Outline, Gitea, Vikunja before starting those services
- Caddy Caddyfile updated with all service blocks (see Phase: Caddy)

## Installation

```bash
apt update && apt upgrade -y
apt install -y curl wget git nano ufw
timedatectl set-timezone <your/timezone>
curl -fsSL https://get.docker.com | sh
systemctl enable docker
```

## Directory Structure

```bash
mkdir -p /opt/docker/apps/{shared,outline,gitea,vikunja,ghostfolio,hoarder,grist,glance,actual,freshrss,memos,traggo,baikal,filebrowser,bytestash}
```

## Shared Infrastructure (Postgres + Redis)

Start this first, before anything else.

### Shared .env

```bash
# /opt/docker/apps/.env
PG_ROOT_PASSWORD=      # openssl rand -base64 32
REDIS_PASSWORD=        # openssl rand -base64 24
PG_PASS_OUTLINE=       # openssl rand -base64 24
PG_PASS_GITEA=         # openssl rand -base64 24
PG_PASS_VIKUNJA=       # openssl rand -base64 24
PG_PASS_GHOSTFOLIO=    # openssl rand -base64 24
PG_PASS_HOARDER=       # openssl rand -base64 24
PG_PASS_GRIST=         # openssl rand -base64 24
```

```bash
chmod 600 /opt/docker/apps/.env
```

### shared/docker-compose.yml

```yaml
services:
  postgres:
    image: postgres:<see private/configs/versions.md>
    container_name: apps-postgres
    restart: unless-stopped
    env_file: /opt/docker/apps/.env
    environment:
      POSTGRES_PASSWORD: ${PG_ROOT_PASSWORD}
      POSTGRES_USER: postgres
    volumes:
      - ./pgdata:/var/lib/postgresql/data
      - ./init:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
    ports:
      - "127.0.0.1:5432:5432"
    networks:
      - apps-net

  redis:
    image: redis:<see private/configs/versions.md>
    container_name: apps-redis
    restart: unless-stopped
    command: redis-server --requirepass ${REDIS_PASSWORD} --save 60 1 --loglevel warning
    env_file: /opt/docker/apps/.env
    volumes:
      - ./redisdata:/data
    ports:
      - "127.0.0.1:6379:6379"
    networks:
      - apps-net

networks:
  apps-net:
    name: apps-net
    driver: bridge
```

### Database init script

```bash
mkdir -p /opt/docker/apps/shared/init
```

`/opt/docker/apps/shared/init/01-create-databases.sql`:

```sql
CREATE USER outline WITH PASSWORD 'PG_PASS_OUTLINE_VALUE';
CREATE DATABASE outline OWNER outline;

CREATE USER gitea WITH PASSWORD 'PG_PASS_GITEA_VALUE';
CREATE DATABASE gitea OWNER gitea;

CREATE USER vikunja WITH PASSWORD 'PG_PASS_VIKUNJA_VALUE';
CREATE DATABASE vikunja OWNER vikunja;

CREATE USER ghostfolio WITH PASSWORD 'PG_PASS_GHOSTFOLIO_VALUE';
CREATE DATABASE ghostfolio OWNER ghostfolio;

CREATE USER hoarder WITH PASSWORD 'PG_PASS_HOARDER_VALUE';
CREATE DATABASE hoarder OWNER hoarder;

CREATE USER grist WITH PASSWORD 'PG_PASS_GRIST_VALUE';
CREATE DATABASE grist OWNER grist;
```

Replace each `_VALUE` with the actual password from your .env. This script runs once on first `docker compose up`.

### Start shared infrastructure

```bash
cd /opt/docker/apps/shared
docker compose --env-file /opt/docker/apps/.env up -d
docker compose logs -f  # wait for "ready to accept connections"
```

### Verify

```bash
docker exec apps-postgres pg_isready -U postgres
docker exec apps-postgres psql -U postgres -c "\l"
# Should show: outline, gitea, vikunja, ghostfolio, hoarder, grist
```

## Startup Order

```bash
# 1. Shared infrastructure always first
cd /opt/docker/apps/shared && docker compose up -d
sleep 15  # give postgres time on first run

# 2. Run Outline migrations before starting Outline
cd /opt/docker/apps/outline && docker compose run --rm outline-migrate
docker compose up -d outline

# 3. Everything else in parallel
for svc in gitea vikunja ghostfolio hoarder grist glance actual freshrss memos traggo baikal filebrowser bytestash; do
  cd /opt/docker/apps/$svc && docker compose up -d
done
```

## Caddy Configuration

Add to Caddyfile on the infra LXC (10.2.0.20). Services with native OIDC don't need forward auth; others do.

```caddyfile
# OIDC-native (no forward auth needed)
outline.lerkolabs.com {
    reverse_proxy 10.2.0.60:3000
}
gitea.lerkolabs.com {
    reverse_proxy 10.2.0.60:3001
}
tasks.lerkolabs.com {
    reverse_proxy 10.2.0.60:3456
}

# Forward auth
finance.lerkolabs.com {
    import authentik_forward_auth
    reverse_proxy 10.2.0.60:3333
}
# ... repeat pattern for remaining services
```

## Pi-hole DNS Records

All records point to 10.2.0.20 (Caddy), not 10.2.0.60 directly:

```
outline.lerkolabs.com    → 10.2.0.20
gitea.lerkolabs.com      → 10.2.0.20
tasks.lerkolabs.com      → 10.2.0.20
finance.lerkolabs.com    → 10.2.0.20
hoarder.lerkolabs.com    → 10.2.0.20
grist.lerkolabs.com      → 10.2.0.20
glance.lerkolabs.com     → 10.2.0.20
budget.lerkolabs.com     → 10.2.0.20
rss.lerkolabs.com        → 10.2.0.20
memos.lerkolabs.com      → 10.2.0.20
time.lerkolabs.com       → 10.2.0.20
dav.lerkolabs.com        → 10.2.0.20
files.lerkolabs.com      → 10.2.0.20
```

## Verification

```bash
# All containers running
docker ps --format "table {{.Names}}\t{{.Status}}" | sort

# Outline health
curl -s http://localhost:3000/api/health

# From LAN — check Authentik gate works
curl -I https://outline.lerkolabs.com
```

## Useful Commands

```bash
# Logs for a service
docker logs -f outline

# Postgres: connect to a database
docker exec -it apps-postgres psql -U postgres -d outline

# Postgres: backup
docker exec apps-postgres pg_dump -U postgres outline > /opt/backups/outline-$(date +%Y%m%d).sql

# Disk usage by service
du -sh /opt/docker/apps/*/
```
