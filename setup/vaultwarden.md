# Vaultwarden Setup

## Overview

Vaultwarden runs in the `vault` LXC (10.2.0.X) in VLAN 1020 (Homelab). It is isolated — no shared containers, no shared Postgres. Accessible at `https://vault.lerkolabs.com` via Caddy with Authentik forward auth. VPN-only access (not exposed to internet directly).

## LXC Spec

| Property | Value |
|----------|-------|
| Hostname | vault |
| IP | 10.2.0.X/24 (TBD) |
| Gateway | 10.2.0.1 |
| DNS | 10.2.0.11 |
| Cores | 1 |
| RAM | 256MB |
| Disk | 4GB |
| Template | debian-12-standard |
| Nesting | ✓ |

## Prerequisites

- Caddy running at 10.2.0.20
- Pi-hole DNS record: `vault.lerkolabs.com → 10.2.0.20`

## Installation

```bash
apt update && apt upgrade -y
apt install -y curl nano
timedatectl set-timezone <your/timezone>
curl -fsSL https://get.docker.com | sh
systemctl enable docker
mkdir -p /opt/docker/vaultwarden/data
```

## Configuration

```yaml
# /opt/docker/vaultwarden/docker-compose.yml
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: unless-stopped
    ports:
      - "80:80"
    volumes:
      - ./data:/data
    environment:
      - DOMAIN=https://vault.lerkolabs.com
      - SIGNUPS_ALLOWED=true    # set false after creating your account
      - WEBSOCKET_ENABLED=true
      - LOG_FILE=/data/vaultwarden.log
      - LOG_LEVEL=warn
      - ROCKET_PORT=80
```

```bash
cd /opt/docker/vaultwarden
docker compose up -d
docker logs -f vaultwarden
```

## Initial Account Setup

1. Navigate to `https://vault.lerkolabs.com`
2. Create your account
3. Set `SIGNUPS_ALLOWED=false` in docker-compose.yml and restart:
   ```bash
   docker compose up -d
   ```

## Enable Admin Panel

```bash
openssl rand -base64 48  # generate admin token
```

Add to environment in docker-compose.yml:

```yaml
- ADMIN_TOKEN=<generated_token>
```

Access admin panel at: `https://vault.lerkolabs.com/admin`

## Caddy Configuration

Add to Caddyfile on infra LXC:

```caddyfile
vault.lerkolabs.com {
    import authentik_forward_auth
    reverse_proxy 10.2.0.X:80
    header {
        Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
        X-Content-Type-Options "nosniff"
        X-Frame-Options "DENY"
        Referrer-Policy "no-referrer"
    }
}
```

## Connecting Bitwarden Clients

In any official Bitwarden client (mobile, desktop, browser extension):

```
Settings → Self-hosted Environment
Server URL: https://vault.lerkolabs.com
```

## Backup

```bash
#!/bin/bash
# /opt/backup-vaultwarden.sh
BACKUP_DIR="/opt/backups/vaultwarden"
DATE=$(date +%Y%m%d-%H%M%S)
mkdir -p "$BACKUP_DIR"

docker stop vaultwarden
tar -czf "$BACKUP_DIR/vaultwarden-$DATE.tar.gz" /opt/docker/vaultwarden/data/
docker start vaultwarden

find "$BACKUP_DIR" -name "*.tar.gz" -mtime +7 -delete
```

```bash
chmod +x /opt/backup-vaultwarden.sh
crontab -e
# Add: 0 2 * * * /opt/backup-vaultwarden.sh >> /var/log/vaultwarden-backup.log 2>&1
```

## Verification

```bash
# Container running
docker ps

# Accessible via Caddy
curl -I https://vault.lerkolabs.com
# Expected: HTTP/2 200 or 302 (Authentik redirect)

# Data directory exists
ls /opt/docker/vaultwarden/data/
```

## Updates

```bash
cd /opt/docker/vaultwarden
docker compose pull
docker compose up -d
docker image prune -f
```
