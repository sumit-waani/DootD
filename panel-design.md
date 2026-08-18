# Panel — Design Document

> A single-binary, self-hosted control panel for hosting 3–5 small PHP apps
> on a single VPS. PHP 8.5 + SQLite only. GitHub-based deploys. S3 backups.
> Cloudflare DNS. One SSH, one install command.

**Status:** Draft v1 — decisions locked, ready for implementation handoff.

---

## 0. Locked Decisions (TL;DR)

| # | Decision | Choice | Reason |
|---|----------|--------|--------|
| 1 | Language | Go 1.22+ | Static binary, easy single-file install, no runtime |
| 2 | Binary model | Single static binary, `embed.FS` for UI assets | "SSH once, install once" promise |
| 3 | Database (panel state) | SQLite via `modernc.org/sqlite` (pure Go, no CGO) | Matches user's "SQLite only" rule; no CGO = simple cross-compile |
| 4 | Web server / reverse proxy | **Caddy** (not our own host-header routing) | Auto HTTPS, on-demand TLS, hot reload via local API |
| 5 | PHP runtime | PHP 8.5 FPM (single service, multiple pools) | User requirement; one FPM service is much lighter than N services |
| 6 | Per-app FPM isolation | One pool per app, each running as its own system user | Sandbox; matches `open_basedir` model |
| 7 | Frontend | Server-rendered HTML + HTMX + Alpine.js + Tailwind (compiled, embedded) | No build step beyond `tailwindcss`; minimal JS |
| 8 | HTTP framework | `go-chi/chi` v5 | Lightweight, idiomatic, stdlib-compatible |
| 9 | Deployment | `git` over `os/exec`; optional `composer install` | No magic; transparent, debuggable |
| 10 | Auto-deploy | Signed webhook per app (`/api/hooks/deploy/:token`) | Simple, no public port for a separate worker |
| 11 | Backups | `rclone` invoked via `os/exec`, daily systemd timer | Battle-tested, supports all S3-compatible providers |
| 12 | Cloudflare | Official `cloudflare-go` SDK | Stable, well-maintained |
| 13 | Process supervision | systemd | Universal on supported distros |
| 14 | Supported distros (v1) | Ubuntu 22.04+, Debian 12+ | Sury provides PHP 8.5; Caddy has official repos |
| 15 | Dashboard port | `:8080` HTTP (apps take 80/443) | Apps and dashboard decouple; user can Cloudflare-Tunnel the dashboard |
| 16 | Encryption at rest | AES-256-GCM, key in `/etc/panel/key` (0600) | Sufficient for env vars + Cloudflare token |
| 17 | Auth | Session cookies (httpOnly, Secure, SameSite=Strict) + bcrypt | Simple, no external IdP needed for 1–2 users |
| 18 | Update channel | Self-update via GitHub releases (`go-update` library) | No package manager involvement |
| 19 | Logs | JSON-line structured logs to `/var/log/panel/`, plus tail in UI | Greppable + UI-friendly |
| 20 | Metrics in panel | `gopsutil` for CPU/RAM/disk; FPM status page scraped | No Prometheus dependency |

---

## 1. Goals & Non-Goals

### 1.1 Goals

1. **One SSH, one install command.** A user with a fresh VPS runs a single command
   and ends up with a working dashboard, all required system packages, and a
   default admin user.
2. **Host 3–5 small PHP apps per VPS.** Each app is a separate repo, separate
   FPM pool, separate system user.
3. **GitHub-based deploys.** Paste a repo URL, pick a branch, paste env vars,
   click Deploy. Optionally wire a webhook for push-to-deploy.
4. **SQLite only.** No MySQL/Postgres/Redis/Mailhog/etc. shipped or required.
5. **S3-compatible daily backups.** Push app sources + sqlite files + panel
   state to any S3-compatible endpoint (R2, Backblaze, MinIO, DO Spaces, etc.).
6. **Cloudflare DNS in the panel.** User pastes an API token, panel can create
   / update / delete records for app domains.
7. **Operational visibility.** System health (CPU, RAM, disk), per-app status
   (FPM workers, last deploy, uptime), log tailing, and backup history — all
   from the dashboard.

### 1.2 Non-Goals (v1)

- Multi-tenant hosting (one panel user is enough for personal projects).
- Horizontal scaling / multi-node clusters. Single VPS is the design point.
- Docker / containerization. FPM pools are the isolation primitive.
- Built-in mail server. Apps must use SMTP relays (Mailgun, SES, etc.).
- Built-in object storage. S3 is the answer; the panel itself doesn't store
  uploads.
- Built-in reverse proxy magic for non-PHP apps. Node/Go apps are out of scope.
- 2FA / SSO / SAML. Username + password + (optional) Cloudflare Access in
  front of the dashboard.
- Prometheus / Grafana integration. Panel has its own minimal health view.
- Database management UI beyond SQLite. Apps own their own `.sqlite` files;
  panel doesn't open a SQL console.
- Auto-scaling, blue/green deploys, traffic splitting. Each app runs as its
  current code, period.

---

## 2. Personas

| Persona | What they want | What they tolerate |
|---------|----------------|--------------------|
| **Solo dev (primary)** | Friction-free deploys, backups, DNS in one place | One-time setup pain, no fancy UI |
| **Dev's CI / GitHub** | A webhook URL to ping on push | — |
| **Cloudflare DNS** | An API token to act on the user's behalf | — |

There is exactly one human user in v1. The product is "private PAAS for me."

---

## 3. High-Level Architecture

