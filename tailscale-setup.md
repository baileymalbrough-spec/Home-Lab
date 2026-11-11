# Tailscale Remote Access

Secure remote access to my home network from anywhere, with ad-blocking DNS following my devices off-network. It's built on Tailscale (WireGuard), with the home lab server acting as subnet router and exit node.

## Overview
- **Subnet router and exit node:** the Ubuntu server (Dell XPS 8700), which also runs Pi-hole, Jellyfin and Nextcloud
- **DNS:** all tailnet devices use Pi-hole, so ad blocking works on cellular and public Wi-Fi too
- **Clients:** Android phone, Android tablet, Windows desktops, laptop
- **No open ports:** nothing is port-forwarded on the router, because Tailscale handles NAT traversal
- **HTTPS for lab services:** Tailscale Serve (tailnet only) and Funnel (public) with certificates for the server's MagicDNS name

## Three rules that keep it from breaking
1. **On the Pi-hole host, Tailscale DNS is off** (`tailscale set --accept-dns=false`). Otherwise the server sends its own DNS through Tailscale back to itself and loops.
2. **On the Pi-hole host, the systemd-resolved stub listener is off,** so Pi-hole can bind port 53 on all interfaces.
3. **On every device that's ever on home Wi-Fi, "Use Tailscale subnets" is off.** Otherwise LAN traffic loops into the tunnel and you get "connected, no internet."

## Server setup

### Install and advertise routes
```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --advertise-routes=<lan-subnet> --advertise-exit-node --ssh --accept-dns=false
```

### IP forwarding (persistent)
`/etc/sysctl.d/99-tailscale.conf`
```
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
```
```bash
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
```

### UDP GRO tuning for forwarding performance (persistent)
`/etc/systemd/system/tailscale-gro.service`
```ini
[Unit]
Description=Tailscale UDP GRO tuning
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/sbin/ethtool -K <interface> rx-udp-gro-forwarding on rx-gro-list off

[Install]
WantedBy=multi-user.target
```
```bash
sudo systemctl enable --now tailscale-gro.service
```

### Free port 53 for Pi-hole
`/etc/systemd/resolved.conf.d/pihole.conf`
```ini
[Resolve]
DNSStubListener=no
```
```bash
sudo systemctl restart systemd-resolved
docker restart pihole
```

### Serve and Funnel
The server has one MagicDNS name, so each published service gets its own HTTPS port:

| URL | Backend | Exposure |
|---|---|---|
| `https://<server>.<tailnet>.ts.net` | Nextcloud on `127.0.0.1:11000` | tailnet only (Serve) |
| `https://<server>.<tailnet>.ts.net:8443` | Jellyfin on `127.0.0.1:8096` | public (Funnel) |

```bash
sudo tailscale funnel --bg --https=8443 http://127.0.0.1:8096
sudo tailscale serve  --bg --https=443  http://127.0.0.1:11000
tailscale serve status
```
- Requires MagicDNS and HTTPS certificates enabled in the admin console. Funnel only supports ports 443, 8443 and 10000.
- Serve/Funnel binds those ports on the server's tailnet address, which blocks a container from publishing the same port on `0.0.0.0`. That's why [Nextcloud](nextcloud-setup.md) runs behind Serve on a localhost port.

## Admin console
- Approved the advertised subnet route and **exit node** for the server
- **Disabled key expiry** on the server, so it never drops off the tailnet
- **DNS:** the server's Tailscale IP set as the global nameserver, with **Override DNS servers** on and **MagicDNS** on
- **HTTPS certificates** and **Funnel** enabled for Serve/Funnel

## Pi-hole change
Changed Pi-hole's `FTLCONF_dns_listeningMode` from `LOCAL` to `ALL` so it answers queries from Tailscale (100.x) addresses. This is safe because port 53 is not forwarded on the router, so nothing outside the LAN or the tailnet can reach it.

## Client settings
- **Phones and tablets:** "Use Tailscale subnets" off in the Tailscale app
- **Windows:** `tailscale set --accept-routes=false`
- **The server is reached by its MagicDNS name,** which works at home and away without subnet routes. Subnet routes are only turned on temporarily when I need other LAN devices from outside the house.

## Verification
```bash
tailscale status
dig @<server-lan-ip> google.com +short
dig @<server-tailscale-ip> google.com +short
sudo ss -lunp | grep ':53 '                  # only pihole-FTL, Recv-Q should be 0
docker ps --format '{{.Names}}  {{.Status}}' # pihole should be (healthy)
sysctl net.ipv4.ip_forward                   # should be 1
tailscale serve status                       # :443 → Nextcloud (tailnet), :8443 → Jellyfin (Funnel)
```
I confirmed the setup from a phone on cellular: the Pi-hole admin page loaded over Tailscale, and Pi-hole's query log showed the phone's Tailscale IP.

## Troubleshooting

### Whole-house outage after a server reboot
- **Symptoms:** every device in the house showed "connected, no internet." The Pi-hole container was `(unhealthy)`, `dig` timed out, and `ss` showed ~200,000 queued packets on port 53.
- **Root cause:** an Ubuntu update re-enabled systemd-resolved's stub listener. With `listeningMode: ALL`, Pi-hole's DNS engine couldn't bind port 53 (`failed to create listening socket ... Address in use`). It held a socket it never read from.
- **Fix:** disabled the stub listener with a drop-in config file (so updates won't revert it), and turned off Tailscale DNS on the server.
- **Lessons:**
  - I now run the verification commands after every update or reboot.
  - I documented an emergency bypass: point the router's DHCP DNS at a public resolver until Pi-hole is fixed.

### No internet on home Wi-Fi with Tailscale on
- **Cause:** a routing loop. The device sent LAN traffic, including its WireGuard connection to the server, into the Tailscale tunnel while it was also directly on that LAN.
- **Fix:** turned off "Use Tailscale subnets" on devices that live on the home network.

### Other notes
- **"TUN device tailscale0 is busy":** a second tailscaled instance was holding the interface. A reboot cleared it. If it comes back, `lsof /dev/net/tun` finds the culprit.
- **Relay check:** `tailscale ping <server>` shows whether a connection is peer-to-peer or relayed through DERP.

## Next improvements
- A secondary DNS server, so one machine going down doesn't take DNS down for the whole house
