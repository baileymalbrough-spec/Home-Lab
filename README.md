# Home Lab

My self-hosted home lab: an always-on Ubuntu server running containerized services, behind a mesh network that I rewired and racked myself. Each project below explains why I built it, how it's configured, and the problems I ran into and solved along the way.

## Lab at a glance
- **Server:** Dell XPS 8700 running Ubuntu Linux, managed headless over SSH
- **Containers:** Docker / Docker Compose (Pi-hole, Jellyfin, Nextcloud All-in-One)
- **Network:** TP-Link Deco X10 (AX1500) mesh (3 nodes) as the router, ISP gateway in bridge mode, Netgear GS108E switch, 10" mini server rack
- **Remote access:** Tailscale (WireGuard) subnet router and exit node, with Tailscale Serve/Funnel for HTTPS

## Projects

| Project | What it does | Skills |
|---|---|---|
| [Pi-hole network DNS](pihole-setup.md) | Network-wide ad and tracker blocking for every device; migrated from Windows to Linux without losing config | DNS, DHCP, Docker, Linux, migration |
| [Tailscale remote access](tailscale-setup.md) | Secure access to the home network and ad-blocking DNS from anywhere; HTTPS publishing for lab services | VPN/WireGuard, routing, DNS, TLS, systemd |
| [Jellyfin live TV](jellyfin-live-tv-setup.md) | One live TV channel lineup streamed to every TV in the house, and shared securely with a remote user | Docker, media streaming, hardware transcoding, access control |
| [Nextcloud private cloud](nextcloud-setup.md) | Self-hosted file sync, document editing, chat and whiteboard, reachable only over the tailnet | Docker, reverse proxy, TLS, port management, troubleshooting |
| [Network rack and rewire](network-rack-setup.md) | Consolidated networking gear, removed double NAT, and isolated a throughput ceiling one variable at a time | Network design, cabling, performance testing, troubleshooting |

## Highlights
- **Root-caused a whole-house outage:** after a server reboot, DNS failed for every device. I traced it to a systemd-resolved port conflict with Pi-hole, fixed it, and added a post-update verification checklist.
- **Resolved a three-way port conflict to add a new service:** Nextcloud wouldn't start because Pi-hole held ports 80/443 and Tailscale Funnel held 443 for Jellyfin. I mapped every listener with `ss`, moved Pi-hole's web UI, put Nextcloud behind a reverse proxy on a localhost port, and split HTTPS between services by port, all without interrupting house DNS.
- **Diagnosed a "device up, service down" failure:** a switch answered pings but not its admin page. I used ping-versus-port testing to isolate the cause, then recovered it with a factory reset.
- **Migrated a live service with zero downtime:** I moved Pi-hole to a new host, tested it there, and only then cut DNS over in DHCP.

## Tools
Ubuntu · Docker / Docker Compose · Pi-hole · Tailscale (Serve / Funnel) · Jellyfin · Nextcloud AIO · systemd · SSH · PowerShell · Wireshark · Fing · Advanced IP Scanner · TP-Link Deco · Netgear GS108E

_Addresses, hostnames and identifiers in these docs have been replaced with placeholders._