```
                              Internet
                                 │
                  ┌──────────────┴──────────────┐
                  │                             │
       (HTTP/HTTPS :80/:443)             (HTTP :8080)
                  │                             │
                  ▼                             ▼
            ┌──────────┐                  ┌──────────┐
            │  Caddy   │                  │  Panel   │
            │  v2.x    │                  │  binary  │
            └────┬─────┘                  └────┬─────┘
                 │                             │
   Host: app1.x  │─────────────────┐   /api/*  │
   Host: app2.y  │────────────┐    │           │
                 │            │    │           │
                 ▼            ▼    ▼           ▼
       ┌──────────────┐ ┌──────────────┐  ┌──────────┐
       │ php8.5-fpm   │ │ php8.5-fpm   │  │ SQLite   │
       │ pool: app1   │ │ pool: app2   │  │ panel.db │
       │ unix socket  │ │ unix socket  │  └────┬─────┘
       │ user:        │ │ user:        │       │
       │  panel-app1  │ │  panel-app2  │       │
       └──────┬───────┘ └──────┬───────┘       │
              │                │               │
              ▼                ▼               ▼
       /var/lib/panel/apps/app1/{app, runtime, sqlite, backups}
       /var/lib/panel/apps/app2/{app, runtime, sqlite, backups}

   The Panel binary drives:
     - systemctl (caddy, php8.5-fpm)
     - Caddy admin API (localhost:2019)  — hot-reload vhosts
     - git                              — pull code
     - composer                         — install deps
     - rclone                           — push backups to S3
     - cloudflare-go SDK                — manage DNS records
     - A systemd timer (panel-backup.timer) for daily backups
```

**Why Caddy, not our own host-header routing**

