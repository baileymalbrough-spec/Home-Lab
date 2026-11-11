# Jellyfin Live TV

One live TV channel lineup available on every TV in the house, without a cable box. It runs in Docker on the home lab server.

## Overview
- **Server:** Jellyfin in Docker on the Ubuntu server, port `8096`
- **Source:** free, publicly available streams from the iptv-org project (M3U playlist)
- **Clients:** Jellyfin apps on an LG webOS TV and a TCL TV, any web browser, and one remote user outside the house
- **Remote access:** a public HTTPS link through Tailscale Funnel on port `8443` (see [Sharing with a remote user](#sharing-with-a-remote-user))

## Why Jellyfin instead of VLC
VLC can't share a playlist across the network on its own, and it **isn't available on LG webOS or Roku**, so it couldn't reach every TV. Jellyfin runs as a server and has native apps for LG, Google/Android TV, Roku and Fire TV. It also gives a channel list, a guide view and favorites.

## Playlist choice
The iptv-org master list (`index.m3u`) has 10,000+ worldwide channels, many of them dead, geo-blocked or in other languages, which makes scanning slow and the guide unusable. I used the English-language list instead:
```bash
mkdir -p ~/jellyfin/{config,cache} ~/iptv
curl -o ~/iptv/tv.m3u https://iptv-org.github.io/iptv/languages/eng.m3u
```

## Compose file
`~/jellyfin/docker-compose.yml`
```yaml
services:
  jellyfin:
    image: jellyfin/jellyfin:latest
    container_name: jellyfin
    network_mode: host
    environment:
      - TZ=America/Chicago
    volumes:
      - ~/jellyfin/config:/config
      - ~/jellyfin/cache:/cache
      - ~/iptv:/playlists:ro
    devices:
      - /dev/dri:/dev/dri     # Intel QuickSync hardware transcoding
    restart: unless-stopped
```
```bash
ls /dev/dri                       # card0 and renderD128 confirm the iGPU is available
cd ~/jellyfin && docker compose up -d
```
- **Host networking** lets the TV apps auto-discover the server, and its ports don't conflict with Pi-hole on the same host.
- **Passing `/dev/dri`** uses the CPU's integrated GPU (Intel QuickSync) for transcoding instead of the CPU, so the older hardware can handle more streams.

## Server configuration
1. **First-run wizard:** created an admin user and skipped media libraries, since live TV doesn't need one. Remote connections are allowed but **automatic port mapping is off**, so no router ports are opened. The only public path is the Tailscale Funnel link described below.
2. **Dashboard → Live TV → Tuner Devices:** added an **M3U Tuner** pointing at `/playlists/tv.m3u`
3. **Scheduled Tasks → Refresh Guide:** builds the channel list
4. **Quick Connect** (Dashboard → General): new devices sign in with a 6-character code approved from a signed-in device, instead of typing a password with a TV remote

## Sharing with a remote user
A friend outside my tailnet wanted access to the channel lineup. I compared three options:

| Option | Verdict |
|---|---|
| Share just the server through Tailscale (machine sharing) | Most private, but requires them to install Tailscale |
| **Tailscale Funnel** (public HTTPS link) | **Chosen:** works in any browser or Jellyfin app, with no port forwarding through the ISP's double NAT |
| Port forwarding / Cloudflare Tunnel | Rejected: exposes the home IP; Cloudflare's free plan doesn't allow video streaming |

```bash
sudo tailscale funnel --bg --https=8443 http://127.0.0.1:8096
```
Hardening, since the login page is now public:
- **A dedicated user** with no admin rights, access to Live TV only, an internet bitrate cap of about 4–8 Mbps, 1–2 max sessions, and downloads off
- **Failed-login lockout** (about 5 attempts) on every account, and a long admin password
- **Known proxies = `127.0.0.1`:** Funnel traffic arrives from localhost, so without this Jellyfin treats remote viewers as local and skips their bitrate limit

Funnel originally used port 443. When I added [Nextcloud](nextcloud-setup.md), 443 went to Nextcloud (tailnet only) and Jellyfin moved to 8443, one of the three ports Funnel supports.

## TV clients
| TV | App |
|---|---|
| LG (webOS) | Jellyfin from the LG Content Store |
| TCL (Google TV / Roku) | Jellyfin for Android TV, or the Jellyfin Roku channel |

## Troubleshooting
- **TV app can't find the server:** the TV was on the guest/IoT Wi-Fi, which is isolated from the server by design. It needs to be on the main network.
- **A channel won't play:** public streams come and go, so some are simply dead. If one plays in a browser but not on a TV, enable transcoding under Dashboard → Playback.
- **Buffering on the TVs:** I checked the known mesh backhaul instability first (see [network rack](network-rack-setup.md)) before blaming the streams.
- **Server address must not change:** the TVs connect by IP, so the server has a DHCP reservation.
- **Remote link won't connect:** usually a typo in the hostname (dots instead of hyphens) or a missing `:8443`. `tailscale funnel status` confirms what's published, and the first visit can be slow while the certificate is issued.

## Next improvements
- Nightly script and cron job to refresh the playlist and filter out geo-blocked and part-time channels
- A local DNS name in Pi-hole (e.g. `jellyfin.home`)
- TV guide data (XMLTV) from the iptv-org EPG project
