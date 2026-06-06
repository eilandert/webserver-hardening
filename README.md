# webserver-hardening

Patches and config snippets that harden **nginx**, **Angie** and **OpenSSL**
for production — the same set that powers [deb.myguard.nl](https://deb.myguard.nl/)
and the matching container images.

This repo is the source-of-truth, human-readable companion to the pre-built
packages. If you just want the binaries, point `apt` at
[deb.myguard.nl](https://deb.myguard.nl/); if you want to understand *what* is
changed and *why*, or rebuild it yourself, read on.

## What's here

| Path | Contents |
|------|----------|
| [`nginx/patches/`](nginx/patches/) | Core nginx patches: zlib-ng, dynamic TLS records, full HPACK, async cert callback, **certificate compression** (brotli/zstd), **JA3/JA4 fingerprinting** hooks. → [README](nginx/README.md) |
| [`nginx/snippets/`](nginx/snippets/) | Drop-in `include` files: TLS hardening, security headers, **AI-bot/scanner blocking**, PageSpeed, and ready-made hardened vhosts for Roundcube, ViMbAdmin and Vaultwarden. → [README](nginx/snippets/README.md) |
| [`openssl/patches/`](openssl/patches/) | `openssl-nginx` (OpenSSL 4.0.0) patches: async session-lookup yield, JA4 ClientHello accessor. → [README](openssl/README.md) |

## Why

Upstream nginx and OpenSSL leave a lot of practical hardening and performance
on the table because it's "out of scope" for them. This repo collects the
deltas that matter for a real, attacked-every-second public server:

- **Performance:** zlib-ng compression, dynamic TLS record sizing, full HTTP/2
  HPACK, RFC 8879 certificate compression (brotli).
- **Security/visibility:** JA3/JA4 TLS fingerprinting, async cert/session
  callbacks for per-SNI logic, and a battle-tested set of vhost hardening
  snippets.
- **Bots:** the snippets include a coarse-but-cheap gate against scanners and
  the new wave of automated AI exploit crawlers — background reading:
  **[Defending your webserver against vibe-coded AI exploit scanners and bots](https://deb.myguard.nl/2026/06/defend-webserver-vibe-coded-ai-exploit-scanners-bots/)**.

## The stack these target

- **Packages:** [deb.myguard.nl](https://deb.myguard.nl/) — hardened nginx,
  Angie, ModSecurity, openssl-nginx, PHP, Postfix, Dovecot, rspamd.
  - [nginx modules list](https://deb.myguard.nl/nginx-modules/) ·
    [modules synopsis](https://deb.myguard.nl/nginx/modules-synopsis/) ·
    [optimized/extended Angie](https://deb.myguard.nl/angie-modules-optimized-extended/)
- **Containers:** [github.com/eilandert/dockerized](https://github.com/eilandert/dockerized)
  → `docker.io/eilandert/{nginx,angie,roundcube,vimbadmin,…}` — same hardened
  packages, in a container.
- **Chat / bugs / module requests:** Discord
  [discord.gg/UQNsFg2y](https://discord.gg/UQNsFg2y).

## Using it

Patches are quilt-style (`-p1`), apply in the order documented in
[nginx/README.md](nginx/README.md) and verified zero-fuzz (`patch -F0`).
Snippets go in `/etc/nginx/snippets/` (or `/etc/angie/snippets/`) and are
`include`d per vhost — see [nginx/snippets/README.md](nginx/snippets/README.md).

## License

Patches retain their upstream authorship/licence (noted in each patch header
and the per-directory READMEs). Original snippets and docs in this repo are
released under the **MIT License** unless a file states otherwise.