| Concern | Caddy | Custom Go router |
|---------|-------|------------------|
| TLS (Let's Encrypt, renewals, OCSP) | Free | Pain |
| HTTP/2, HTTP/3 | Free | Pain |
| On-demand TLS for first-time hostnames | Built-in | DIY |
| Hot reload of vhost config | `POST /load` JSON | DIY |
| Resource cost | One extra process | Embedded but you re-implement the above |
| Operational risk | Battle-tested | New code |

We add one systemd unit (Caddy). We delete ~2,000 lines of proxy/TLS code we'd
otherwise have to write, test, and CVE-patch. Worth it.

---

## 4. System Layout (Filesystem)

```
/usr/local/bin/panel                 # the binary (installer + daemon + CLI)
/etc/panel/
  panel.yaml                         # top-level YAML config
  key                                # 32-byte AES key, 0600, owned by panel:panel
  caddy/
    Caddyfile                        # main, imports apps/*.caddy
    apps/
      app1.caddy                     # generated per app
      app2.caddy
  php/
    fpm/
      pool.d/
        panel-app1.conf              # FPM pool per app
        panel-app2.conf
      php.ini                        # shared FPM php.ini
/var/lib/panel/
  panel.db                           # panel's own SQLite
  apps/
    app1/
      app/                           # git checkout
      runtime/                       # writable: storage, sessions, uploads, tmp
      sqlite/                        # *.sqlite files
      backups/                       # rolling local snapshots before S3 push
      .env                           # decrypted env vars (0600)
      meta.json                      # panel-side metadata (port, socket, etc.)
    app2/...
/var/log/panel/
  panel.log                          # main daemon log (JSON lines)
  panel.err
  caddy/
    access.log
    error.log
  apps/
    app1/
      access.log                     # mirrored from Caddy
      error.log
      fpm.log
      deploy.log                     # deployment audit trail
/var/lib/rclone/                     # rclone config (encrypted at rest by rclone)
/etc/systemd/system/
  panel.service                      # the panel daemon
  panel-backup.timer                 # daily
  panel-backup.service               # One-shot, runs panel backup run
```

### 4.1 Per-app system user

On `app create`, the panel creates a system user `panel-app-<name>` (no login,
no shell, home = `/var/lib/panel/apps/<name>`) and chowns the app directory
to it. The FPM pool runs as this user. Caddy runs as `www-data` (or
`caddy`) but only talks to FPM via a unix socket owned by `panel-app-<name>`
and group `panel-app-<name>`, with mode `0660`.

---

## 5. Installer (`panel install`)

### 5.1 Invocation

The recommended path is a one-liner:

```bash
curl -sSL https://get.panel.example/install | sudo bash
```

This script:
1. Detects arch (amd64 / arm64), downloads the matching static binary from
   the latest GitHub release to `/usr/local/bin/panel`.
2. Verifies the SHA-256 against the published `checksums.txt`.
3. Runs `sudo panel install`.

Alternatively, the user can download the binary directly and run
`sudo ./panel install`. Same flow.

### 5.2 `panel install` behavior

```
checks:
  - running as root
  - systemd is PID 1
  - OS is in supported set (Ubuntu 22.04+, Debian 12+)
  - port 80, 443, 8080 are free

package manager actions:
  - add Sury repo (for PHP 8.5), Caddy official repo
  - apt update
  - apt install -y \
      caddy \
      php8.5 php8.5-fpm php8.5-cli \
      php8.5-sqlite3 php8.5-mbstring php8.5-xml \
      php8.5-curl php8.5-zip php8.5-intl php8.5-opcache \
      php8.5-bcmath \
      unzip git rclone sqlite3 ca-certificates

system actions:
  - create group 'panel' (system)
  - create user 'panel' (system, no login, home /var/lib/panel)
  - mkdir -p /etc/panel/{caddy/apps,php/fpm/pool.d}
  - mkdir -p /var/lib/panel/{apps,backups}
  - mkdir -p /var/log/panel/{caddy,apps}
  - mkdir -p /var/lib/rclone
  - if /etc/panel/key missing: generate 32 random bytes, write 0600 panel:panel

panel setup:
  - run migrations → /var/lib/panel/panel.db
  - create default user 'admin' with random 24-char password
  - write /etc/panel/panel.yaml from defaults

systemd:
  - write /etc/systemd/system/panel.service
  - write /etc/systemd/system/panel-backup.service
  - write /etc/systemd/system/panel-backup.timer (OnCalendar=daily, 03:00)
  - write /etc/caddy/Caddyfile (base + import)
  - systemctl daemon-reload
  - systemctl enable --now caddy php8.5-fpm panel panel-backup.timer

firewall hint (not enforced):
  - print: "Open port 80, 443, 8080 in your cloud firewall / ufw."

final:
  - print dashboard URL: http://<vps-ip>:8080
  - print admin username + one-time password
  - print: "Run 'panel user passwd admin' to change the password."
  - print: "Optionally put Cloudflare in front of :8080 for HTTPS + Access."
```

### 5.3 Idempotency

Re-running `panel install` is safe. It is a no-op if everything is already
in place, and it will:
- Skip already-installed packages.
- Not rotate the AES key (or warn loudly before doing so).
- Not rotate the admin password.
- Refresh systemd unit files to the latest template.
- Reload services only if their unit changed.

---

## 6. Per-App Lifecycle

### 6.1 Create

API: `POST /api/apps`

Request body:

```json
{
  "name": "app1",
  "repo_url": "git@github.com:me/app1.git",
  "branch": "main",
  "deploy_key": "-----BEGIN OPENSSH PRIVATE KEY-----\n...",
  "env": {
    "APP_ENV": "production",
    "APP_KEY": "base64:...",
    "DB_DATABASE": "/var/lib/panel/apps/app1/sqlite/database.sqlite"
  },
  "domain": "app1.example.com",
  "cloudflare_managed": true,
  "auto_deploy": true
}
```

Steps:

1. Validate name (lowercase, `[a-z0-9-]{3,32}`).
2. Acquire `flock(/var/lock/panel-app-create.lock)`.
3. `useradd -r -M -d /var/lib/panel/apps/app1 -s /usr/sbin/nologin panel-app-app1`.
4. `mkdir -p /var/lib/panel/apps/app1/{app,runtime/...,sqlite,backups}` and
   `chown -R panel-app-app1:panel-app-app1`.
5. `git clone --branch <branch> --depth 1 <repo_url> app/`.
   (If HTTPS: use token in URL or git credential helper. If SSH: use deploy key
   written to `/var/lib/panel/apps/app1/.ssh/id_ed25519`, 0600.)
6. Write `/var/lib/panel/apps/app1/.env` (decrypted from request), 0600.
7. If `composer.json` exists in app/: `composer install --no-dev --no-interaction
   --prefer-dist --no-progress`.
8. Auto-detect document root:
   - If `public/` exists: use that.
   - If `index.php` at root: use root.
   - Else: error out and let user override in app settings.
9. Write FPM pool config `/etc/panel/php/fpm/pool.d/panel-app1.conf`.
10. Write Caddy vhost `/etc/caddy/apps/app1.caddy`.
11. `systemctl reload php8.5-fpm` (graceful — picks up new pool).
12. `curl -X POST http://localhost:2019/load` (Caddy hot reload).
13. If `cloudflare_managed`: `POST /zones/:id/dns_records` with the A record.
14. Insert row in `apps` table.
15. Return app object.

### 6.2 FPM pool config template

`/etc/panel/php/fpm/pool.d/panel-app-<name>.conf`:

```ini
[panel-app-<name>]
user  = panel-app-<name>
group = panel-app-<name>

listen       = /run/php/panel-app-<name>.sock
listen.owner = www-data
listen.group = www-data
listen.mode  = 0660

pm                   = ondemand
pm.max_children      = 10
pm.process_idle_timeout = 60s
pm.max_requests      = 500

php_admin_value[upload_tmp_dir] = /var/lib/panel/apps/<name>/runtime/tmp
php_admin_value[open_basedir]   = /var/lib/panel/apps/<name>:/tmp
php_admin_flag[display_errors]   = off
php_admin_flag[log_errors]       = on
php_admin_value[error_log]       = /var/log/panel/apps/<name>/fpm.log

env[PATH] = /usr/local/bin:/usr/bin:/bin
```

### 6.3 Caddy vhost template

`/etc/caddy/apps/<name>.caddy`:

```
https://<domain> {
    encode gzip zstd
    root * /var/lib/panel/apps/<name>/app/<docroot>

    php_fastcgi unix//run/php/panel-app-<name>.sock

    file_server

    @blocked {
        path /vendor/*
        path /.git/*
        path /.env
        path /storage/*.sqlite
    }
    respond @blocked 404

    log {
        output file /var/log/panel/apps/<name>/access.log {
            roll_size 50mb
            roll_keep 5
        }
    }
}
```

`/etc/caddy/Caddyfile` (root, generated once by installer):

```
{
    admin localhost:2019
    auto_https on
    https_port 443
    http_port 80
    servers :80, :443
}
import /etc/caddy/apps/*.caddy
```

### 6.4 Deploy (manual or webhook)

API: `POST /api/apps/:id/deploy` (or `POST /api/hooks/deploy/:token` for
GitHub).

```
acquire per-app flock: /var/lock/panel-app-<name>.lock
open /var/log/panel/apps/<name>/deploy.log (append)
insert deployments row (status=running)
if env changed: rewrite /var/lib/panel/apps/<name>/.env
git -C /var/lib/panel/apps/<name>/app fetch --depth 1 origin <branch>
git -C /var/lib/panel/apps/<name>/app reset --hard origin/<branch>
if composer.json changed: composer install --no-dev --no-interaction
if artisan exists: php artisan migrate --force
systemctl reload php8.5-fpm     # graceful worker recycle
update deployments row (status=success, commit_sha, finished_at)
release flock
emit SSE event 'deploy.complete' to dashboard (if open)
```

Webhook security: each app gets a 32-byte random token. Webhook URL is
`/api/hooks/deploy/<token>`. Panel verifies `X-Hub-Signature-256` against
the GitHub HMAC if the user pasted a webhook secret, or accepts the
unguarded token URL if they didn't (printed with a warning).

### 6.5 Stop / Start / Restart

PHP-FPM pools support `pm = ondemand` so empty pools have zero workers. To
fully stop traffic to an app:

- Stop: set the app to `disabled` in DB; rewrite the Caddy vhost to
  `respond "App disabled" 503`; `curl -X POST localhost:2019/load`.
- Start: restore vhost; reload.
- Restart: `systemctl reload php8.5-fpm` (recycles workers).

There is no separate "stop" flag in FPM — we control traffic at the Caddy
layer, which is clean and instant.

### 6.6 Delete

1. Stop traffic (Caddy vhost → 503).
2. `rm /etc/caddy/apps/<name>.caddy`; reload Caddy.
3. `rm /etc/panel/php/fpm/pool.d/panel-app-<name>.conf`; reload FPM.
4. `userdel panel-app-<name>`.
5. Optionally archive app to `/var/lib/panel/backups/<name>-<ts>.tar.gz` and
   push to S3, then `rm -rf /var/lib/panel/apps/<name>`.
6. Delete DB rows.

---

## 7. Routing & HTTPS

### 7.1 Dashboard

Dashboard binds to `:8080` over plain HTTP. The user is expected to either:
- Hit it directly on the VPS IP for initial setup, or
- Front it with a Cloudflare Tunnel and put Cloudflare Access in front for
  SSO / IP allowlist, or
- Reverse-proxy it through Caddy with a special Host header (e.g.
  `panel.example.com` → `:8080`) and let Caddy handle TLS.

The panel supports a "trusted_proxies" config so the original client IP is
honored when behind Cloudflare. It also sets
`SetRemoteAddrFromHeader("CF-Connecting-IP")` when `trust_cloudflare: true`.

### 7.2 Apps

Apps live on `:80` and `:443`, served by Caddy. Caddy handles ACME via
Let's Encrypt or the Cloudflare DNS challenge. The panel does not touch
certificates at all.

### 7.3 Shared IP, multiple apps

Host-header routing is Caddy's bread and butter. One `:443` listener, many
vhosts. No port conflicts. No SNI gymnastics.

### 7.4 Wildcard domains (optional)

If the user wants `*.example.com` → dispatch by subdomain to different apps,
they add a single wildcard Caddy vhost that imports per-subdomain snippets.
Out of scope for v1; documented as a manual escape hatch.

---

## 8. Security Model

### 8.1 Threat model (in scope)

- Random internet scanner hitting the dashboard.
- Compromised app reading another app's files.
- Compromised app exfiltrating the panel's secrets.
- Stolen S3 credentials or Cloudflare token.
- Local privilege escalation from `www-data` to `root`.
- Log injection / log poisoning.

### 8.2 Threat model (out of scope)

- Nation-state APT with kernel exploits.
- Physical access to the VPS.
- Compromised Sury/Caddy/rclone upstream packages (we trust apt trust chain).

### 8.3 Defenses

| Threat | Defense |
|--------|---------|
| Random scanner on :8080 | Username + password, session cookies (httpOnly, Secure, SameSite=Strict), CSRF token, rate-limited login (5/min/IP) |
| Compromised app reading neighbors | Per-app unix socket, per-app FPM user, `open_basedir` per app, separate filesystems in mind (separate dirs) |
| Compromised app reading panel secrets | Panel DB + `/etc/panel/key` are `0600 panel:panel`. FPM user for an app is `panel-app-<name>`, NOT in group `panel`. No path leads from app to panel config. |
| Stolen S3 creds | Encrypted at rest in panel DB. Use a bucket policy restricting by IP if the provider supports it. Document. |
| Stolen CF token | Encrypted at rest. Token has scoped permissions (Zone:DNS:Edit only). |
| Privilege escalation | `www-data` is the only user Caddy runs as. FPM workers run as `panel-app-<name>` with no shell. systemd `NoNewPrivileges=yes`, `ProtectSystem=strict`, `ProtectHome=yes` on the panel service. |
| Log injection | All log lines JSON-encoded via `slog`. Caddy uses default combined format. Tailing in UI is done via `tail -n 200` over a `*Server-Sent Events*` stream that reads + escapes. |

### 8.4 Encryption at rest

- Master key: 32 random bytes, generated by installer at
  `/etc/panel/key` (mode `0600`, owner `panel:panel`).
- Algorithm: AES-256-GCM. Per-record random nonce, stored as
  `nonce || ciphertext || tag`, base64.
- Encrypted fields: `apps.env.<key>`, `cloudflare.api_token`,
  `apps.deploy_key` (the SSH private key).
- The key never leaves the host. There is no recovery mechanism — if the
  user loses it, they re-paste env vars. Document loudly.

### 8.5 Auth

- Username + password, bcrypt cost 12.
- Session: 32-byte random ID, stored in `sessions` table, expires in 7 days
  of inactivity, sliding window.
- Cookie: `panel_sid`, `HttpOnly`, `Secure` (when behind HTTPS), `SameSite=Strict`.
- CSRF: double-submit cookie. Non-GET requests must send
  `X-CSRF-Token` header matching the `panel_csrf` cookie.
- Login rate limit: 5 attempts / minute / IP, exponential backoff on
  failure, lockout after 20 / hour / IP / username.
- First-run default user is `admin` with a random 24-character password,
  printed exactly once. Force change on first login.

### 8.6 What the panel binary refuses to do

- Run as non-root in `install` mode (root is required for `apt` / `useradd`).
- Run as root in `serve` mode. The systemd unit has `User=panel`. Any
  operation needing root (reloading services, writing `/etc/caddy/...`) is
  done via a small allow-listed sudoers drop-in: `/etc/sudoers.d/panel`,
  granting `panel` NOPASSWD on exactly:
  - `/usr/bin/systemctl reload caddy`
  - `/usr/bin/systemctl reload php8.5-fpm`
  - `/usr/bin/systemctl restart php8.5-fpm`
  - `/usr/bin/systemctl reload panel-backup.timer`
  - `/usr/bin/tee /etc/caddy/apps/*.caddy`
  - `/usr/bin/tee /etc/panel/php/fpm/pool.d/*.conf`
  - `/usr/sbin/useradd`, `/usr/sbin/userdel` (for `panel-app-*` only —
    enforced via sudoers `Cmnd_Alias` with arg matching).

No blanket `NOPASSWD: ALL`. The dev agent MUST NOT add any.

---

## 9. Backups

### 9.1 What gets backed up

For each app:
- `/var/lib/panel/apps/<name>/app/` (git checkout, deterministic)
- `/var/lib/panel/apps/<name>/sqlite/` (the actual data)
- `/var/lib/panel/apps/<name>/.env` (encrypted — yes, back it up; if you
  lose it you lose the app)
- `/var/lib/panel/apps/<name>/runtime/` is **not** backed up (uploads,
  sessions, caches — user opted in by writing to runtime)

Plus panel-wide:
- `/var/lib/panel/panel.db`
- `/etc/panel/key` (encrypted blob of env vars becomes unrecoverable
  without it)
- `/etc/caddy/Caddyfile` and `/etc/caddy/apps/*.caddy`
- `/etc/panel/php/fpm/pool.d/panel-app-*.conf`

### 9.2 Schedule

- `panel-backup.timer` → daily at 03:00 local time, `RandomizedDelaySec=15m`.
- Configurable in `panel.yaml`: `backup.schedule` (systemd `OnCalendar`).
- Manual run: `panel backup run` (CLI) or "Run Now" button in UI.

### 9.3 Mechanism

1. `panel backup run` (invoked by timer or manually).
2. For each app: `tar -czf /var/lib/panel/backups/<app>-<date>.tar.gz`
   (incremental would be nicer but tar is enough for 3–5 small apps).
3. Same for panel state.
4. `rclone sync /var/lib/panel/backups/ remote:panel-backups/`
   (remote configured via `panel backup config` wizard, or via
   `/etc/rclone/rclone.conf` directly).
5. Apply retention: keep last 7 dailies, 4 weeklies, 12 monthlies.
   Older archives deleted from both local and remote.

### 9.4 Restore

- UI: "Backups" page lists every backup, per-app and whole-panel.
  Each has a "Download" (pulls tar from S3 via rclone serve) and a
  "Restore" (pipes tar into `/var/lib/panel/apps/<name>/`, then reloads
  FPM).
- CLI: `panel backup restore <id>`.

### 9.5 S3-compatible targets

Any rclone-supported S3 backend: AWS S3, Cloudflare R2, Backblaze B2,
DigitalOcean Spaces, MinIO, Wasabi, etc. Config stored in
`/var/lib/rclone/rclone.conf`, encrypted by rclone's own
`--password-command` shim. We don't reinvent this.

---

## 10. Cloudflare Integration

### 10.1 Setup

- Settings page: paste Cloudflare API token, pick zone.
- Token must have `Zone:DNS:Edit` permission only. UI shows a one-time
  reminder of how to scope it.
- Panel calls `GET /zones` to list zones for the token, lets the user pick
  which zone(s) to manage.

### 10.2 Capabilities (v1)

- List zones the token has access to.
- List DNS records for a zone.
- Create an A record on app create (if `cloudflare_managed: true`):
  `<domain>` → VPS IPv4, TTL=1 (auto), proxied=true (orange cloud).
  - If proxied: panel port 80/443 must be reachable from Cloudflare IPs.
  - Caddy will obtain a Let's Encrypt cert on the fly (on-demand TLS).
- Update / delete records on app rename or delete.
- Toggle proxy (orange cloud ↔ DNS-only) from the UI.
- Show last 50 audit-log entries for the zone (read-only).

### 10.3 Out of scope for v1

- WAF rules, page rules, Workers, Tunnel setup UI. Document CLI escape
  hatches.

### 10.4 Failure handling

- Cloudflare API errors are surfaced in the UI as a banner on the
  affected app.
- Deploy does not block on DNS. The app is reachable even if the DNS
  record creation failed; the user sees a warning and a "Retry" button.

---

## 11. System Health & Logs

### 11.1 Health page (UI)

- VPS: CPU%, load, RAM, swap, disk per mount, uptime.
- Per app: FPM pool status (active workers, idle workers, max), last
  deploy, last deploy status, current commit SHA, last 50 access log lines,
  last 50 error log lines.
- Panel: daemon uptime, version, DB size, panel.db last backup.

### 11.2 Implementation

- `gopsutil` for host metrics (sampled every 5s, cached).
- FPM status: enable `pm.status_path = /fpm-status` per pool. Caddy does
  NOT expose `/fpm-status` publicly. Instead, the panel scrapes it via a
  tiny internal HTTP call to `http://127.0.0.1:9xxx/fpm-status` where
  `9xxx` is a per-app FPM "status" pool listening on a separate socket the
  panel reverse-proxies via an internal-only Caddy listener on
  `127.0.0.1:2019`. Cleaner alternative: parse FPM's `pm.status_list` from
  the unix socket — actually FPM doesn't expose this over unix sockets
  easily, so we go with a small internal Caddy listener. Locked: yes.
- Log tail: `tail -n 200 -F /var/log/panel/apps/<name>/error.log` exposed
  over Server-Sent Events at `GET /api/apps/:id/logs/stream?file=error&token=...`.

### 11.3 Alerts

- Out of scope for v1. The user can grep the panel log or use an external
  uptime checker against the dashboard URL.

---

## 12. Configuration

`/etc/panel/panel.yaml`:

```yaml
panel:
  listen: ":8080"
  base_url: "http://203.0.113.10:8080"     # for absolute URLs in webhooks etc
  trust_cloudflare: true
  session_ttl_hours: 168                   # 7 days

backup:
  schedule: "daily"                         # or "*-*-* 03:00:00" systemd
  retention:
    daily: 7
    weekly: 4
    monthly: 12
  rclone_remote: ""                         # set via `panel backup config`
  local_staging: /var/lib/panel/backups

cloudflare:
  default_proxied: true
  default_ttl: 1                            # auto

php:
  default_version: "8.5"
  opcache_enable: true
  opcache_memory_mb: 128
  upload_max_mb: 64
  post_max_mb: 64

logging:
  level: info                               # debug | info | warn | error
  format: json
```

---

## 13. Data Model (SQLite)

`/var/lib/panel/panel.db`:

```sql
PRAGMA journal_mode = WAL;
PRAGMA foreign_keys = ON;
PRAGMA busy_timeout = 5000;

CREATE TABLE schema_version (
    version INTEGER PRIMARY KEY
);

CREATE TABLE users (
    id              INTEGER PRIMARY KEY,
    username        TEXT NOT NULL UNIQUE,
    password_hash   TEXT NOT NULL,         -- bcrypt
    role            TEXT NOT NULL DEFAULT 'admin',
    created_at      INTEGER NOT NULL,      -- unix seconds
    last_login_at   INTEGER,
    must_change_password INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE sessions (
    id              TEXT PRIMARY KEY,       -- random 32 bytes, base64
    user_id         INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    created_at      INTEGER NOT NULL,
    expires_at      INTEGER NOT NULL,
    user_agent      TEXT,
    ip              TEXT
);
CREATE INDEX idx_sessions_expires ON sessions(expires_at);

CREATE TABLE apps (
    id                  INTEGER PRIMARY KEY,
    name                TEXT NOT NULL UNIQUE,        -- slug
    repo_url            TEXT NOT NULL,
    branch              TEXT NOT NULL DEFAULT 'main',
    domain              TEXT,                         -- nullable until set
    docroot             TEXT NOT NULL DEFAULT '',     -- auto-detected
    deploy_key_enc      BLOB,                         -- SSH key, encrypted
    webhook_token       TEXT NOT NULL,                -- 32 bytes base64
    webhook_secret      TEXT,                         -- optional, for HMAC
    cloudflare_zone_id  TEXT,
    cloudflare_record_id TEXT,
    cloudflare_proxied  INTEGER NOT NULL DEFAULT 1,
    auto_deploy         INTEGER NOT NULL DEFAULT 1,
    status              TEXT NOT NULL DEFAULT 'created', -- created|running|stopped|errored
    last_deploy_at      INTEGER,
    last_deploy_sha     TEXT,
    created_at          INTEGER NOT NULL,
    updated_at          INTEGER NOT NULL
);

CREATE TABLE app_env (
    app_id     INTEGER NOT NULL REFERENCES apps(id) ON DELETE CASCADE,
    key        TEXT NOT NULL,
    value_enc  BLOB NOT NULL,                 -- AES-GCM(value)
    PRIMARY KEY (app_id, key)
);

CREATE TABLE deployments (
    id          INTEGER PRIMARY KEY,
    app_id      INTEGER NOT NULL REFERENCES apps(id) ON DELETE CASCADE,
    commit_sha  TEXT,
    status      TEXT NOT NULL,                -- running|success|failed
    log_path    TEXT,                         -- tail of /var/log/panel/apps/<name>/deploy.log
    started_at  INTEGER NOT NULL,
    finished_at INTEGER
);
CREATE INDEX idx_deployments_app_started ON deployments(app_id, started_at DESC);

CREATE TABLE backups (
    id           INTEGER PRIMARY KEY,
    kind         TEXT NOT NULL,                -- 'app' | 'panel'
    app_id       INTEGER REFERENCES apps(id) ON DELETE CASCADE,
    archive_path TEXT NOT NULL,                -- local
    remote_path  TEXT,                          -- s3 path
    size_bytes   INTEGER NOT NULL,
    sha256       TEXT NOT NULL,
    created_at   INTEGER NOT NULL
);
CREATE INDEX idx_backups_app_created ON backups(app_id, created_at DESC);

CREATE TABLE settings (
    key   TEXT PRIMARY KEY,
    value BLOB                                   -- encrypted if sensitive
);

-- Encrypted settings:
-- 'cloudflare.api_token'
-- 'rclone.config_encrypted'
```

---

## 14. HTTP API (REST)

All endpoints under `/api`. All non-GET require CSRF header
`X-CSRF-Token` matching the `panel_csrf` cookie. Auth is the session cookie.

| Method | Path | Purpose |
|--------|------|---------|
| `POST` | `/api/auth/login` | `{username, password}` → set cookie |
| `POST` | `/api/auth/logout` | invalidate session |
| `GET`  | `/api/auth/me` | current user info |
| `POST` | `/api/auth/password` | change own password |
| `GET`  | `/api/system` | host metrics + panel version |
| `GET`  | `/api/system/logs?tail=200` | tail of panel.log |
| `GET`  | `/api/apps` | list apps |
| `POST` | `/api/apps` | create app |
| `GET`  | `/api/apps/:id` | app detail |
| `PATCH`| `/api/apps/:id` | update (rename, domain, env, auto-deploy, CF proxied) |
| `DELETE`| `/api/apps/:id` | delete app |
| `POST` | `/api/apps/:id/deploy` | manual deploy |
| `POST` | `/api/apps/:id/stop` | disable (Caddy → 503) |
| `POST` | `/api/apps/:id/start` | re-enable |
| `GET`  | `/api/apps/:id/logs?file=error|access|deploy&tail=200` | log tail (one-shot) |
| `GET`  | `/api/apps/:id/logs/stream?file=...&token=...` | SSE log stream |
| `GET`  | `/api/apps/:id/fpm-status` | FPM pool metrics |
| `GET`  | `/api/apps/:id/deployments` | deployment history |
| `POST` | `/api/hooks/deploy/:token` | GitHub webhook (no CSRF, HMAC-validated if secret set) |
| `GET`  | `/api/backups` | list backups |
| `POST` | `/api/backups/run` | trigger full backup |
| `POST` | `/api/backups/run/:app_id` | backup one app |
| `POST` | `/api/backups/restore/:id` | restore (requires `confirm: true`) |
| `GET`  | `/api/cloudflare/zones` | list zones (proxied) |
| `GET`  | `/api/cloudflare/zones/:id/records` | list DNS records |
| `POST` | `/api/cloudflare/zones/:id/records` | create record |
| `PATCH`| `/api/cloudflare/zones/:zid/records/:rid` | update |
| `DELETE`| `/api/cloudflare/zones/:zid/records/:rid` | delete |
| `POST` | `/api/cloudflare/test` | validate stored token |
| `GET`  | `/api/settings` | get sanitized settings |
| `PATCH`| `/api/settings` | update (e.g. base_url, listen) |

Response shape (success):

```json
{ "ok": true, "data": { ... } }
```

Error:

```json
{ "ok": false, "error": { "code": "app_not_found", "message": "..." } }
```

---

## 15. Frontend Pages

All server-rendered Go html/template files, embedded in the binary. HTMX
for partial updates, Alpine.js for tiny client state. Tailwind CSS compiled
once at build time (`tailwindcss --input src.css --output static/app.css`
in the Makefile), embedded in the binary.

| Page | Route | Notes |
|------|-------|-------|
| Login | `/login` | Single field form. Wrong-password banner. |
| Dashboard (home) | `/` | VPS health strip + apps grid (status, last deploy, CTA) |
| Apps list | `/apps` | Table view with status pills |
| App detail | `/apps/:id` | Tabs: Overview / Env / Domain / Deployments / Logs / Backups |
| New app | `/apps/new` | Form: name, repo, branch, env, domain, CF toggle |
| Settings | `/settings` | Tabs: General / Cloudflare / Backups / Updates / Account |
| Backups | `/backups` | Table of all backups, "Run now" |
| System | `/system` | Full health, host metrics, panel logs |

### 15.1 Visual language

- Dark mode by default, light mode toggle (persisted in localStorage).
- Tailwind palette: zinc + sky. No marketing copy. Pure utility.
- One SVG logo. No icon font; inline SVG only.

---

## 16. CLI Surface

The same binary, no separate CLI program. Subcommands:

```
panel install                    # idempotent installer
panel uninstall                  # removes services, keeps /var/lib/panel
panel serve                      # foreground, used by systemd
panel status                     # one-line health summary
panel app list
panel app create --from-yaml <file>
panel app deploy <name>
panel app stop <name>
panel app start <name>
panel app logs <name> [--follow] [--file error|access|deploy]
panel app env set <name> KEY=VALUE
panel app env unset <name> KEY
panel backup list
panel backup run [--app <name>]
panel backup restore <id>
panel backup config              # interactive wizard for rclone remote
panel user list
panel user create <username>
panel user passwd <username>
panel user delete <username>
panel cloudflare zones
panel cloudflare test
panel update [--channel stable|beta]
panel version
```

`--from-yaml` enables scripted installs: define the whole app in a YAML
file, ship it to the VPS, `panel app create --from-yaml app.yaml`.

---

## 17. Distribution & Updates

### 17.1 Install path

```
curl -sSL https://get.panel.example/install | sudo bash
```

`install` shell script:
1. `uname -m` → `amd64` or `arm64`.
2. Download latest release from `https://github.com/<org>/panel/releases/latest/download/panel-<ver>-linux-<arch>.tar.gz`.
3. Verify `checksums.txt` + `cosign` signature.
4. Extract `panel` to `/usr/local/bin/panel`, `chmod 0755`.
5. `exec /usr/local/bin/panel install`.

### 17.2 Self-update

- `panel update` downloads the new tarball, verifies, replaces
  `/usr/local/bin/panel`, and `systemctl restart panel`.
- Channel: `stable` (semver tags) or `beta` (latest main commit).
- A `panel-update.timer` (weekly) can be enabled by the user to auto-update
  with a 7-day delay after release (so we don't ship breaking builds).
- Auto-updates default to **off**.

### 17.3 Build

- `Makefile` targets: `build`, `test`, `lint`, `embed-assets`, `release`.
- Cross-compile: `GOOS=linux GOARCH=amd64 go build` produces a fully static
  binary thanks to `modernc.org/sqlite` (no CGO).
- `goreleaser` for release artifacts + checksums + cosign signing.

---

## 18. Edge Cases & Failure Modes

| Scenario | Behavior |
|----------|----------|
| `git clone` fails (auth, 404) | Deploy row marked `failed`; UI shows last 50 lines of deploy log |
| `composer install` fails | Deploy row marked `failed`; previous code left in place (we only touched `app/` after a successful clone) — actually no, we did `reset --hard`. So we `git stash` or `git checkout` to the previous SHA before the reset. |
| FPM reload fails | Panel rolls back the pool config; previous pool keeps running |
| Caddy hot reload fails | Panel rolls back the vhost snippet; old Caddy config still loaded |
| Cloudflare API down | DNS step logged as failed; deploy still succeeds; user can retry from app detail page |
| S3 backup target unreachable | Local backup still kept; UI shows "Remote sync failed — last success: X hours ago" |
| Disk full during deploy | Deploy fails; UI surfaces `df` output; previous code untouched (we do a clone to a temp dir, swap atomically) |
| Two deploys of same app in flight | `flock` per app; second waits |
| User loses `/etc/panel/key` | Cannot decrypt env vars. UI shows a "rotate" wizard: user re-pastes env vars, key stays. (A lost key = irrecoverable encrypted data.) |
| User deletes the only admin | Recovery: `panel user create admin` from CLI |
| `php8.5-fpm` crashes | systemd restarts; if it keeps crashing, panel surfaces the FPM error log on the dashboard |
| Cloudflare token revoked | DNS operations fail; UI shows banner; deploy still works locally |
| VPS reboots | systemd starts caddy, php8.5-fpm, panel, panel-backup.timer — all apps come back online automatically |

### 18.1 Atomic deploys

`git reset --hard` is not atomic. Pattern:

```
git clone --depth 1 <repo> /var/lib/panel/apps/<name>/app.new
if composer.json: composer install in app.new
rsync -a --delete app.new/ app/         # or just mv
rm -rf app.new
systemctl reload php8.5-fpm
```

OK, `mv` is atomic on the same filesystem. Use that.

---

## 19. Operational Runbook (for the user, baked into docs)

- "How do I add an app?" → paste repo, click Deploy.
- "How do I see logs?" → app detail → Logs tab.
- "How do I rollback?" → Deployments tab → "Redeploy this commit" button.
- "How do I update the panel?" → Settings → Updates → Check / Apply.
- "How do I restore from backup?" → Backups → pick one → Restore.
- "How do I rotate the encryption key?" → CLI: `panel key rotate`
  (re-encrypts all `app_env` and settings under the new key; old key
  kept as a backup file until you `rm` it).
- "How do I uninstall everything?" → `panel uninstall`. Refuses unless
  `--yes` is passed.

---

## 20. Phased Delivery

### Milestone 1 — Installer + empty dashboard
- `panel install` provisions everything.
- Dashboard at `:8080` shows "no apps yet."
- User can log in, change password, see system health.

### Milestone 2 — App lifecycle (no GitHub yet)
- `panel app create` with local path (no git) for testing.
- FPM pool + Caddy vhost generated.
- App reachable on `:80/:443`.
- `panel app logs`, `panel app stop/start`.

### Milestone 3 — GitHub deploys
- Git clone + composer + atomic swap.
- Manual deploy button + webhook.
- Deploy history.

### Milestone 4 — Backups
- `panel-backup.timer`.
- rclone wizard, daily push, retention.
- UI for list / restore.

### Milestone 5 — Cloudflare
- Token setup, zones, A record on app create, delete on app delete.
- UI: records list, proxied toggle.

### Milestone 6 — Polish
- Log streaming (SSE).
- FPM status scraping.
- Self-update.
- Tests, CI, signed releases.

Each milestone is independently shippable. Stop at any milestone and the
panel is useful.

---

## 21. Testing Strategy

- **Unit tests:** Go stdlib `testing` for: key wrapping, env var encryption,
  Caddyfile / FPM pool generation, deploy script, API handlers (with
  `httptest`).
- **Integration tests:** `dockertest`-style with `docker run -d
  --privileged ubuntu:24.04` (or just a local VM via `vagrant`). Skip on
  CI without privileged access; mark as "manual gate."
- **Smoke test on every release:** an Ansible/CI job that runs `panel
  install` in a fresh LXC, creates a sample PHP app, deploys, hits the
  endpoint, runs `panel backup run` against a local MinIO.
- **No E2E browser tests** in v1. Manual QA.

---

## 22. Open Questions (for the dev agent to confirm before coding)

1. **Caddy DNS challenge for Let's Encrypt** — does the user want to use
   Cloudflare DNS challenge (zero open port for `:80`) or HTTP-01 (requires
   `:80`)? Recommend Cloudflare DNS challenge since the user already has a
   CF account. Default: DNS challenge via Cloudflare token.
2. **Multi-user?** — confirmed single user. If multi-user comes later,
   `users` table is ready; UI just needs a "Users" tab.
3. **Custom domains per env (staging/prod)?** — v1 is one domain per app.
4. **Laravel-specific niceties** (queue workers, scheduler, octane)? — out
   of scope. The user can run `* * * * * php artisan schedule:run` via a
   per-app cron file. Document.
5. **Database migrations** — generic "run `php artisan migrate --force` if
   `artisan` exists" is the rule. If the user uses a framework without
   `artisan` or with a different migrate binary, they can add a
   `panel.post_deploy` script in the app root that the panel will execute.
6. **Build steps** (npm, vite, etc.) — same: `panel.post_deploy` script in
   the repo, executed after `composer install`. v1 doesn't auto-detect.
7. **ARM64 VPS** — supported (Caddy, PHP 8.5 from Sury both have arm64
   builds). Confirmed.

---

## 23. Locked Decisions Summary (restated, for the dev agent)

1. **Go 1.22+, single static binary, `embed.FS` for all UI assets.**
2. **SQLite via `modernc.org/sqlite` (pure Go).** No CGO. No MySQL/Postgres.
3. **Caddy is the edge web server. Period.** The panel binary is not a
   reverse proxy.
4. **PHP 8.5 only.** Single `php8.5-fpm` service, one pool per app.
5. **Per-app FPM user = `panel-app-<name>`, no login, no shell.**
6. **Per-app `open_basedir`.** Per-app unix socket for FPM.
7. **Dashboard on `:8080`, apps on `:80/:443`.**
8. **Frontend: server-rendered + HTMX + Alpine + Tailwind, embedded.**
9. **Auth: bcrypt + session cookie + CSRF + login rate limit.**
10. **Encryption at rest: AES-256-GCM, key in `/etc/panel/key` (0600).**
11. **Backups: rclone to S3, daily systemd timer, retention 7/4/12.**
12. **Cloudflare: official SDK, scoped DNS-only API token.**
13. **Deploy: git + composer + atomic `mv` of new checkout into place.**
14. **Per-app webhook: signed token in URL, optional HMAC.**
15. **Self-update: `go-update` against GitHub releases, default off.**
16. **Supported OS v1: Ubuntu 22.04+, Debian 12+.**
17. **No Docker, no Kubernetes, no Swarm, no Nomad. FPM pools are the
    isolation primitive.**
18. **No built-in mail server. Use SMTP relays.**
19. **No built-in Redis. If the app needs cache/session storage, it
    must use SQLite or an external service. (We could add a `memcached`
    or `redis-server` apt-install in v1.1 if the user pushes back.)**
20. **Encryption key rotation is supported. Key loss is unrecoverable —
    surface this in the UI before the user sets a Cloudflare token or
    creates their first app.**

---

## 24. Appendix A — Example `panel app create --from-yaml`

```yaml
# app.yaml
name: blog
repo_url: git@github.com:me/blog.git
branch: main
deploy_key: |
  -----BEGIN OPENSSH PRIVATE KEY-----
  b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
  ...
  -----END OPENSSH PRIVATE KEY-----
env:
  APP_ENV: production
  APP_DEBUG: "false"
  APP_KEY: base64:K7qJ9z9oM4C5p2wQr8bN0=
  DB_CONNECTION: sqlite
  DB_DATABASE: /var/lib/panel/apps/blog/sqlite/blog.sqlite
domain: blog.example.com
cloudflare_managed: true
auto_deploy: true
```

## 25. Appendix B — `panel.service`

```ini
[Unit]
Description=Panel control plane
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=panel
Group=panel
WorkingDirectory=/var/lib/panel
EnvironmentFile=-/etc/panel/panel.env
ExecStart=/usr/local/bin/panel serve
Restart=always
RestartSec=3
NoNewPrivileges=yes
ProtectSystem=strict
ProtectHome=yes
PrivateTmp=yes
PrivateDevices=yes
ReadWritePaths=/var/lib/panel /var/log/panel /var/lib/rclone
ProtectKernelTunables=yes
ProtectKernelModules=yes
ProtectControlGroups=yes
RestrictSUIDSGID=yes
LockPersonality=yes
RestrictRealtime=yes
SystemCallArchitectures=native

[Install]
WantedBy=multi-user.target
```

## 26. Appendix C — `panel-backup.service` + timer

```ini
# /etc/systemd/system/panel-backup.service
[Unit]
Description=Panel backup job
After=network-online.target

[Service]
Type=oneshot
User=panel
Group=panel
ExecStart=/usr/local/bin/panel backup run
Nice=10
IOSchedulingClass=best-effort
```

```ini
# /etc/systemd/system/panel-backup.timer
[Unit]
Description=Daily panel backup

[Timer]
OnCalendar=*-*-* 03:00:00
RandomizedDelaySec=15m
Persistent=true

[Install]
WantedBy=timers.target
```

---

*End of document.*
