# Pi-hole Network DNS

Network-wide DNS ad and tracker blocking for every device in the house, running in Docker on an always-on Ubuntu server.

## Overview
- **Host:** Ubuntu server (Dell XPS 8700), Pi-hole in Docker with host networking
- **Clients:** every device on the network. The Deco's DHCP server hands out Pi-hole as the DNS server.
- **Remote clients:** devices on Tailscale use it from anywhere (see [Tailscale remote access](tailscale-setup.md))

## Background: why I moved it off Windows
I first ran Pi-hole in Docker Desktop on my Windows PC. Blocking worked, but there were two problems:
- **No per-device visibility.** Docker Desktop's bridge network hid the real client IPs. Every query appeared to come from one internal Docker address (`172.18.0.x`).
- **Single point of failure.** DNS for the whole house went down whenever the Windows PC slept or rebooted.

The fix was to migrate it to an always-on Linux server using host networking.

## Migration plan (zero downtime)
1. Build and test the new Pi-hole on Ubuntu while the old one keeps serving the house
2. Copy all settings across with Pi-hole's **Teleporter** export/import
3. Verify blocking on the new server directly
4. Only then point the router's DHCP DNS at the new server
5. Retire the old instance once clients show up on the new dashboard

## Server setup

### Remote management
Installed `openssh-server` and manage the server headless over SSH from my Windows PC.

### Install Docker
```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER   # log out and back in afterwards
```

### Free port 53 from systemd-resolved
Ubuntu's built-in stub resolver already listens on port 53.
```bash
sudo sed -i 's/^#\?DNSStubListener=.*/DNSStubListener=no/' /etc/systemd/resolved.conf
sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
sudo systemctl restart systemd-resolved
```

### Never sleep (it's the house's DNS server)
```bash
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```

### Compose file
`~/pihole/docker-compose.yml`
```yaml
services:
  pihole:
    container_name: pihole
    image: pihole/pihole:latest
    network_mode: host
    environment:
      TZ: 'America/Chicago'
      FTLCONF_webserver_api_password: '<redacted>'
      FTLCONF_dns_listeningMode: 'ALL'   # 'LOCAL' originally; 'ALL' to answer Tailscale clients
      FTLCONF_webserver_port: '8081'     # frees 80/443 for other services
    volumes:
      - './etc-pihole:/etc/pihole'
    cap_add:
      - NET_ADMIN
      - SYS_NICE
    restart: unless-stopped
```
- `network_mode: host` gives real client IPs, so no port mappings are needed.
- Settings controlled by `FTLCONF_` variables show as locked in the web UI, so they're changed in this file.
- **Web UI on 8081:** with host networking, Pi-hole v6's web server claims ports 80 and 443 by default. I moved it to 8081 when adding [Nextcloud](nextcloud-setup.md). DNS on port 53 is unaffected.

### Settings migration
**Settings → Teleporter → Export** on the old instance, then **Import** on the new one, then **Tools → Update Gravity**. Teleporter carries over config, blocklists, clients and groups, but not query history.

## Router (Deco) settings
- **DHCP → Primary DNS:** the Pi-hole server's IP
- **Address reservation** for the server, so its IP never changes
- **IPv6 off**, so devices can't get an ISP IPv6 DNS server and bypass Pi-hole

## Results
After the move, the dashboard showed real per-device stats, which wasn't possible on Windows:

| Device | Queries | Blocked |
|---|---|---|
| Windows PC | 136 | 20 |
| LG smart TV | 128 | 40 (~31%) |
| Android phone | 17 | 1 |
| Network printer | 4 | 0 |

- **The smart TV was the noisiest device on the network.** Its top blocked domain was an ACR (automatic content recognition) viewing tracker, along with Nielsen and Conviva analytics.
- I added an extra blocklist on top of the default (~77.8k domains total).

## Verification
```powershell
nslookup doubleclick.net <pihole-ip>   # should return 0.0.0.0
ipconfig /all                           # DNS Servers should show the Pi-hole IP
```

## Troubleshooting
- **"Could not resolve host" on Ubuntu during setup:** the server still held a stale ISP IPv6 DNS server from before IPv6 was turned off on the router. Fixed by restarting networking (`nmcli networking off && nmcli networking on`) and confirming with `resolvectl status`.
- **Streaming app buffers forever:** some blocked analytics domains (e.g. Conviva) can break playback. Allowlist the domain if that happens.
- **Another service can't bind port 80 or 443:** Pi-hole's web server holds them by default. `sudo ss -tlnp` shows `pihole-FTL` as the owner. Fixed with `FTLCONF_webserver_port`.
- **Whole-house DNS outage after a reboot:** see the Tailscale doc. An OS update re-enabled the systemd-resolved stub listener and blocked Pi-hole from binding port 53.

## Next improvements
- Conditional forwarding so the dashboard shows device names instead of IPs
- A secondary DNS server so one machine going down doesn't take out the whole house
