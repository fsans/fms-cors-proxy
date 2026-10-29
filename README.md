# fms-cors-proxy

Practical guides for the classic FileMaker Server problem: **a Web
Viewer page needs to `fetch()` FMS APIs (OData, Data API, WebDirect),
but the page and the API end up on different origins** — and the
browser enforces CORS / TLS trust.

There are three proven ways out. Two eliminate the cross-origin problem
entirely; one embraces it.

## The three options

| | A — LE cert on FMS | B — nginx reverse proxy | C — CORS on FMS |
| --- | --- | --- | --- |
| Resulting origins | Same origin (web app hosted **on FMS**) | Same origin (web app + proxy path on your web server) | Cross origin (web app hosted elsewhere) |
| Cert work on FMS | Full: request + install + renewal | None — FMS keeps self-signed | None |
| Extra moving part | DNS-01 renewal pipeline | One nginx `location /fmi/` | Apache `<Location>` edits on FMS |
| Client cert trust | Valid LE cert everywhere | Valid LE cert on the proxy | Needs an already-trusted cert on FMS |
| Best when | FMS should be the single endpoint | You already run a primary web server with a valid cert | Web app must live on another host |
| Guide | **[fms-letsencrypt-dns01](https://github.com/fsans/fms-letsencrypt-dns01)** (sister repo) | [`docs/reverse-proxy.md`](docs/reverse-proxy.md) | [`docs/cors.md`](docs/cors.md) |

**Rule of thumb**: prefer a same-origin architecture (A or B) — it
removes CORS, cookie, and `fetch()` trust issues entirely. Use CORS (C)
only when the web app genuinely cannot be co-hosted, and only for
trusted LAN origins.

Note that option A (this org's production choice) is documented in the
sister repo **[fsans/fms-letsencrypt-dns01](https://github.com/fsans/fms-letsencrypt-dns01)**
because it is a whole cert-pipeline topic of its own.

## Contents

| Path | What |
| ---- | ---- |
| `docs/reverse-proxy.md` | Full guide: nginx on your primary web server proxies `/fmi/` to internal FMS (`proxy_ssl_verify off`) |
| `docs/cors.md` | Full guide: CORS headers + OPTIONS handling in FMS's own Apache config |
| `config/nginx-fms-proxy.conf` | nginx server-block template (web app + `/fmi/` proxy) |
| `config/apache-cors-location.conf` | The `<Location "/fmi">` CORS block template for FMS's httpd.conf |

## Requirements

- FileMaker Server on macOS (Apache paths are macOS-specific; concepts
  transfer to other platforms).
- Option B: an existing web server with ports 80/443, a valid cert, and
  nginx (Apache `mod_proxy` works too, config not included).
- Option C: `sudo` on the FMS machine to edit
  `/Library/FileMaker Server/HTTPServer/conf/httpd.conf{,.2.4}`.
- OData/`fmodata` privileges configured on the database if you proxy or
  CORS-enable OData endpoints.
