# CORS configuration for FileMaker Server OData

## Problem

The web viewer page is served from `http://192.168.x.x:8080` (a LAN
machine running nginx), while the OData API lives at
`https://fms.example.com/fmi/odata/v4/`. This is a **cross-origin**
request (different host, different protocol, different port), so the
browser enforces CORS.

> **Before doing this**: consider whether you even need CORS. Hosting
> the web app on FMS itself (same origin — see the
> [fms-letsencrypt-dns01](https://github.com/fsans/fms-letsencrypt-dns01)
> sister repo) or behind a reverse proxy (`docs/reverse-proxy.md`)
> eliminates the problem entirely. CORS is the right answer only when
> the web app must live on a different origin.

Symptoms in the web viewer console:

```text
Preflight response is not successful. Status code: 405
Fetch API cannot load https://fms.example.com/fmi/odata/v4/... due to access control checks.
```

Root causes on the FileMaker Server Apache config:

1. No `Access-Control-Allow-Origin` header was being sent for OData
   requests.
2. The only CORS header present pointed at the server's internal
   hostname (e.g. `SERVERNAME.local`), not the browser's origin.
3. Apache returned `405 Method Not Allowed` for `OPTIONS` preflight
   requests.

## Solution

Add a CORS policy inside the `<Location "/fmi">` block in FileMaker
Server's Apache config that:

- Reflects the browser's `Origin` header back as
  `Access-Control-Allow-Origin` (only for trusted LAN origins).
- Allows the request headers and methods used by `fms-odata-js`.
- Returns `200` for `OPTIONS` preflight requests.
- Removes the old global CORS headers that conflicted with the new
  policy.

## Which config file to edit

FileMaker Server on macOS keeps two config files in
`/Library/FileMaker Server/HTTPServer/conf/`:

| File | Role |
| ---- | ---- |
| `httpd.conf.2.4` | Source config. Edited by the admin. |
| `httpd.conf` | The file FMS's Apache **actually loads** when HTTP/2 is **off** (the default). |

The `Enable_HTTP2.sh` script (in `/Library/FileMaker Server/HTTPServer/`)
determines which file is loaded:

- **HTTP/2 off (default)**: `httpdctl` runs `patch_mod_mpm`, which takes
  `httpd.conf.2.4`, patches the MPM module lines, and writes the result
  as `httpd.conf`. FMS loads `httpd.conf`.
- **HTTP/2 on** (after running the script): `patch_mod_mpm` is
  commented out, so `httpd.conf.2.4` is loaded directly.

Check which protocol is active:

```bash
curl -I -k --http2 https://localhost/fmi/webd 2>/dev/null | grep HTTP
```

- `HTTP/2` → edit `httpd.conf.2.4` (loaded directly).
- `HTTP/1.1` → edit `httpd.conf` (the generated file FMS loads). Also
  apply the same edits to `httpd.conf.2.4` so they stay in sync.

> **Note**: In practice, `httpd.conf` is **not** always regenerated
> from `httpd.conf.2.4` on every restart. Treat both as standalone
> files and keep them in sync manually.

## Edits

### Edit 1 — Replace the `<Location "/fmi">` block

Find the existing `<Location "/fmi">` block and replace its contents
with (also in `config/apache-cors-location.conf`):

```apache
<Location "/fmi">
    Header set X-Content-Type-Options nosniff

    # Set cookie expiration date (12 hours in seconds), path and httponly
    Header edit Set-Cookie ^(.*(JSESSIONID).*)$ $1;Max-Age=43200

    # CORS configuration for 192.168.x.* range (with optional port)
    # — adjust the subnet to your LAN:
    RewriteEngine On
    RewriteCond %{HTTP:Origin} ^http://192\.168\.x\.[0-9]{1,3}(:[0-9]+)?$ [NC]
    RewriteRule ^ - [E=ALLOW_ORIGIN:%{HTTP:Origin}]

    Header always set Access-Control-Allow-Origin "%{ALLOW_ORIGIN}e" env=ALLOW_ORIGIN
    Header always set Access-Control-Allow-Headers "Origin, X-Requested-With, Content-Type, Accept, Authorization, odata-maxversion, odata-version, Prefer, OData-MaxVersion, OData-Version"
    Header always set Access-Control-Allow-Methods "PATCH, GET, POST, DELETE, OPTIONS"
    Header always set Access-Control-Allow-Credentials "true"

    RewriteCond %{REQUEST_METHOD} OPTIONS
    RewriteRule ^ - [R=200,L]

</Location>
```

Key details:

- The `RewriteCond` regex **must allow the port** (`(:[0-9]+)?`). The
  browser sends the Origin with the port (e.g.
  `http://192.168.x.x:8080`). Without this, the env var is never set
  and `Access-Control-Allow-Origin` is omitted.
- `Access-Control-Allow-Origin` is a **response** header and must
  **not** appear in `Access-Control-Allow-Headers` (it is not a
  request header).
- `Accept` must be in `Access-Control-Allow-Headers` because
  `fms-odata-js` sends an `Accept` header.
- `fms-odata-js` also sends OData-specific headers (`odata-maxversion`,
  `odata-version`, `Prefer`, and capitalized variants). All of these
  must be in `Access-Control-Allow-Headers` or the browser rejects the
  preflight. Alternatively, use `"*"` for Allow-Headers (safe even
  with credentials, unlike Allow-Origin).
- `RewriteRule ^ - [R=200,L]` returns `200` with an empty body for
  `OPTIONS`. Do **not** use `RewriteRule ^(.*)$ $1 [R=200,L]` — it
  produces a malformed response.

### Edit 2 — Remove the old global CORS headers

Find these lines (outside the `<Location>` block, further down in the
file) and **delete or comment them out**:

```apache
# Add headers for Cross-Origin Resource Sharing (CORS) policies
Header set Access-Control-Allow-Origin ${SERVER_NAME}
Header set Access-Control-Allow-Headers "Content-Type, Authorization"
Header set Access-Control-Allow-Credentials "true"
```

These set `Access-Control-Allow-Origin` to `${SERVER_NAME}` (the
server's internal hostname), which conflicts with the per-origin
policy in the `<Location>` block.

### Edit 3 (optional) — Suppress the startup warning

Find:

```apache
ServerName "${SERVER_NAME}"
```

Replace with:

```apache
ServerName fms.example.com
```

This silences the `AH00558: Could not reliably determine the server's
fully qualified domain name` warning on startup.

## Apply and reload

`fmsadmin restart wpe` only restarts the Web Publishing Engine, **not**
Apache. To reload Apache's config — including the cert and all
`httpd.conf` changes — use a graceful reload of the dedicated httpd
launchd job:

```bash
# Validate config syntax first:
'/Library/FileMaker Server/HTTPServer/bin/httpdctl' configtest 2>/dev/null \
  || /usr/sbin/httpd -t -f '/Library/FileMaker Server/HTTPServer/conf/httpd.conf'

# Graceful reload of FMS's Apache — rereads httpd.conf, no dropped
# connections, and does NOT bounce the database server:
sudo launchctl kickstart -k system/com.filemaker.httpd.graceful
```

> **Do not** use `launchctl stop` + `launchctl start` on
> `com.filemaker.fms` for Apache config changes: the DB server is a
> separate launchd job, `start` has been observed to silently no-op
> leaving FMS down, and restarting the whole stack is unnecessary —
> Apache config lives behind the `com.filemaker.httpd.*` jobs and a
> graceful reload is enough.

## Verify

```bash
curl -ksS -X OPTIONS \
  -H "Origin: http://192.168.x.x:8080" \
  -H "Access-Control-Request-Method: GET" \
  -H "Access-Control-Request-Headers: authorization" \
  -D - -o /dev/null \
  "https://fms.example.com/fmi/odata/v4/YOURDB/Contacts"
```

Expected response headers:

```text
HTTP/1.1 200 OK
Access-Control-Allow-Origin: http://192.168.x.x:8080
Access-Control-Allow-Headers: Origin, X-Requested-With, Content-Type, Accept, Authorization, odata-maxversion, odata-version, Prefer, OData-MaxVersion, OData-Version
Access-Control-Allow-Methods: PATCH, GET, POST, DELETE, OPTIONS
Access-Control-Allow-Credentials: true
```

If `Access-Control-Allow-Origin` is missing or Allow-Headers still
shows the old value, Apache didn't reload — run the `kickstart`
above, not `fmsadmin restart wpe`.

## Troubleshooting

### Edits not taking effect

1. Confirm the edits are in the file FMS actually loads (see "Which
   config file to edit" above).
2. Use `launchctl kickstart -k system/com.filemaker.httpd.graceful`,
   **not** `fmsadmin restart wpe`.
3. Check for overriding `Include` files:

   ```bash
   grep -rn "Access-Control-Allow" '/Library/FileMaker Server/HTTPServer/conf/extra/'
   ```

### Still getting 405 on preflight

The `OPTIONS` RewriteRule isn't firing. Verify `mod_rewrite` is loaded
(`LoadModule rewrite_module`) and that `RewriteEngine On` is inside
the `<Location>` block.

### `Access-Control-Allow-Origin` header missing

The `ALLOW_ORIGIN` env var isn't being set. Check that the
`RewriteCond` regex matches the actual Origin header the browser sends
(including the port). For a quick test, hardcode the origin instead of
using the env var:

```apache
Header always set Access-Control-Allow-Origin "http://192.168.x.x:8080"
```

## Security notes

- The `RewriteCond` allows **any** `192.168.x.x` origin. This is fine
  for a trusted LAN. To lock it down, replace the regex with the
  specific IP(s).
- `Access-Control-Allow-Credentials: true` combined with a reflected
  origin means the browser sends cookies/auth with cross-origin
  requests. This is required for OData Basic auth but must never be
  combined with `Access-Control-Allow-Origin: *`.
- FMS may overwrite `httpd.conf` on a future update/reinstall. Reapply
  the edits if that happens, and keep `httpd.conf.2.4` in sync as a
  backup.
