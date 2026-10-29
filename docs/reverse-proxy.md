# Reverse-proxy deployment: nginx in front of FileMaker Server

## Problem

FileMaker Server requires HTTPS for OData (it does not serve OData over
plain HTTP). On a private network, getting a trusted SSL certificate
onto FMS directly is hard because:

- The FMS machine is not reachable from the internet on port 80, so the
  standard Let's Encrypt **HTTP-01** challenge does not work.
- Ports 80 and 443 on the public-facing router are reserved for another
  web server, so they cannot be forwarded to FMS.
- A self-signed cert installed directly on FMS would require manual
  trust installation in Keychain on every client Mac, and breaks
  `fetch()` inside the Web Viewer unless the cert is trusted at the OS
  level.

## Solution

Put an **nginx reverse proxy** on your primary web server (the machine
that already has ports 80/443 and a valid Let's Encrypt cert). The
proxy:

1. **Serves the web app** (`index.html` + JS files) directly from its
   own web root.
2. **Reverse-proxies `/fmi/`** to the internal FMS machine.

The browser (and `fetch()`) only ever talks to the proxy — it sees the
valid LE cert and a single origin. The proxy talks to FMS internally
over HTTPS and **ignores FMS's self-signed cert** via
`proxy_ssl_verify off`. FMS keeps its default self-signed cert; no cert
installation is needed on FMS itself.

```text
Browser (Web Viewer)
    │
    │  HTTPS — valid LE cert on fms.example.com
    ▼
nginx (primary web server, ports 80/443, LE cert)
    │
    │  HTTPS — FMS self-signed cert (nginx ignores it)
    ▼
FileMaker Server (internal IP, default self-signed cert)
```

This gives you:

- **Same origin** (web app + OData both on `https://fms.example.com`) →
  no CORS configuration needed.
- **Valid LE cert** on the only endpoint the browser sees → no
  client-side cert installation, no `fetch()` trust errors on any
  client (FileMaker Pro, Go, WebDirect).
- **No cert work on FMS** — it keeps its default self-signed cert. The
  proxy handles trust.
- **No port 80/443 forwarding** to FMS required — FMS stays internal.

> **Key insight**: FMS still serves OData over HTTPS (as it requires),
> but only the proxy talks to it. The proxy is told to ignore the
> self-signed cert. The browser never sees FMS's cert — it only sees
> the LE cert on the proxy.

## Prerequisites

- A **primary web server** that:
  - Has ports 80 and 443 open to the network (LAN or WAN).
  - Already has a valid Let's Encrypt (or other CA) certificate for a
    domain (e.g. `fms.example.com`).
  - Runs **nginx** (or Apache — this guide uses nginx; the Apache
    equivalent is `mod_proxy` with `SSLProxyCheckPeerCN off`).
- A **FileMaker Server** on the internal network (e.g. `192.168.x.x`)
  with:
  - OData enabled (`fmsadmin start odata` or Admin Console → Connectors
    → OData → Enabled).
  - The `fmodata` extended privilege enabled in the database's
    privilege set (File → Manage → Security → Extended Privileges →
    Access via OData).
  - Its default self-signed cert (no custom cert needed on FMS).
- The primary web server can reach FMS on port 443 internally
  (`https://192.168.x.x/fmi/...`).

## Step 1 — Deploy the web app on the primary web server

Copy the runtime files into a subfolder of the web root:

```bash
# On the primary web server (nginx):
sudo mkdir -p /var/www/my-web-app
sudo cp index.html onfmready.js fms-odata.esm.min.js /var/www/my-web-app/
```

The web viewer URL will be:

```text
https://fms.example.com/my-web-app/index.html
```

## Step 2 — Add the nginx reverse-proxy config

Edit the nginx server block for `fms.example.com` (typically in
`/etc/nginx/sites-available/fms.example.com` or `/etc/nginx/conf.d/`).
Add two `location` blocks inside the existing `server { ... }` (the one
with the LE cert). See `config/nginx-fms-proxy.conf` for a ready
template:

```nginx
server {
    listen 443 ssl http2;
    server_name fms.example.com;

    # Your existing LE cert config (already there):
    ssl_certificate     /etc/letsencrypt/live/fms.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/fms.example.com/privkey.pem;

    # ... your existing locations ...

    # --- Serve the web app ---
    location /my-web-app/ {
        alias /var/www/my-web-app/;
        try_files $uri $uri/ =404;
    }

    # --- Reverse-proxy /fmi/ to the internal FileMaker Server ---
    location /fmi/ {
        proxy_pass https://192.168.x.x/fmi/;   # FMS internal IP
        proxy_ssl_verify off;                   # FMS uses self-signed cert; proxy ignores it
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # OData responses can be large; bump the buffer/proxy sizes if needed.
        proxy_buffers 8 16k;
        proxy_buffer_size 16k;
        proxy_read_timeout 60s;
    }
}
```

Key directives:

| Directive | Why |
| --------- | --- |
| `proxy_pass https://192.168.x.x/fmi/` | Forward all `/fmi/` traffic (OData, WebDirect, Admin Console) to FMS internally. |
| `proxy_ssl_verify off` | **The critical line.** nginx accepts FMS's self-signed cert without validating it against a CA. Without this, nginx rejects the connection with an SSL error. |
| `proxy_set_header Host $host` | Pass the original Host header so FMS sees the requested domain (some FMS endpoints are sensitive to this). |
| `alias /var/www/my-web-app/` | Serve the web app files from the local web root (not proxied). |

> **Trailing slashes matter**: `proxy_pass https://192.168.x.x/fmi/;`
> with the trailing `/fmi/` on both the `location` and the `proxy_pass`
> means nginx strips the matched prefix and forwards the rest. So a
> request to `https://fms.example.com/fmi/odata/v4/YOURDB/Contacts`
> becomes `https://192.168.x.x/fmi/odata/v4/YOURDB/Contacts` on FMS.
> Keep the slashes consistent or the path gets mangled.

## Step 3 — Reload nginx

```bash
# Test config syntax first:
sudo nginx -t

# Reload (no downtime):
sudo nginx -s reload
```

## Step 4 — Verify the proxy works

From any machine that can reach `fms.example.com`:

```bash
# 1. The web app is served (should return the HTML):
curl -sS https://fms.example.com/my-web-app/index.html | head -5

# 2. OData is proxied (should return JSON, no -k needed — LE cert is valid):
curl -sS -u <odata-user>:<odata-password> \
  "https://fms.example.com/fmi/odata/v4/YOURDB/Contacts" | head -c 200
```

If the OData call works **without** `-k`, the proxy is correctly
terminating SSL with the LE cert and forwarding to FMS internally.

### If the OData call fails

| Symptom | Cause | Fix |
| ------- | ----- | --- |
| `502 Bad Gateway` | nginx can't reach FMS on `https://192.168.x.x` | Check FMS is running, check the internal IP, check firewall allows 443 from the proxy to FMS. |
| `502 Bad Gateway` + `SSL handshake failed` in nginx logs | `proxy_ssl_verify off` is missing or nginx version is very old | Confirm the directive is in the `location /fmi/` block. |
| `404 Not Found` | Path mismatch — trailing slashes wrong | Ensure both `location /fmi/` and `proxy_pass .../fmi/` have trailing slashes. |
| `403 Forbidden` | OData listener running but privilege set lacks `fmodata` | In FileMaker Pro: File → Manage → Security → enable "Access via OData (fmodata)". |
| `502.3 Bad Gateway` | OData listener not running on FMS | `sudo fmsadmin start odata` on FMS. |

## Step 5 — Point the app at the proxy domain

Whatever configures the web app's API base URL (for this org's apps,
the `initialize` bridge payload's `odata.host`) must use the **proxy
domain**, not the internal FMS IP:

```json
{ "odata": { "host": "https://fms.example.com", "...": "..." } }
```

The web app's `fetch()` calls then hit
`https://fms.example.com/fmi/odata/v4/...` — same origin as the page →
no CORS.

## Step 6 — Remove CORS config on FMS (if previously added)

Because the web app and OData are now on the **same origin**, the
browser does not enforce CORS. If you previously added CORS headers to
FMS's Apache config (see `docs/cors.md`), you can remove them — they're
no longer needed and could even conflict.

## Architecture summary

```text
┌─────────────────────────────────────────────────────────────┐
│  FileMaker Pro (client Mac)                                  │
│    └─ Web Viewer → https://fms.example.com/my-web-app/      │
│         │                                                    │
│         │ fetch() calls https://fms.example.com/fmi/odata/… │
└─────────┼────────────────────────────────────────────────────┘
          │  (valid LE cert — no -k, no Keychain trust needed)
          ▼
┌─────────────────────────────────────────────────────────────┐
│  Primary web server (fms.example.com, ports 80/443, LE cert) │
│    nginx                                                     │
│      location /my-web-app/  →  /var/www/my-web-app/         │
│      location /fmi/         →  proxy_pass https://192.168…  │
│                                  proxy_ssl_verify off        │
└─────────┬────────────────────────────────────────────────────┘
          │  (HTTPS with FMS self-signed cert — proxy ignores it)
          ▼
┌─────────────────────────────────────────────────────────────┐
│  FileMaker Server (192.168.x.x, internal, self-signed cert)  │
│    OData listener on 443                                     │
└─────────────────────────────────────────────────────────────┘
```

## Comparison with the DNS-01 option

| | Reverse proxy (this doc) | DNS-01 LE cert on FMS ([sister repo](https://github.com/fsans/fms-letsencrypt-dns01)) |
| --- | --- | --- |
| Cert on FMS | Default self-signed (no work) | Valid LE cert (request + install + renew) |
| CORS needed? | No (same origin via proxy) | No (same origin on FMS) |
| Client cert trust? | None (LE cert via proxy) | None (LE cert on FMS) |
| Renewal | LE cert on proxy (already automated) | LE cert on FMS (launchd + deploy hook) |
| FMS reachable from clients? | No (only proxy talks to FMS) | Yes (clients hit FMS directly) |
| FM client TLS on 5003? | Still self-signed — FM Pro clients connecting on 5003 see the untrusted cert | Covered — port 5003 gets the LE cert too |
| Extra moving part | nginx proxy config | DNS TXT record + renewal pipeline |
| Best when | You already have a primary web server with LE | You want FMS to be the single endpoint (and green locks on FM client connections) |

**Use the reverse proxy** if you already have a primary web server with
an LE cert. It's the least work — no cert management on FMS at all.

**Use DNS-01** if you want FMS to be the single endpoint, or if FM Pro
clients connecting on port 5003 need a trusted cert too (the proxy only
covers the web/443 side).

## Security notes

- `proxy_ssl_verify off` means nginx does not validate FMS's cert. This
  is safe **only because the proxy-to-FMS connection is on a trusted
  internal network**. Keep FMS behind the firewall.
- OData credentials (Basic auth) travel browser→proxy encrypted with
  the LE cert, then proxy→FMS encrypted with FMS's self-signed cert.
  Both legs are encrypted; the self-signed cert provides encryption,
  just not identity verification.
- Restrict the proxy's `/fmi/` location if you don't want all FMS web
  endpoints exposed. To expose only OData:

  ```nginx
  location /fmi/odata/ {
      proxy_pass https://192.168.x.x/fmi/odata/;
      proxy_ssl_verify off;
      # ... headers ...
  }
  ```

  Then `/fmi/webd`, `/fmi/admin-console`, etc. are not proxied.
