---
tags: [research, reverse-proxy, tls, caddy, traefik, npm, authelia, acme]
created: 2026-09-17 21:38:22 +05
modified: 2026-09-17 21:42:45 +05
---

# Reverse Proxy + TLS + Auth Stack

Research date: 2026-09-17.

## Caddy vs Traefik vs NPM (2026)

- Caddy v2.11: automatic ACME + renew + `tls internal` local CA, HTTP/3 on by default, `forward_auth` built in. Minimal config.
- Traefik v3.7: config lives in Docker labels, HTTP/3 production-ready, powerful middleware, steep learning curve (entrypoints/routers/services).
- NPM on Nginx 1.25 (v3 slow in progress): GUI for proxy host + LE cert toggle; needs MariaDB sidecar, no HTTP/3, doesn't scale or git-version. OK for <10 static services; dev velocity lags.

| Scenario | Choice |
|----------|--------|
| Minimal config, auto HTTPS | Caddy |
| Many dynamic Docker services / GitOps | Traefik |
| GUI-first, few services | NPM |
| Internal LAN + Tailscale | Caddy `tls internal` + AdGuard split DNS |
| A few public services | Caddy DNS-01 wildcard or Cloudflare Tunnel |

## Split recommendation

- Keep everything internal (AdGuard DNS rewrite `*.home` -> 10.10.10.x, or Tailscale split DNS) and expose only 2-4 services. Public exposure does NOT mean opening 80/443: prefer DNS-01 wildcard certs (no inbound ports) or Cloudflare Tunnel.
- 80% internal-only, 20% public (Vaultwarden, Gitea, maybe Jellyfin).

## Configs

Caddyfile:

```
{
    email you@example.com
}
vaultwarden.home:80 {
    tls internal
    reverse_proxy vaultwarden:8000
}
jellyfin.example.com {
    tls { dns cloudflare {env.CF_API_TOKEN} }
    reverse_proxy jellyfin:8096
}
auth.example.com { reverse_proxy authelia:9091 }
gitea.example.com {
    forward_auth authelia:9091 {
        uri /api/authz/forward-auth
        copy_headers Remote-User Remote-Name Remote-Email
    }
    reverse_proxy gitea:3000
}
```

(Requires custom image with caddy-dns/cloudflare for the DNS challenge.)

Traefik labels (per service):

```
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.grafana.rule=Host(`grafana.example.com`)"
  - "traefik.http.routers.grafana.entrypoints=websecure"
  - "traefik.http.routers.grafana.tls.certresolver=letsencrypt"
  - "traefik.http.routers.grafana.middlewares=auth"
  - "traefik.http.services.grafana.loadbalancer.server.port=3000"
```

NPM: Web UI :81 -> Proxy Hosts -> Add (domain, forward IP/port, Websockets + Block common exploits) -> SSL -> Request new cert (Let's Encrypt auto, or Cloudflare/DuckDNS creds for DNS challenge) -> Force SSL.

## Auth

- Authelia: lightweight Go binary, forward-auth, TOTP/WebAuthn (MFA), YAML config, SQLite. Perfect for "only panels + *arrs need a login".
- Authentik: full SSO/OIDC IdP (Google/GitHub providers, app library) but needs Postgres + Redis, heavier updates. Choose only if you want central SSO across many apps.
- Don't protect apps with their own good auth (Vaultwarden, Jellyfin). Basic option: Caddy basic_auth or Tailscale identity as auth layer.

## TLS details

- LE limits: 50 certs/registered-domain/week, 5 for same cert set/week, 300 orders/account/3h. Use staging for tests.
- Wildcards only via DNS-01.
- Internal: Caddy `tls internal` is zero-ops; mkcert fine for one/small fleets; step-ca gives ACME auto-renewal across all devices (needs root installed per client).

## Recommendation for this box

One Caddy on 10.10.10.x (Proxmox LXC/VM). `*.home` vhosts + `tls internal`, AdGuard rewrite -> proxy. Public: wildcard `*.example.com` via DNS-01 (Cloudflare) so ports 80/443 stay closed. Authelia forward_auth on the 3-4 apps that need it. Skip NPM (GUI, slow) and Authentik (overkill). Add Tailscale for anywhere-access identity.

## Sources

- https://wz-it.com (Caddy/Traefik/nginx 2026 comparison)
- https://dokploy.com/blog/caddy-vs-traefik-vs-nginx
- https://caddyserver.com/docs/caddyfile/directives/tls
- https://www.authelia.com/integration/proxies/caddy
- https://letsencrypt.org/docs/rate-limits