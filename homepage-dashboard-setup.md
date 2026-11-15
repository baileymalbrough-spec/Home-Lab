# Homepage Dashboard Setup — Home Lab

_Deployed Nov 15, 2026. One portal for every Home Lab service (the dashboard style from the CodeWarr1or TikTok is [Homepage](https://gethomepage.dev)). It runs in Docker on the XPS next to everything else. This doc matches what's actually running. Use it to check settings or rebuild._

## Overview
- **What it is:** a fast dashboard driven by YAML files. Each service gets a tile with a link, an up/down dot and, where supported, live stats.
- **Host:** Ubuntu PC (Dell XPS 8700), `192.168.68.56`, Docker with **host networking**
- **URLs:**
  - At home: `http://192.168.68.56:3000`
  - Tailnet: `http://bailey-malbrough-xps-8700:3000`
- **Port map neighbors:** 3000 Homepage · 3001 Uptime Kuma (planned) · 3002 ntopng
- **RAM cost:** about 100–150 MB
- **Adding a service** = paste one block into `config/services.yaml` and refresh. No restart is needed for YAML changes; changes to `.env` or the compose file need `docker compose up -d`.

## Status (Nov 15, 2026): all tiles working ✅

| Group | Tile | What it shows | Link |
|---|---|---|---|
| Network | Pi-hole | queries, blocked (%), gravity | `http://192.168.68.56:8081/admin` |
| Network | Tailscale (XPS) | tailnet address, last seen, update available | Tailscale admin console |
| Network | ntopng | container up/down (Homepage has no ntopng widget) | `http://192.168.68.56:3002` |
| Network | Deco router | ping up/down | `http://192.168.68.1` |
| Media | Jellyfin | movies/series/episodes/songs counts, now playing | `http://192.168.68.56:8096` |
| Cloud | Nextcloud | free space, active users, files, shared items | `https://bailey-malbrough-xps-8700.tailbe32a5.ts.net` |
| Cloud | Nextcloud AIO | container up/down | `https://192.168.68.56:8080` |
| Security | Wazuh | site up/down (no official widget) | `https://192.168.68.56:5601` |

Top bar: XPS CPU / RAM / disk / uptime, date/time and a search box. Bookmarks: Tailscale console, Deco, GS108Ev4 switch.

Notes from first launch:
- **Jellyfin shows 0 everywhere.** That's accurate: Jellyfin is used for IPTV live TV, and channels don't count as library items. To hide the row: `sed -i 's/enableBlocks: true/enableBlocks: false/' ~/homepage/config/services.yaml`
- **Pi-hole was blocking 49%.** That's high; a typical home is 10–25%. Check **Top Blocked Domains / Top Clients** in Pi-hole; the LG TV's `alphonso.tv` is the likely cause.
- **The Tailscale tile shows "Update available"** for the XPS. After updating, run the verification block in `tailscale-setup.md`, because the Nov 7 outage followed an update.

---

## Files

All under `/home/bailey-malbrough/homepage/`. Write files with a heredoc (`cat > file <<'EOF'` … `EOF`) or `nano`. Pasting raw YAML at the prompt just runs each line as a command.

### `docker-compose.yml`
```yaml
services:
  homepage:
    image: ghcr.io/gethomepage/homepage:latest
    container_name: homepage
    network_mode: host                # needed to reach Nextcloud on 127.0.0.1 / the ts.net name
    env_file: .env                    # API keys live here, never in the YAML
    environment:
      # Every hostname:port typed in the browser. Missing one = "Host validation failed".
      HOMEPAGE_ALLOWED_HOSTS: "192.168.68.56:3000,100.92.19.94:3000,bailey-malbrough-xps-8700:3000,bailey-malbrough-xps-8700.tailbe32a5.ts.net:3000"
    volumes:
      - ./config:/app/config
      - /var/run/docker.sock:/var/run/docker.sock:ro   # container up/down dots
    restart: unless-stopped
```
- **Why host networking:** on Docker's default bridge network, the Nextcloud widget failed with `ENETUNREACH` to the XPS's Tailscale address. Nextcloud's Apache only listens on `127.0.0.1:11000` behind `tailscale serve`, so a bridged container can't reach it. With host networking, Homepage reaches it exactly like the XPS does. There are no `ports:` lines, and Homepage still listens on 3000.
- **Security note:** the Docker socket is mounted read-only, but socket access is still powerful. Keep Homepage LAN/tailnet only, never on Funnel.
- After Nginx Proxy Manager is set up, add `home.bailey-lab.duckdns.org` to `HOMEPAGE_ALLOWED_HOSTS`.

### `.env` (secrets: `chmod 600`, listed in `.gitignore`)
```ini
HOMEPAGE_VAR_PIHOLE_KEY=
HOMEPAGE_VAR_JELLYFIN_KEY=
HOMEPAGE_VAR_NEXTCLOUD_TOKEN=
HOMEPAGE_VAR_TAILSCALE_DEVICE=
HOMEPAGE_VAR_TAILSCALE_KEY=
```
Homepage replaces `{{HOMEPAGE_VAR_NAME}}` in the YAML with these values. All keys are also stored in Bitwarden under **"Homepage API keys"**.

| Key | Where it comes from | Expires? |
|---|---|---|
| Pi-hole | **Settings → Web interface / API → Expert → Configure app password.** Copy it, then click **Enable new app password**. Check it saved: `docker exec pihole pihole-FTL --config webserver.api.app_pwhash` prints a hash. | No |
| Jellyfin | **Dashboard → API Keys → +**, named `homepage` | No |
| Nextcloud | `TOKEN=$(openssl rand -hex 24)` then `sudo docker exec --user www-data nextcloud-aio-nextcloud php occ config:app:set serverinfo token --value "$TOKEN"`, then `sed -i "s/^HOMEPAGE_VAR_NEXTCLOUD_TOKEN=.*/HOMEPAGE_VAR_NEXTCLOUD_TOKEN=$TOKEN/" ~/homepage/.env` | No |
| Tailscale device ID | Admin → Machines → `bailey-malbrough-xps-8700` → Machine Details → **ID** (ends in `CNTRL`) | No |
| Tailscale API token | Admin → Settings → Keys → **Generate access token**, named `homepage`, 90 days | **Yes, ~Feb 13, 2027.** A scheduled Claude reminder fires **Feb 8, 2027, 12:15 PM** with the replacement steps. |

### `config/settings.yaml`
```yaml
title: Bailey's Home Lab
theme: dark
color: slate
headerStyle: boxed
statusStyle: dot
hideVersion: true
layout:
  Network:
    style: row
    columns: 4
  Media:
    style: row
    columns: 2
  Cloud:
    style: row
    columns: 2
  Security:
    style: row
    columns: 2
```

### `config/docker.yaml`
```yaml
xps:
  socket: /var/run/docker.sock
```

### `config/services.yaml`
```yaml
- Network:
    - Pi-hole:
        icon: pi-hole.png
        href: http://192.168.68.56:8081/admin
        description: Network-wide DNS ad blocking
        server: xps
        container: pihole
        widget:
          type: pihole
          url: http://192.168.68.56:8081
          version: 6
          key: "{{HOMEPAGE_VAR_PIHOLE_KEY}}"
    - Tailscale:
        icon: tailscale.png
        href: https://login.tailscale.com/admin/machines
        description: Subnet router + exit node (XPS)
        widget:
          type: tailscale
          deviceid: "{{HOMEPAGE_VAR_TAILSCALE_DEVICE}}"
          key: "{{HOMEPAGE_VAR_TAILSCALE_KEY}}"
          fields: ["address", "last_seen", "update_available"]
    - ntopng:
        icon: ntopng.png
        href: http://192.168.68.56:3002
        description: Live traffic monitoring
        server: xps
        container: ntopng
    - Deco Router:
        icon: tp-link.png
        href: http://192.168.68.1
        description: TP-Link Deco AX1500
        ping: 192.168.68.1

- Media:
    - Jellyfin:
        icon: jellyfin.png
        href: http://192.168.68.56:8096
        description: Movies, shows and live TV
        server: xps
        container: jellyfin
        widget:
          type: jellyfin
          url: http://192.168.68.56:8096
          key: "{{HOMEPAGE_VAR_JELLYFIN_KEY}}"
          version: 2                  # required: Jellyfin 12.4.0 removed the old /emby endpoints
          enableBlocks: true
          enableNowPlaying: true

- Cloud:
    - Nextcloud:
        icon: nextcloud.png
        href: https://bailey-malbrough-xps-8700.tailbe32a5.ts.net
        description: Files, photos, calendar
        server: xps
        container: nextcloud-aio-nextcloud
        widget:
          type: nextcloud
          url: https://bailey-malbrough-xps-8700.tailbe32a5.ts.net
          key: "{{HOMEPAGE_VAR_NEXTCLOUD_TOKEN}}"
    - Nextcloud AIO:
        icon: nextcloud-blue.png
        href: https://192.168.68.56:8080
        description: AIO admin, updates and backups
        server: xps
        container: nextcloud-aio-mastercontainer

- Security:
    - Wazuh:
        icon: wazuh.png
        href: https://192.168.68.56:5601
        description: SIEM · alerts and vulnerability detection
        siteMonitor: https://192.168.68.56:5601
```

### `config/widgets.yaml`
```yaml
- resources:
    label: XPS 8700
    cpu: true
    memory: true
    disk: /
    uptime: true
- datetime:
    text_size: xl
    format:
      dateStyle: medium
      timeStyle: short
- search:
    provider: duckduckgo
    target: _blank
```
To add weather later, put an `openmeteo` block here with your home's latitude/longitude, `timezone: America/Chicago` and `units: imperial`.

### `config/bookmarks.yaml`
```yaml
- Admin:
    - Tailscale Console:
        - abbr: TS
          href: https://login.tailscale.com/admin
    - Deco:
        - abbr: DC
          href: http://192.168.68.1
    - GS108Ev4 Switch:
        - abbr: SW
          href: http://192.168.68.52
```

## Future tiles (add to `services.yaml` as each roadmap step lands)
```yaml
- Management:
    - Portainer:
        icon: portainer.png
        href: https://192.168.68.56:9443
        server: xps
        container: portainer
        widget:
          type: portainer
          url: https://192.168.68.56:9443
          env: 1
          key: "{{HOMEPAGE_VAR_PORTAINER_KEY}}"     # Portainer → My account → Access tokens
    - Uptime Kuma:
        icon: uptime-kuma.png
        href: http://192.168.68.56:3001
        widget:
          type: uptimekuma
          url: http://192.168.68.56:3001
          slug: homelab                           # status page slug
    - Nginx Proxy Manager:
        icon: nginx-proxy-manager.png
        href: http://192.168.68.56:81
        widget:
          type: npm
          url: http://192.168.68.56:81
          username: "{{HOMEPAGE_VAR_NPM_USER}}"
          password: "{{HOMEPAGE_VAR_NPM_PASS}}"
    - Vaultwarden:
        icon: vaultwarden.png
        href: https://vault.bailey-lab.duckdns.org
        server: xps
        container: vaultwarden
    - Home Assistant:
        icon: home-assistant.png
        href: http://<ha-ip>:8123
        widget:
          type: homeassistant
          url: http://<ha-ip>:8123
          key: "{{HOMEPAGE_VAR_HA_TOKEN}}"         # HA profile → Long-lived access tokens
```
Add a `Management:` entry to `settings.yaml` → `layout` when you use this group.

## Verification
```bash
docker ps --format '{{.Names}}  {{.Status}}' | grep homepage
sudo ss -tlnp | grep 3000
docker logs --tail 20 homepage          # no "error:" lines
```
- [x] Every tile has a green dot (Tailscale has no dot because it isn't a container; its widget data is the check)
- [x] Pi-hole, Tailscale, Jellyfin and Nextcloud widgets show data
- [ ] Opens from the Pixel 8 on cellular via `http://bailey-malbrough-xps-8700:3000`
- [ ] Add an Uptime Kuma HTTP monitor for `http://192.168.68.56:3000` once Kuma is installed

## Troubleshooting (all hit during the Nov 15 deploy)
- **"Host validation failed":** the address in the browser isn't in `HOMEPAGE_ALLOWED_HOSTS`. Add it with `:3000`, then `docker compose up -d`.
- **Pi-hole "API Error", log says `Failed to login to Pi-Hole API, status: 401`:** the app password was copied but never activated. Test the key without printing it:
  ```bash
  source <(grep PIHOLE_KEY ~/homepage/.env)
  curl -s -X POST http://192.168.68.56:8081/api/auth -H 'Content-Type: application/json' \
    -d "{\"password\":\"$HOMEPAGE_VAR_PIHOLE_KEY\"}" | grep -o '"valid":[a-z]*'
  ```
  `"valid":false` means you need to generate a new one, click **Enable new app password**, and update `.env`.
- **Jellyfin "API Error", log shows `HTTP Error 404 calling .../emby/Sessions`:** the widget is using old endpoints. Add `version: 2`. Check with `curl -s -o /dev/null -w '%{http_code}\n' http://192.168.68.56:8096/emby/System/Info/Public` (404 = needs v2).
- **Nextcloud "API Error", log shows `ENETUNREACH fd7a:...:443` / `HTTP Error 500`:** the container can't reach the ts.net address from Docker's bridge network. Fixed with `network_mode: host`. Test the token directly:
  ```bash
  curl -s -H "NC-Token: $(grep NEXTCLOUD_TOKEN ~/homepage/.env | cut -d= -f2)" \
    "https://bailey-malbrough-xps-8700.tailbe32a5.ts.net/ocs/v2.php/apps/serverinfo/api/v1/info?format=json" | head -c 200
  ```
  `"status":"ok"` = the token is good.
- **Gray dashes on a widget:** it's still loading. Nextcloud's first response is slow. Hard refresh with Ctrl+F5.
- **Tailscale `HTTP Error 404 calling .../device/`:** the device ID is empty or wrong. After about 90 days, an error here means the token expired.
- **Check a key loaded without printing it:** `docker exec homepage printenv HOMEPAGE_VAR_<NAME> | wc -c` (more than 1 = loaded).
- **Wazuh dot turns red while it's actually up:** the self-signed certificate can trip `siteMonitor`. Swap it for `server: xps` + `container: <name from docker ps | grep -i wazuh>`.

## Wall tablet idea
The Galaxy Tab S6 Lite (planned HA wall panel) can show this dashboard: open `http://192.168.68.56:3000` in Chrome → **Add to Home screen**, or use Fully Kiosk Browser.

## Portfolio write-up
- Screenshot the full dashboard (blur the Tailscale IPs if posting publicly).
- Commit `docker-compose.yml` and `config/*.yaml` to the `HomeLab` repo. `.env` stays out via `.gitignore`.
- Talking points:
  - single pane of glass
  - secrets kept in environment variables instead of hard-coded, with Bitwarden as the source of truth
  - read-only Docker socket
  - least-privilege, revocable API tokens (Pi-hole app password, a dedicated Jellyfin key, Nextcloud serverinfo token)
  - scheduled rotation of the expiring Tailscale token
  - troubleshooting from logs: reading 401 vs 404 vs ENETUNREACH to find three different root causes

## To-do
- [ ] Update Tailscale on the XPS, then run the verification block in `tailscale-setup.md`
- [ ] Look into the 49% Pi-hole block rate
- [ ] Optional: hide the Jellyfin counts row and add a weather widget
- [ ] Rotate the Tailscale API token around Feb 8–13, 2027 (reminder scheduled)
- [ ] Add Portainer / Uptime Kuma / NPM / Vaultwarden tiles as each roadmap step lands
- [ ] After NPM: add a `home.` proxy host and update `HOMEPAGE_ALLOWED_HOSTS`
