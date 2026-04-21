# Monitor LXC Setup

## Overview

The `monitor` LXC (10.2.0.51) in VLAN 1020 runs the full observability stack: Victoria Metrics (metrics storage), Grafana (dashboards and alerting), and Beszel (container + host monitoring). All services run via Docker Compose.

## LXC Spec

| Property | Value |
|----------|-------|
| Hostname | monitor |
| IP | 10.2.0.51/24 |
| Gateway | 10.2.0.1 |
| DNS | 10.2.0.11 |
| Cores | 4 |
| RAM | 4GB |
| Template | debian-12-standard |
| Nesting | ✓ |

## Prerequisites

- Caddy running at 10.2.0.20
- Pi-hole DNS records added (see Verification)
- Beszel agents deployed on all LXCs to be monitored

## Installation

```bash
apt update && apt upgrade -y
apt install -y curl nano
timedatectl set-timezone <your/timezone>
curl -fsSL https://get.docker.com | sh
systemctl enable docker
mkdir -p /opt/docker/monitor/{victoria-metrics,grafana,beszel}
```

## Victoria Metrics

```yaml
# /opt/docker/monitor/victoria-metrics/docker-compose.yml
services:
  victoria-metrics:
    image: victoriametrics/victoria-metrics:latest
    container_name: victoria-metrics
    restart: unless-stopped
    ports:
      - "8428:8428"
    volumes:
      - ./data:/storage
    command:
      - "--storageDataPath=/storage"
      - "--retentionPeriod=90d"
```

```bash
cd /opt/docker/monitor/victoria-metrics && docker compose up -d
```

## Grafana

```yaml
# /opt/docker/monitor/grafana/docker-compose.yml
services:
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    volumes:
      - ./data:/var/lib/grafana
    environment:
      GF_SERVER_ROOT_URL: https://grafana.lerkolabs.com
      GF_AUTH_GENERIC_OAUTH_ENABLED: "true"
      GF_AUTH_GENERIC_OAUTH_NAME: Authentik
      GF_AUTH_GENERIC_OAUTH_CLIENT_ID: <from Authentik OIDC provider>
      GF_AUTH_GENERIC_OAUTH_CLIENT_SECRET: <from Authentik OIDC provider>
      GF_AUTH_GENERIC_OAUTH_SCOPES: openid email profile
      GF_AUTH_GENERIC_OAUTH_AUTH_URL: https://auth.lerkolabs.com/application/o/authorize/
      GF_AUTH_GENERIC_OAUTH_TOKEN_URL: https://auth.lerkolabs.com/application/o/token/
      GF_AUTH_GENERIC_OAUTH_API_URL: https://auth.lerkolabs.com/application/o/userinfo/
      GF_AUTH_SIGNOUT_REDIRECT_URL: https://auth.lerkolabs.com/application/o/grafana/end-session/
      GF_AUTH_OAUTH_AUTO_LOGIN: "true"
```

```bash
cd /opt/docker/monitor/grafana && docker compose up -d
```

Add Victoria Metrics as a data source in Grafana: `http://localhost:8428`

## Beszel

Beszel hub runs on the monitor LXC. Beszel agents run on each LXC/VM being monitored.

### Hub (monitor LXC)

```yaml
# /opt/docker/monitor/beszel/docker-compose.yml
services:
  beszel:
    image: henrygd/beszel:latest
    container_name: beszel
    restart: unless-stopped
    ports:
      - "8090:8090"
    volumes:
      - ./data:/beszel_data
```

```bash
cd /opt/docker/monitor/beszel && docker compose up -d
```

### Agents (each LXC)

On each LXC that needs monitoring:

```bash
curl -sL https://raw.githubusercontent.com/henrygd/beszel/main/supplemental/scripts/install-agent.sh -o install-agent.sh
chmod +x install-agent.sh
./install-agent.sh  # follow prompts, enter hub address and key
```

## Caddy Configuration

Add to Caddyfile on infra LXC:

```caddyfile
grafana.lerkolabs.com {
    reverse_proxy 10.2.0.51:3000
}
```

Beszel and Victoria Metrics are internal-only (no public Caddy entries needed unless you want external access).

## Pi-hole DNS Records

```
grafana.lerkolabs.com → 10.2.0.20
```

## Verification

```bash
# All containers running
docker ps

# Victoria Metrics health
curl http://localhost:8428/health

# Grafana reachable
curl -I https://grafana.lerkolabs.com

# Beszel agents reporting
# Check Beszel web UI at http://10.2.0.51:8090
```
