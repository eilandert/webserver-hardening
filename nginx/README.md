# nginx patches

The core patches applied on top of mainline nginx in the
[deb.myguard.nl](https://deb.myguard.nl/) build. They add TLS/HTTP-2 features
that upstream considers out of scope, plus the hooks the hardened dynamic
modules depend on.

These are **quilt** patches (`-p1`, applied from the nginx source root). Order
matters — some touch the same structures (notably `ngx_ssl_connection_s`), so
apply them in the sequence below. All are verified to apply **zero-fuzz**
(`patch -F0`) on the build they ship with.

> The matching pre-built `.deb` packages (with every dynamic module) live at
> **[deb.myguard.nl](https://deb.myguard.nl/)**. The full module list +
> directives is on the **[nginx modules page](https://deb.myguard.nl/nginx-modules/)**
> and the **[modules synopsis](https://deb.myguard.nl/nginx/modules-synopsis/)**.
> Angie users: **[optimized/extended Angie](https://deb.myguard.nl/angie-modules-optimized-extended/)**.

## The patches (apply in this order)

### 1. [`1.30.0-zlib-ng.patch`](patches/1.30.0-zlib-ng.patch)
Build nginx against **zlib-ng** (native API) instead of stock zlib. Swaps the
`auto/lib/zlib/conf` probe to `zlib-ng.h` / `libz-ng`, and adds
`src/core/zlib-ng-compat.h` — a shim that aliases the classic `deflate`/
`inflate` calls to their `zng_*` equivalents, so nginx's gzip/gunzip code is
unchanged. Faster response compression on every request. Build-time only.
Pair with `libz-ng-dev` from the repo.

### 2. [`nginx_dynamic_tls_records.patch`](patches/nginx_dynamic_tls_records.patch)
Cloudflare **dynamic TLS record sizing**. Sends small TLS records during
the first round-trips (fast first paint, no head-of-line stall) then grows to
~16 KB for bulk throughput. Adds directives:
`ssl_dyn_rec_enable`, `ssl_dyn_rec_size_lo`, `ssl_dyn_rec_size_hi`,
`ssl_dyn_rec_threshold`, `ssl_dyn_rec_timeout`. Touches `ngx_ssl_connection_s`
(reorders fields — later TLS patches rebase onto this).

### 3. [`nginx_hpack.patch`](patches/nginx_hpack.patch)
Full **HPACK header compression** for HTTP/2 (Cloudflare/kn007). Upstream nginx
only does static-table + Huffman; this adds dynamic-table encoding for response
headers, cutting header bytes on multiplexed connections. Adds
`http2_max_header_size` / `http2_max_field_size` tuning.
Origin: <https://github.com/kn007/patch>.

### 4. [`nginx-ssl_cert_cb_yield.patch`](patches/nginx-ssl_cert_cb_yield.patch)
OpenResty patch making `SSL_CTX_set_cert_cb()` callbacks **yield** for
non-blocking I/O. Lets 3rd-party modules (e.g. `ngx_lua` / lua-resty) serve
certificates and private keys dynamically and lazily — fetch a cert from an
external store mid-handshake without blocking the worker. Needed for
on-the-fly / per-SNI certificate logic.
Author: Yichun Zhang (agentzh), OpenResty.

### 5. [`ssl-cert-compression.patch`](patches/ssl-cert-compression.patch)
**TLS certificate compression** (RFC 8879) with algorithm selection. OpenSSL
3.2+/4.0 implements brotli/zlib/zstd cert compression internally and prefers
brotli automatically; this patch adds the companion directive
`ssl_certificate_compression_algorithms brotli zlib zstd;` (http/stream/mail)
to restrict/reorder the set via `SSL_CTX_set1_cert_comp_preference()`. The
on/off `ssl_certificate_compression` directive is upstream; this adds the
control knob. Needs `openssl-nginx` built with `enable-brotli`/`enable-zstd`
(see [`../openssl/`](../openssl/)). BoringSSL/AWS-LC zlib-callback path left
intact. Compresses the cert chain once at startup (cached) — no per-handshake
cost.

### 6. [`ssl-fingerprint.patch`](patches/ssl-fingerprint.patch)
**JA3/JA4 TLS fingerprinting** — captures the raw ClientHello in nginx core and
exposes `fp_*` fields on `ngx_ssl_connection_s` for the
`libnginx-mod-ssl-fingerprint` dynamic module to compute JA3/JA4. All additions
guarded by `NGX_SSL_FINGERPRINT` (set by the module's `config`), so the patch
is inert unless that module is built. Must apply **after**
`nginx_dynamic_tls_records.patch` (which reorders the connection struct).
Needs the matching openssl-nginx accessor — see
[`../openssl/patches/openssl-4.0.0-ssl-fingerprint.patch`](../openssl/patches/openssl-4.0.0-ssl-fingerprint.patch).
Origin: <https://github.com/HanadaLee/ngx_ssl_fingerprint_module>.

## Applying

```sh
cd nginx-<version>/
for p in 1.30.0-zlib-ng nginx_dynamic_tls_records nginx_hpack \
         nginx-ssl_cert_cb_yield ssl-cert-compression ssl-fingerprint; do
    patch -F0 -p1 < /path/to/nginx/patches/$p.patch || { echo "FAIL $p"; break; }
done
```

`patch -F0` (zero fuzz) is deliberate: the Debian packaging uses `dpkg-source`,
which also runs `patch -F0`, so any fuzz here would fail the package build.

## See also

- [snippets/](snippets/) — hardening config snippets for real vhosts
- [../openssl/README.md](../openssl/README.md) — the openssl-nginx patches these depend on
- [../README.md](../README.md) — repo overview
- [deb.myguard.nl](https://deb.myguard.nl/) · [nginx modules](https://deb.myguard.nl/nginx-modules/) · Discord [discord.gg/UQNsFg2y](https://discord.gg/UQNsFg2y)
