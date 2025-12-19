# nbCloudWebsite – Architecture & Deployment Notes

## Overview

This project hosts a Symfony application running in Docker on a Raspberry Pi 4.
The service is **securely exposed to the internet without any port forwarding**
using **Cloudflare Tunnel**.

Key components:

- **Symfony** (PHP application)
- **FrankenPHP + Caddy** (application server & reverse proxy)
- **MySQL** (Docker container)
- **Docker Compose**
- **Cloudflare Tunnel (cloudflared)**
- **Basic Authentication enforced at Caddy level**
- **Cloudflare DNS + TLS**

The system is designed to be:
- Secure by default
- Fully functional behind NAT / CGNAT / IPv6-only routers
- Free of secrets in git
- Accessible both from LAN and the public internet (with auth)

---

## High-Level Architecture

```
Browser
   |
   |  HTTPS (Cloudflare TLS)
   v
Cloudflare Edge
   |
   |  Encrypted Tunnel (outbound-only)
   v
cloudflared (NAS)
   |
   |  HTTPS / HTTP (LAN)
   v
Caddy + FrankenPHP (Raspberry Pi)
   |
   v
Symfony Application
```

### Important Property

**No inbound ports are open.**
The Raspberry Pi and NAS only create **outgoing connections** to Cloudflare.

---

## Why No Port Forwarding Is Needed

- `cloudflared` initiates outbound connections to Cloudflare
- Cloudflare routes requests for the domain through the tunnel
- The router firewall allows outbound traffic by default
- No public IP is exposed
- Works with:
  - IPv6-only ISPs
  - CGNAT
  - Strict home routers

---

## Authentication Model

### Layer 1 – Cloudflare
- TLS termination
- Tunnel authentication
- Optional WAF / rate limiting (minimal features enabled)

### Layer 2 – Caddy (mandatory)
- HTTP Basic Auth
- Enforced for:
  - Public access (`https://nilsbaczynski.de`)
  - LAN access (`http://192.168.0.43`)

This guarantees:
- No unauthenticated access from anywhere
- LAN is not implicitly trusted

---

## Environment Variables & Secrets

### Tracked
- `.env`
  - Contains defaults and placeholders only
  - No real secrets
  - Required by Symfony

### Not tracked
- `.env.prod`
- `.env.local`
- `.env.prod.local`

All real secrets live only in untracked files.

Example secrets:
- `DATABASE_URL`
- `APP_SECRET`
- `BASIC_AUTH_USER`
- `BASIC_AUTH_HASH`
- `MERCURE_*`

A `.env.example` file documents required variables without values.

---

## Cloudflare Tunnel Configuration

### Location
```
/volume1/docker/cloudflared/
├── config.yml
└── .cloudflared/
    ├── cert.pem
    └── <tunnel-id>.json
```

### Minimal ingress example

```yaml
tunnel: <TUNNEL_ID>
credentials-file: /etc/cloudflared/.cloudflared/<TUNNEL_ID>.json

ingress:
  - hostname: nilsbaczynski.de
    service: http://192.168.0.43:80
  - service: http_status:404
```

### Security
- Credentials readable only by root
- Tunnel auto-reconnects
- No public origin IP exposure

---

## Docker

- Symfony + Caddy run on the Raspberry Pi
- `cloudflared` runs as a persistent container on the NAS
- Docker Compose uses `--env-file .env.prod` for production

---

## Known Warnings (Expected)

```
ICMP proxy feature is disabled
```
Safe to ignore. ICMP is optional and not required for HTTP traffic.

---

## Operational Notes

- Restarting `cloudflared` temporarily causes Cloudflare error 1033
- Service recovers automatically once tunnel reconnects
- Basic Auth remains enforced during all restarts

---

## Security Summary

| Item | Status |
|-----|-------|
| Port forwarding | ❌ None |
| Public IP exposure | ❌ No |
| TLS | ✅ Yes |
| Authentication | ✅ Mandatory |
| Secrets in git | ❌ No |
| Works behind NAT | ✅ Yes |

---

## Final Notes

This setup is intentionally minimal, auditable, and robust.
It favors **outbound-only connectivity**, explicit authentication,
and clean separation of secrets from source control.

