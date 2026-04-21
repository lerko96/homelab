# Decisions

Architecture Decision Records (ADR-lite). Key choices and rationale.

---

## D001 — Public/private split: git subtree + push script + pre-push hook

**Decision:** Public content lives under `public/`. Push script uses `git subtree push --prefix=public` to publish it as root to the public remote. Pre-push hook blocks direct pushes that bypass the script.  
**Why:** git-filter-repo is designed for one-time rewrites, not recurring pushes. Separate branches require manual discipline. git subtree is pure git, produces clean history on the public remote, and the script stays two lines.  
**Status:** decided

---

## D002 — Public remote: self-hosted Gitea → GitHub mirror

**Decision:** Public remote is a self-hosted Gitea instance. Gitea mirrors to GitHub automatically.  
**Why:** Matches existing portfolio site setup. Local workflow only pushes to Gitea; GitHub propagation is transparent. No extra tooling needed.  
**Constraint:** Filtering must be airtight before push — whatever reaches Gitea lands on GitHub within seconds.  
**Status:** decided

---

## D003 — Private remote: private repo on same Gitea instance

**Decision:** Private remote is a separate private repository on the same self-hosted Gitea instance.  
**Why:** Easiest path — infrastructure already exists, one tool to manage.  
**Risk:** Single point of failure. If Gitea host goes down, both remotes are inaccessible. Accepted for now.  
**Status:** decided

---

## D004 — Shared Postgres + Redis in apps LXC

**Decision:** Single Postgres instance with multiple databases + single Redis instance, both in the `apps` LXC. All productivity apps share this infrastructure.  
**Why:** Avoids 15 separate DB containers. A single init script provisions all schemas on first run.  
**Risk:** If Postgres goes down, all productivity apps go down simultaneously.  
**Status:** decided

---

## D005 — AT&T gateway kept in-line (IP Passthrough, not EAP bypass)

**Decision:** BGW320 stays in-line with IP Passthrough mode (DHCPS-fixed to pfSense WAN MAC). pfSense gets the public IP directly. Gateway WiFi disabled.  
**Why:** AT&T locks 802.1X certificate auth to their gateway hardware. EAP proxy bypass breaks on AT&T firmware updates and only saves 1–2ms latency. True bridge mode not supported.  
**Status:** decided

---

## D006 — Caddy over NGINX Proxy Manager, with Cloudflare DNS-01

**Decision:** Caddy with `caddy-dns/cloudflare` plugin. DNS-01 challenge via Cloudflare API. All `*.lerkolabs.com` subdomains → 10.2.0.20 in Pi-hole. Caddy terminates SSL, proxies to backends.  
**Why:** Single Caddyfile, auto-cert, no UI overhead. No port 80/443 needed on WAN.  
**Alternatives:** NGINX Proxy Manager (more UI overhead), Traefik (more complex config, same result), self-signed certs (browser warnings).  
**Status:** decided

---

## D007 — WireGuard over OpenVPN

**Decision:** WireGuard on pfSense, UDP 51820, VPN subnet 10.200.0.0/24. VPN clients get same access as LAN.  
**Why:** Lower latency, better mobile battery life, ~600Mbps on the N100. OpenVPN adds complexity with no advantage here.  
**Status:** decided

---

## D008 — Authentik over Authelia

**Decision:** Authentik as SSO provider for all services.  
**Why:** Full OIDC provider + forward auth in one. Lets services like Outline, Gitea, and Vikunja use real SSO rather than just a login gate. Authelia only does forward auth.  
**Status:** decided

---

## D009 — Pi-hole in Homelab VLAN (1020), not MGMT

**Decision:** Pi-hole at 10.2.0.11 in VLAN 1020. Firewall allows port 53 inbound from all VLANs. MGMT VLAN uses pfSense as primary DNS.  
**Why:** Placing Pi-hole in MGMT would require allowing all VLANs to reach MGMT — larger attack surface than filtering DNS traffic from Homelab VLAN.  
**Status:** decided

---

## D010 — Intel N100 for pfSense

**Decision:** Intel N100 mini PC. 4-core 3.4GHz, ~6W idle. Handles 2–3Gbps routing, 600–900Mbps WireGuard.  
**Why:** Right-sized for 1Gbps fiber with headroom. Raspberry Pi insufficient for 1Gbps + VPN. Full rack server overkill power draw.  
**Status:** decided
