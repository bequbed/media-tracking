# GoPlaxt & Plexytrack — Runbook (Oracle Cloud)

*Last updated: 28 Aug 2025 (NZ)*

This is the working reference for how we set up and operate **GoPlaxt** (live Trakt scrobbles from Plex) and **Plexytrack** (Plex ⇄ Trakt/Simkl sync) across your Oracle Cloud servers.

---

## 1) High‑level architecture

* **Media trackers:** Trakt, Simkl, and Letterboxd (primary record of truth for watch history and lists).
* **Live scrobbling:** **GoPlaxt** on the **Oracle Ubuntu server** (ARM) — listens for Plex webhooks / activity and instantly sends scrobbles to Trakt.
* **Batch syncs:** **Plexytrack** on the **Oracle Sydney server** — reconciles Plex ↔ Trakt/Simkl using API tokens and scheduled jobs.
* **Remote access:** Management via SSH/Tailscale. Both apps are containerized with Docker Compose and keep app config in `/opt/.../config` on the host.

---

## 2) Server & paths

* **GoPlaxt**

  * Host: Oracle Ubuntu server (ARM)
  * Path: `/opt/goplaxt`
  * Config: `/opt/goplaxt/config` (mapped to container `/config`)
* **Plexytrack**

  * Host: Oracle *Sydney* server
  * Path: `/opt/plexytrack`
  * Config: `/opt/plexytrack/config` (mapped to container `/config`)

> **Back up** both `config` folders periodically — they contain your tokens (Trakt/Simkl/Plex) and app settings.

---

## 3) Prerequisites (on both hosts)

1. OS packages

   ```bash
   sudo apt update && sudo apt install -y docker.io docker-compose-plugin git
   sudo usermod -aG docker $USER
   # log out/in so docker group applies
   ```
2. Timezone

   ```bash
   sudo timedatectl set-timezone Pacific/Auckland
   ```
3. Create app folders

   ```bash
   sudo mkdir -p /opt/goplaxt/config /opt/plexytrack/config
   sudo chown -R $USER:$USER /opt/goplaxt /opt/plexytrack
   ```

---

## 4) GoPlaxt (live scrobbles)

### 4.1 Get the code & fix Git ownership warning

We keep a local working tree primarily for reference and updates.

```bash
cd /opt/goplaxt
git clone https://github.com/bequbed/goplaxt.git .
# If you see: "detected dubious ownership" run:
git config --global --add safe.directory /opt/goplaxt
```

### 4.2 Docker Compose file

Create `/opt/goplaxt/docker-compose.yml`:

```yaml
services:
  goplaxt:
    image: ghcr.io/bequbed/goplaxt:latest
    container_name: goplaxt
    restart: unless-stopped
    environment:
      # Replace with your actual values
      TRAKT_CLIENT_ID: "<trakt_client_id>"
      TRAKT_CLIENT_SECRET: "<trakt_client_secret>"
      TRAKT_REDIRECT_URI: "urn:ietf:wg:oauth:2.0:oob"
      PLEX_TOKEN: "<plex_token>"
      PLEX_SERVER_URL: "https://<plex-direct-or-fqdn>:32400"
      TZ: "Pacific/Auckland"
    volumes:
      - ./config:/config
    ports:
      - "<optional_host_port>:<app_port>"  # only if you expose a UI/health port
```

> Notes
>
> * Use your **Plex token** and a **plex.direct** or FQDN URL reachable from the server. Self‑signed certs are fine; GoPlaxt handles webhook events.
> * If you placed secrets in this file, keep it **out of Git** (see §7.3).

### 4.3 First run & auth

```bash
cd /opt/goplaxt
docker compose pull
docker compose up -d
# Tail logs
docker compose logs -f
```

* Follow on‑screen instructions to authorize Trakt if prompted (device code / URL flow).
* Confirm a live scrobble by playing something in Plex and watching logs for scrobble events.

---

## 5) Plexytrack (Plex ↔ Trakt/Simkl sync)

### 5.1 Folder layout

