# openssl-nginx patches

Two patches carried on top of **OpenSSL 4.0.0** in the `openssl-nginx` build —
a private, webserver-only OpenSSL that ships as `libssl-nginx.so` /
`libcrypto-nginx.so` so it coexists with the system OpenSSL without conflict.
Both exist to support features in the hardened nginx/Angie build that upstream
OpenSSL doesn't expose.

> These are **our** patches only. The `openssl-nginx` source also carries the
> usual Debian packaging patches (targets, ktls-enable, man-section, PATH_MAX);
> those are upstream/distro and not duplicated here.

Pre-built packages: **[deb.myguard.nl](https://deb.myguard.nl/)**.

## The patches

### [`openssl-4.0.0-sess_set_get_cb_yield.patch`](patches/openssl-4.0.0-sess_set_get_cb_yield.patch)
Lets the TLS **session-lookup callback yield** (OpenResty `cb_yield` lineage).
The session-fetch callback (`SSL_CTX_sess_set_get_cb`) can suspend for
non-blocking I/O — so a lua-resty / `ngx_lua` handler can look a session up in
an external store (Redis, a shared dict, a remote service) mid-handshake
without blocking the nginx worker. Without this, any such lookup is synchronous
and stalls the event loop. Pairs with the nginx
[`nginx-ssl_cert_cb_yield.patch`](../nginx/patches/nginx-ssl_cert_cb_yield.patch)
(the certificate-callback equivalent).

### [`openssl-4.0.0-ssl-fingerprint.patch`](patches/openssl-4.0.0-ssl-fingerprint.patch)
Adds the accessor **`SSL_client_hello_get0_received_ext()`** — returns the
ClientHello extension types in **received (wire) order**, which is exactly what
**JA4** TLS fingerprinting needs (JA4 hashes the extensions in the order the
client sent them). Pure addition: one new accessor + prototype, ABI-safe. The
symbol is also added to the export map (`util/libssl.num`) so it is exported
from the shared `libssl-nginx.so` — upstream omits this because they link nginx
against OpenSSL statically; we build OpenSSL as a shared library, so an
unexported symbol would be an undefined reference at nginx link time.
Feeds [`../nginx/patches/ssl-fingerprint.patch`](../nginx/patches/ssl-fingerprint.patch)
and `libnginx-mod-ssl-fingerprint`.
Origin: <https://github.com/HanadaLee/ngx_ssl_fingerprint_module>.

## Build note — certificate compression

The hardened build also enables **brotli + zstd TLS certificate compression**
(RFC 8879). That needs **no patch** — OpenSSL 4.0.0 implements it internally;
it is purely a build option (`enable-brotli enable-zstd` in the openssl-nginx
configure flags, plus `libbrotli-dev`/`libzstd-dev`). nginx then drives it via
[`../nginx/patches/ssl-cert-compression.patch`](../nginx/patches/ssl-cert-compression.patch).

## See also

- [../nginx/README.md](../nginx/README.md) — the nginx patches
- [../README.md](../README.md) — repo overview
- [deb.myguard.nl](https://deb.myguard.nl/) · Discord [discord.gg/UQNsFg2y](https://discord.gg/UQNsFg2y)
