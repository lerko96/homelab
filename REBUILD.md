# Rebuild

Ordered recovery sequence from scratch or after catastrophic failure. **Nothing works until the thing before it works.** For step-by-step setup, see individual service [setup guides](setup/).

## Phase 1 — Network Foundation

1. **pfSense** — restore `config.xml`; verify WAN gets public IP (IP Passthrough active on BGW320); verify all VLAN interfaces up + DHCP serving; verify firewall rules loaded
2. **Omada Switch** — restore controller backup; verify port VLANs match [Network](docs/NETWORK.md) topology; verify trunk port carrying all VLANs tagged
3. **Access points** — auto-adopt into Omada Controller; verify SSIDs on correct VLANs

**Gate:** LAN device gets IP and reaches internet.

## Phase 2 — DNS

4. **Pi-hole LXC** — restore from PBS snapshot (or fresh deploy); restore Teleporter backup; verify all local DNS records → 10.2.0.20 (Caddy); verify ad blocking active
5. **pfSense DNS Resolver** — auto-configured from `config.xml`; verify Pi-hole is upstream for all VLANs

**Gate:** `nslookup outline.lerkolabs.com` returns 10.2.0.20 from LAN.

## Phase 3 — Reverse Proxy + TLS

6. **Infra LXC (Caddy)** — restore from PBS (or fresh deploy); verify Cloudflare API token valid; start Caddy — certs auto-issue (allow 2–3 min); add Pi-hole DNS record: `*.lerkolabs.com → 10.2.0.20`

**Gate:** `curl -I https://pihole.lerkolabs.com` returns HTTP/2 200.

## Phase 4 — Auth

7. **Auth LXC (Authentik)** — restore from PBS; verify admin accessible at `https://auth.lerkolabs.com`; verify OIDC apps configured (Outline, Gitea, Vikunja); verify forward auth flows

## Phase 5 — Secrets

8. **Vault LXC (Vaultwarden)** — restore from PBS; verify accessible at `https://vault.lerkolabs.com`; confirm all credentials accessible before proceeding

## Phase 6 — Core Services

9. **Apps LXC** — restore from PBS (or fresh deploy); start shared Postgres + Redis first; bring up services one by one: Outline → Gitea → Vikunja → Ghostfolio → Hoarder → Grist → Glance → Actual → FreshRSS → Memos → Traggo → Baikal → Filebrowser → Bytestash
10. **Monitor LXC** — restore from PBS; verify Grafana dashboards loading; verify Beszel agents reporting from all LXCs; verify Victoria Metrics receiving metrics

## Phase 7 — VMs

11. **Servarr VM** — restore from PBS; verify Plex/Jellyfin accessible; verify arr stack healthy; verify Gluetun VPN tunnel active for qBittorrent
12. **Home Assistant OS VM** — restore from PBS (or HAOS backup); verify integrations reconnect

## Phase 8 — VPN

13. **WireGuard** — restored with `config.xml`; verify peer configs valid; test from cellular; if keys rotated, distribute new configs

## Post-Rebuild Checklist

- [ ] Internet works from LAN devices
- [ ] DNS resolves internal and external names
- [ ] All `*.lerkolabs.com` reachable via HTTPS
- [ ] Authentik SSO working (log into Outline via Authentik)
- [ ] WireGuard connects from external network
- [ ] Vaultwarden accessible and credentials intact
- [ ] All Docker containers healthy in Beszel
- [ ] PBS scheduled backups running
- [ ] Pi-hole blocking ads
- [ ] Home Assistant automations running
- [ ] Media stack healthy (Plex/Jellyfin playback works)