```
/opt/plexytrack
├─ docker-compose.yml
└─ config/
   ├─ auth.json          # stores Trakt + Simkl tokens
   └─ settings.json      # Plex URL, library mapping, job schedule, SSL verify, etc.
```

### 5.2 Docker Compose file

Create `/opt/plexytrack/docker-compose.yml`:

```yaml
services:
  plexytrack:
    image: ghcr.io/<vendor>/plexytrack:latest
    container_name: plexytrack
    restart: unless-stopped
    environment:
      TZ: "Pacific/Auckland"
    volumes:
      - ./config:/config
```

> Replace the image reference with the actual one you used. We run the container with `/config` mapped so tokens and settings persist.

### 5.3 Configure tokens & settings

Create `/opt/plexytrack/config/auth.json` with **Trakt** and **Simkl** tokens (device auth or OAuth flow). Then create `/opt/plexytrack/config/settings.json` with at least:

```json
{
  "plex": {
    "base_url": "https://<plex-direct-or-fqdn>:32400",
    "token": "<plex_token>",
    "verify_ssl": false
  },
  "trakt": { "client_id": "<id>", "client_secret": "<secret>" },
  "simkl":  { "client_id": "<id>", "client_secret": "<secret>" },
  "sync": {
    "libraries": ["Movies", "TV Shows"],
    "mode": "two_way"
  },
  "schedule": {
    "enabled": true,
    "cron": "*/15 * * * *"  // every 15 minutes
  }
}
```

> **SSL warning:** You may see `InsecureRequestWarning` for `*.plex.direct` certificates. We set `verify_ssl` to `false` in `settings.json` so syncs proceed — expected in home‑lab contexts.

### 5.4 Run & verify

```bash
cd /opt/plexytrack
docker compose pull
docker compose up -d
# Tail logs
docker compose logs -f
```

* On startup you should see:

  * *Loaded Trakt tokens from `/config/auth.json`*
  * *Loaded Simkl token from `/config/auth.json`*
  * *Loaded sync settings from `/config/settings.json`*
* Trigger a manual sync (if supported) or wait for the cron to run; verify items appear on Trakt/Simkl.

---

## 6) Operations

* **Start/Stop/Restart**

  ```bash
  # GoPlaxt
  (cd /opt/goplaxt && docker compose up -d)
  (cd /opt/goplaxt && docker compose restart)
  (cd /opt/goplaxt && docker compose down)

  # Plexytrack
  (cd /opt/plexytrack && docker compose up -d)
  (cd /opt/plexytrack && docker compose restart)
  (cd /opt/plexytrack && docker compose down)
  ```
* **Update images**

  ```bash
  (cd /opt/goplaxt && docker compose pull && docker compose up -d)
  (cd /opt/plexytrack && docker compose pull && docker compose up -d)
  ```
* **Logs**

  ```bash
  (cd /opt/goplaxt && docker compose logs -f)
  (cd /opt/plexytrack && docker compose logs -f)
  ```
* **Backups** (tokens & settings)

  ```bash
  tar -czf ~/goplaxt-config-$(date +%F).tgz -C /opt/goplaxt config
  tar -czf ~/plexytrack-config-$(date +%F).tgz -C /opt/plexytrack config
  ```

---

## 7) Security & secrets hygiene

### 7.1 Rotate keys if exposed

If an API key/token was ever committed publicly, **rotate it** in the provider (Trakt/Simkl/Plex) and update the config files.

### 7.2 Handle self‑signed certs

Using `plex.direct` often yields self‑signed/LE mismatch; `verify_ssl: false` in Plexytrack avoids breaking sync. (Optional: terminate TLS behind a trusted reverse proxy if exposing to the internet — otherwise keep private via Tailscale.)

### 7.3 Keep docker‑compose out of Git (if it contains secrets)

```bash
cd /opt/goplaxt
printf "\ndocker-compose.yml\nconfig/*.json\nconfig/*.yaml\n" >> .gitignore
# If you accidentally committed secrets:
git rm --cached docker-compose.yml
# Commit and push the removal (then rotate secrets!)
```

---

## 8) Troubleshooting we already hit (and fixes)

