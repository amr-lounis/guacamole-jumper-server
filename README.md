# JUMPER Guacamole

Apache Guacamole (browser-based remote desktop gateway) deployed with Docker Compose: RDP/SSH/VNC access over HTTPS behind Nginx, with a custom theme.

## Components

| Service | Image | Port / Role |
|---|---|---|
| `guacd` | `guacamole/guacd:1.6.0` | Protocol proxy (internal 4822), writes session recordings |
| `postgres` | `postgres:16-alpine` | Database `guacamole_db` (initialized from `./init/initdb.sql`) |
| `guacamole` | `guacamole/guacamole:1.6.0` | Web app (internal 8080, `WEBAPP_CONTEXT=/`) |
| `nginx` | `nginx:1.27-alpine` | Gateway: 80 redirects to 443, proxies to guacamole |

Key files:

```
docker-compose.yml          # Service definitions
.env                        # Secrets (never committed — copy from .env.example)
nginx/conf/nginx.conf       # Proxy + WebSocket + TLS settings
nginx/ssl/                  # Certificate and key (never committed)
init/initdb.sql             # DB schema + default guacadmin user
extensions/guacamole-theme/ # Theme source | the .jar is what actually loads
recordings/                 # Session recordings (written by guacd)
saml/                       # Place metadata.xml here when enabling SAML
```

## Quick Start

```powershell
# 1) Secrets
Copy-Item .env.example .env
# Edit .env and set a strong POSTGRES_PASSWORD

# 2) TLS certificate (dev only — use a real CA cert in production)
mkdir -p ./nginx/ssl
openssl req -x509 -nodes -days 365 -newkey rsa:2048 `
  -keyout ./nginx/ssl/cert.key `
  -out ./nginx/ssl/cert.crt `
  -subj "/CN=192.168.100.10" `
  -addext "subjectAltName=IP:192.168.100.10"

# 3) Recordings dir (auto-created, but better to create upfront)
mkdir -p ./recordings

# 4) Start
docker compose up -d
docker compose logs -f
```

Open `https://<server-ip>/` and log in:

```
Username: guacadmin
Password: guacadmin
```

> ⚠️ Change the password immediately under Settings → Users, and set a real `POSTGRES_PASSWORD` before any production use.

## Configuration

### Variables (`.env`)

| Variable | Description |
|---|---|
| `POSTGRES_PASSWORD` | Database password (required) |
| `SAML_ENTITY_ID` | Only needed when enabling SAML |
| `SAML_CALLBACK_URL` | Callback URL, e.g. `https://your-domain/` |

### Enabling SAML

1. Place the IdP file at `./saml/metadata.xml`.
2. In `docker-compose.yml`, uncomment the `SAML_*` env vars and the commented `metadata.xml` volume line.
3. Run `docker compose up -d` and try the single sign-on button on the login page.

### Custom Theme

The source lives in `extensions/guacamole-theme/` (CSS/images/`JUMPER` translation). After any change, **rebuild the JAR** so that `guac-manifest.json` ends up at the archive root (not inside a folder):

```powershell
Set-Location extensions\guacamole-theme
jar -cf ../guacamole-theme.jar *
```

PowerShell alternative without the Java `jar` tool (keeps `/` separators Java expects):

```powershell
Add-Type -AssemblyName System.IO.Compression
$src = "extensions\guacamole-theme"
$dst = "extensions\guacamole-theme.jar"
Remove-Item $dst -Force -ErrorAction SilentlyContinue
$fs = [System.IO.File]::Open($dst, [System.IO.FileMode]::Create)
$zip = New-Object System.IO.Compression.ZipArchive($fs, [System.IO.Compression.ZipArchiveMode]::Create)
Get-ChildItem -LiteralPath $src -Recurse -File | ForEach-Object {
  $rel = $_.FullName.Substring((Get-Item $src).FullName.Length + 1) -replace '\\','/'
  $entry = $zip.CreateEntry($rel, [System.IO.Compression.CompressionLevel]::Optimal)
  $in = [System.IO.File]::OpenRead($_.FullName)
  $out = $entry.Open()
  $in.CopyTo($out); $out.Close(); $in.Close()
}
$zip.Dispose(); $fs.Close()
```

Then `docker compose restart guacamole`.

### Recordings

Recording is configured **per connection** in the Guacamole UI (Recording path/name), not via environment variables. Suggested values from the original setup:

```
Recording path: ${HISTORY_PATH}/${HISTORY_UUID}
Recording name: ${GUAC_DATE}-${GUAC_TIME}
Include key events: true
Automatically create recording path: true
```

![Screen Recording settings](readme/Screen%20Recording.png)

`guacd` writes to `./recordings`; `guacamole` reads from it for playback.

## Backup

```powershell
docker compose exec postgres pg_dump -U guacamole_user guacamole_db > backup.sql
```

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| `502` right after start | Normal — nginx is up before guacamole; wait a few seconds and refresh |
| `POSTGRES_PASSWORD must be set` | Missing `.env` — copy it from `.env.example` |
| Theme does not appear | JAR must contain `guac-manifest.json` at its root |
| Mount error for `saml/metadata.xml` | File missing while SAML is still disabled — keep the line commented |
| No recordings | Make sure `./recordings` exists and Recording is configured per connection |

## Security Notes

- Never commit `.env` or `nginx/ssl/*.key` (covered by `.gitignore`).
- The current certificate is self-signed for development only.
- Change the default `guacadmin` account and use strong passwords.
