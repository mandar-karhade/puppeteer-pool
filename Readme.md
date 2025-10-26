## Puppeteer Pool (Browserless) - n8n-ready

This repo contains a minimal Docker setup for running Browserless (headless Chromium) as a service that n8n and other clients can call for page content and structured scraping.

### What you get
- **Browserless Chromium** exposed on `:3000`
- **Token auth** via `TOKEN`
- **Prebooted Chrome** and sensible concurrency/timeouts
- Optional **default User-Agent** (recommended to avoid 400s when clients send malformed UA)

### Quick start
```bash
docker compose up -d
docker compose logs -f browserless

# Smoke test (replace TOKEN)
curl -sS -X POST "http://127.0.0.1:3000/content?token=TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"url":"https://www.google.com"}' | head -n 20
```

### Configuration (docker-compose.yml)
Environment variables on the `browserless` service:

- `PREBOOT_CHROME` (default: `true`)
- `MAX_CONCURRENT_SESSIONS` (e.g., `4`)
- `CONNECTION_TIMEOUT` (ms)
- `TOKEN` (required if you expose the port)
- `DEFAULT_USER_AGENT` (optional but recommended)

Example snippet:
```yaml
environment:
  - PREBOOT_CHROME=true
  - MAX_CONCURRENT_SESSIONS=4
  - CONNECTION_TIMEOUT=60000
  - TOKEN=your-strong-token
  - DEFAULT_USER_AGENT=Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124 Safari/537.36
```

Note: If you enable a healthcheck or restart policy, apply them under the same service in `docker-compose.yml`.

### n8n integration
Use either the Browserless node (ensure it doesn’t send malformed fields), or the HTTP Request node.

- **Base URL**: `http://<host>:3000`
- **Token**: set the node’s API Key/Token or add `?token=...`
- **URL**: always use `https://...` for target sites
- Avoid sending invalid shapes. Do not leave fields that cause bad JSON (no silent fallbacks).

Working HTTP Request examples:

Content endpoint:
```http
POST http://<host>:3000/content?token=TOKEN
Content-Type: application/json

{ "url": "https://www.fda.gov/..." }
```

Scrape endpoint (do not include unsupported element fields like `type`):
```http
POST http://<host>:3000/scrape?token=TOKEN
Content-Type: application/json

{
  "url": "https://www.fda.gov/...",
  "gotoOptions": { "waitUntil": "networkidle2" },
  "elements": [ { "selector": "article" } ]
}
```

If you must specify a User-Agent, send the correct object shape (not a bare string):
```json
{
  "userAgent": {
    "userAgent": "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124 Safari/537.36",
    "platform": "Linux x86_64"
  }
}
```

Cookies must be an array if provided:
```json
{ "cookies": [] }
```

### Troubleshooting
- 400 with message like "userAgent must be of type object": your client sent a string/empty string; use the object shape above or rely on `DEFAULT_USER_AGENT`.
- 400 with "elements[0].type is not allowed": remove `type`—only `selector` is required.
- Connection refused from n8n: if n8n is in Docker, use the host/LAN IP or a shared network; `localhost` inside a container is not your host.