* **Git ‘detected dubious ownership’** in `/opt/goplaxt` → `git config --global --add safe.directory /opt/goplaxt`.
* **Plexytrack `InsecureRequestWarning`** for `*.plex.direct` → expected; set `"verify_ssl": false` in `settings.json`.
* **Auth failures (401/403) or ‘not found on Trakt’** → usually stale tokens or bad IDs; re‑auth Trakt/Simkl, re‑scan Plex libraries, confirm GUIDs are present (TMDB/TVDB/IMDB) and match provider.

---

## 9) Verification checklist (after changes)

* Play an item in Plex → GoPlaxt logs show instant scrobble to Trakt.
* Run/await Plexytrack scheduled job → new/updated plays appear in Trakt & Simkl; mismatches resolved.
* Confirm no errors in `docker compose logs` for both services.

---

## 10) Change log (key events)

* **27–28 Aug 2025**: Finalized dual‑stack setup — GoPlaxt (Ubuntu host) for live scrobbles; Plexytrack (Sydney host) for scheduled sync. Addressed Git safe.directory warning; acknowledged `InsecureRequestWarning` and set `verify_ssl: false`.

---

### Appendix A — Where your tokens live

* **GoPlaxt:** in `/opt/goplaxt/config` (format depends on upstream image; keep private).
* **Plexytrack:** `/opt/plexytrack/config/auth.json` (Trakt + Simkl), `/opt/plexytrack/config/settings.json` (Plex + schedule).

> Treat `config/` like secrets: back it up, don’t commit it, and rotate on exposure.

## Appendix B — Redacted example config files

### B.1 Redacted `auth.json` (Plexytrack)

```json
{
  "trakt": {
    "access_token": "REDACTED_ACCESS_TOKEN",
    "refresh_token": "REDACTED_REFRESH_TOKEN",
    "expires_at": "2025-09-02T12:34:56Z",
    "client_id": "REDACTED_CLIENT_ID",
    "client_secret": "REDACTED_CLIENT_SECRET"
  },
  "simkl": {
    "access_token": "REDACTED_SIMKL_TOKEN",
    "refresh_token": "REDACTED_SIMKL_REFRESH",
    "expires_at": "2025-09-02T12:34:56Z"
  },
  "plex": {
    "token": "REDACTED_PLEX_TOKEN"
  }
}
```

### B.2 Redacted `settings.json` (Plexytrack)

```json
{
  "plex": {
    "base_url": "https://your-plex-domain.plex.direct:32400",
    "token": "REDACTED_PLEX_TOKEN",
    "verify_ssl": false
  },
  "trakt": {
    "client_id": "REDACTED_CLIENT_ID",
    "client_secret": "REDACTED_CLIENT_SECRET",
    "webhook_enabled": true
  },
  "simkl": {
    "client_id": "REDACTED_SIMKL_ID",
    "client_secret": "REDACTED_SIMKL_SECRET"
  },
  "sync": {
    "libraries": ["Movies", "TV Shows"],
    "mode": "two_way",
    "remove_watched": true
  },
  "schedule": {
    "enabled": true,
    "cron": "*/15 * * * *"
  },
  "logging": {
    "level": "info",
    "file": "/config/sync.log"
  }
}
```

### B.3 Redacted GoPlaxt environment / .env example

```
TRAKT_CLIENT_ID=REDACTED_CLIENT_ID
TRAKT_CLIENT_SECRET=REDACTED_CLIENT_SECRET
PLEX_TOKEN=REDACTED_PLEX_TOKEN
PLEX_SERVER_URL=https://your-plex-domain.plex.direct:32400
TZ=Pacific/Auckland
```

> **Notes:**
>
> * These examples are intentionally redacted. Replace `REDACTED_*` placeholders with the real tokens/IDs on the server.
> * Never commit files containing real secrets to Git. Add `/opt/goplaxt/docker-compose.yml` and `/opt/plexytrack/config/*` to `.gitignore` if you track the repo locally.
> * After updating tokens, restart the containers: `docker compose restart` in each project folder.
