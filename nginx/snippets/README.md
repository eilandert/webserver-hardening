# nginx / Angie hardening snippets

Drop-in `include` files for hardening real-world vhosts on the
[deb.myguard.nl](https://deb.myguard.nl/) nginx/Angie stack. Each file is small,
commented, and safe to include as-is — directives that need an extra module or
patch are no-ops on a vanilla build and are flagged inline.

Copy the files to `/etc/nginx/snippets/` (or `/etc/angie/snippets/`) and
`include` the ones you want.

## The snippets

| File | Context | What it does |
|------|---------|--------------|
| [`ssl.conf`](ssl.conf) | `server` (443) | TLS 1.2/1.3 only, forward-secret AEAD ciphers, OCSP stapling, session hardening, **dynamic TLS records**, **certificate compression** (brotli/zlib/zstd), HSTS. |
| [`security-headers.conf`](security-headers.conf) | `server`/`location` | `X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy`, `Permissions-Policy`, COOP/CORP, a strict default `Content-Security-Policy`. |
| [`bad-bots.conf`](bad-bots.conf) | `http` + `server` | Maps that flag empty-UA, scanners, aggressive scrapers, **AI exploit/data crawlers**, and known probe paths → drop with `444`. |
| [`deny-common.conf`](deny-common.conf) | `server` | Deny dotfiles/VCS, backups, source/config/schema, dep manifests, docs — while keeping ACME reachable. |
| [`realip.conf`](realip.conf) | `http`/`server` | Recover the real client IP behind a trusted proxy (so rate-limits/logs/app brute-force tracking are accurate). Also defines `$connection_upgrade`. |
| [`proxy.conf`](proxy.conf) | `location` | Reverse-proxy hardening: forward real client/scheme, WebSocket upgrade, timeouts/buffers, hide upstream identity. |
| [`php-fpm.conf`](php-fpm.conf) | `location` | Safe FastCGI params for a PHP front controller (no path-traversal `SCRIPT_FILENAME`, hide `X-Powered-By`). |
| [`pagespeed-main.conf`](pagespeed-main.conf) | `http` | ngx_pagespeed global setup (cache, threads, admin handlers OFF). |
| [`pagespeed-site.conf`](pagespeed-site.conf) | `server` | Curated PageSpeed filter set (CSS/JS combine+minify, image recompress, cache extend). |
| [`vhost-roundcube.conf`](vhost-roundcube.conf) | full vhost | Hardened Roundcube webmail vhost (front-controller-only PHP, login throttle). |
| [`vhost-vimbadmin.conf`](vhost-vimbadmin.conf) | full vhost | Hardened ViMbAdmin vhost using a **positive-security** allowlist (method/route/arg). |
| [`vhost-vaultwarden.conf`](vhost-vaultwarden.conf) | full vhost | Hardened TLS **reverse proxy to a Vaultwarden container** (WebSocket sync, `/admin` locked down). |

## Quick start

```nginx
# /etc/nginx/nginx.conf  (http context)
http {
    server_tokens off;
    include snippets/realip.conf;        # real_ip + $connection_upgrade
    include snippets/bad-bots.conf;      # bot/scanner classifiers
    # ... your limit_req_zone / upstream blocks (see each vhost header) ...
}
```

```nginx
# a vhost
server {
    listen 443 ssl;
    http2 on;
    server_name app.example.com;
    ssl_certificate     /etc/letsencrypt/live/app.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/app.example.com/privkey.pem;
    include snippets/ssl.conf;
    include snippets/security-headers.conf;
    include snippets/deny-common.conf;

    if ($hardening_bad_ua)  { return 444; }
    if ($hardening_bad_uri) { return 444; }
    # ...
}
```

Always `nginx -t` (or `angie -t`) before reload.

## A note on `add_header` inheritance

A child `location` that sets **any** `add_header` silently drops **all**
`add_header` directives inherited from the parent `server`. If you add a header
inside a location, re-`include snippets/security-headers.conf;` there too.

## Why these exist — read first

The bot/scanner gate is the practical companion to this writeup on the new wave
of automated, AI-assisted attackers:

- **[Defending your webserver against vibe-coded AI exploit scanners and bots](https://deb.myguard.nl/2026/06/defend-webserver-vibe-coded-ai-exploit-scanners-bots/)**

The TLS, certificate-compression and dynamic-record bits come from the same
hardened build the snippets target — see [`../patches/`](../patches/) and the
[nginx modules page](https://deb.myguard.nl/nginx-modules/).

## Matching containers

The three example vhosts mirror the hardened images in
[github.com/eilandert/dockerized](https://github.com/eilandert/dockerized)
(`docker.io/eilandert/{roundcube,vimbadmin}`, and any app behind
`docker.io/eilandert/{nginx,angie}`). Those images ship the same single-file
hardened config these snippets are distilled from.

## See also

- [deb.myguard.nl](https://deb.myguard.nl/) — the package repo + articles
- [../README.md](../README.md) — the hardened nginx **patches**
- [../../openssl/README.md](../../openssl/README.md) — the **openssl-nginx** patches
- [../../README.md](../../README.md) — repo overview
- Discord: [discord.gg/UQNsFg2y](https://discord.gg/UQNsFg2y)
