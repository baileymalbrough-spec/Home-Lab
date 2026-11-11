# Nextcloud Private Cloud

A self-hosted private cloud (file sync, document editing, chat and whiteboard) running on the home lab server with Nextcloud All-in-One (AIO). It's reachable only over my Tailscale network, with a real HTTPS certificate and no ports open to the internet.

## Overview
- **Host:** the Ubuntu server (Dell XPS 8700), in Docker alongside Pi-hole and Jellyfin
- **Install method:** Nextcloud All-in-One, a mastercontainer that deploys and updates the rest of the stack
- **Access:** `https://<server>.<tailnet>.ts.net`, tailnet devices only
- **TLS:** handled by Tailscale Serve, with a certificate issued for the server's MagicDNS name
- **Add-ons:** Nextcloud Office (Euro-Office), Talk, Whiteboard, Imaginary (image previews)

## Architecture
```
Tailnet device ──HTTPS──▶ tailscale serve :443 ──HTTP──▶ 127.0.0.1:11000 (AIO Apache) ──▶ Nextcloud
LAN admin PC   ──HTTPS──▶ <server-ip>:8080 (AIO management interface)
```
AIO runs in **reverse-proxy mode**: it doesn't terminate TLS itself, and its web server binds to localhost only, so the only way in is through Tailscale.

## The problem: three services wanted the same ports
The standard AIO install failed at the first step:
```
failed to bind host port 0.0.0.0:80/tcp: address already in use
```
Mapping every listener with `sudo ss -tlnp | grep -E ':(80|443|8080|8443)\b'` turned up two conflicts:
1. **Pi-hole v6** runs with host networking, and its built-in web server claims **80 and 443** by default.
2. **Tailscale Funnel** was already publishing Jellyfin on **443** of the server's tailnet address. This blocks any container from publishing `0.0.0.0:443`, which is why AIO's domain check then failed with "Domaincheck container is not running."

## The fix

### 1. Move Pi-hole's web UI off 80/443
Added one variable to Pi-hole's compose file and recreated the container:
```yaml
      FTLCONF_webserver_port: '8081'   # not 8080, which AIO's admin interface needs
```
I confirmed DNS stayed up throughout (`dig @<server-ip> doubleclick.net` still returned `0.0.0.0`). Only the admin page moved.

### 2. Split HTTPS between services by port
The server has one tailnet hostname, so each service gets its own port. Funnel (public) only supports 443, 8443 and 10000, so Jellyfin moved to 8443 and Nextcloud took 443 as tailnet-only:
```bash
sudo tailscale serve reset
sudo tailscale funnel --bg --https=8443 http://127.0.0.1:8096    # Jellyfin, public
sudo tailscale serve  --bg --https=443  http://127.0.0.1:11000   # Nextcloud, tailnet only
tailscale serve status
```

### 3. Run AIO in reverse-proxy mode
```bash
sudo docker run -d \
  --init \
  --sig-proxy=false \
  --name nextcloud-aio-mastercontainer \
  --restart always \
  --publish 8080:8080 \
  --env APACHE_PORT=11000 \
  --env APACHE_IP_BINDING=127.0.0.1 \
  --env SKIP_DOMAIN_VALIDATION=true \
  --volume nextcloud_aio_mastercontainer:/mnt/docker-aio-config \
  --volume /var/run/docker.sock:/var/run/docker.sock:ro \
  ghcr.io/nextcloud-releases/all-in-one:latest
```
- **`APACHE_PORT` / `APACHE_IP_BINDING`:** Nextcloud listens only on `127.0.0.1:11000`, behind Tailscale Serve.
- **`SKIP_DOMAIN_VALIDATION`:** AIO validates the domain from the public internet, which can't reach a tailnet-only name.
- **No ports 80 or 8443 published:** those are only needed when AIO requests its own Let's Encrypt certificate.

### 4. Split-horizon DNS in Pi-hole
After setup, one desktop got `ERR_CONNECTION_CLOSED`. `nslookup` showed the server's MagicDNS name resolving to **Tailscale's public Funnel relays**, not the server. Funnel publishes the hostname in public DNS, and that desktop uses Pi-hole directly with Tailscale DNS off. The relay only forwards the Funnel port (8443), so it dropped the 443 connection to Nextcloud.

The fix was local DNS records in Pi-hole that answer the name with the server's tailnet IPv4 and IPv6 addresses:
```bash
sudo docker exec pihole pihole-FTL --config dns.hosts '["<server-tailscale-ipv4> <server>.<tailnet>.ts.net", "<server-tailscale-ipv6> <server>.<tailnet>.ts.net"]'
sudo docker restart pihole    # clear cached upstream answers
```
Devices on the home DNS now reach Nextcloud directly over the tailnet, while public users still get the Funnel relay for Jellyfin.

### 5. Initial configuration
In the AIO interface I set the domain to the server's MagicDNS name, chose the add-ons, and started the containers. Credentials went straight into a password manager.

**Office suite choice:** I picked Euro-Office over Collabora. It has better fidelity with Word, Excel and PowerPoint files, the formats I actually use, and it does more of its rendering in the browser, which is easier on older server hardware. AIO can switch between the two later.

## Result
| Container | Role |
|---|---|
| Mastercontainer | AIO management, updates and backups |
| Apache & Caddy | web server on localhost:11000 |
| Nextcloud | application |
| PostgreSQL / Redis | database and cache |
| Client Push | instant sync notifications |
| Euro-Office | document editing |
| Talk / Whiteboard / Imaginary | chat and calls, whiteboard, image previews |

## Final port map on the server
| Port | Service | Exposure |
|---|---|---|
| 53 | Pi-hole DNS | LAN + tailnet |
| 8081 | Pi-hole admin | LAN + tailnet |
| 8096 | Jellyfin | LAN + tailnet |
| 8080 | AIO admin interface | LAN |
| 127.0.0.1:11000 | Nextcloud | localhost only |
| 443 (tailnet IP) | Tailscale Serve → Nextcloud | tailnet only |
| 8443 (tailnet IP) | Tailscale Funnel → Jellyfin | public |

## Troubleshooting and lessons
- **A failed `docker run` still leaves a container behind.** Remove it (`docker rm`) before retrying, or you'll get a name conflict.
- **AIO without `-d` stays attached to the terminal, and Ctrl+C doesn't detach** (because of `--sig-proxy=false`). Commands typed afterward go into the log stream and never run. Use `-d` and `docker logs -f` instead.
- **Verify with `tailscale serve status`, not assumptions.** The first time, the Tailscale commands never actually executed, and the Nextcloud address still pointed at Jellyfin.
- **Funnel changes public DNS for the whole hostname.** Clients that don't use Tailscale DNS need a local override, or tailnet-only services on that name break.
- **Pi-hole merges cached upstream answers with a new local record** until the cache is cleared. A non-authoritative `nslookup` answer means the local record isn't being used.
- **Moving a public service's port breaks existing links.** Jellyfin's remote URL changed to include `:8443`, so I updated it for the remote user.

## Verification
```bash
sudo docker ps --format '{{.Names}}: {{.Status}}'   # all nextcloud-aio-* containers healthy
tailscale serve status                             # :443 → 11000 (tailnet only), :8443 → 8096 (Funnel)
sudo ss -tlnp | grep -E ':(80|443|8080|8081)\b'     # no unexpected listeners
nslookup <server>.<tailnet>.ts.net                  # only the server's tailnet IPs
```

## Next improvements
- Automated AIO (Borg) backups to a separate drive, with a tested restore
- A non-admin daily user and 2FA on the admin account
- Moving the data directory to a dedicated drive
