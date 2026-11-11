# Network Rack and Rewire

I consolidated my home networking gear into a 10" mini server rack and rewired the network to remove double NAT, so every wired device sits behind one router. Afterward I inventoried the network and worked through a methodical speed investigation on a 1 Gig internet plan.

## Overview
- **Rack:** TecMojo 10" mini server rack
- **Router:** TP-Link Deco X10 (AX1500) mesh, 3 nodes, in router mode
- **ISP gateway:** Cox Panoramic, in bridge mode
- **ISP plan:** 1 Gig (about 1000 Mbps down / 35 Mbps up)
- **Switch:** Netgear GS108Ev4 8-port managed switch

## Problems I was solving
- **Double NAT:** the ISP gateway and the Deco were both routing. This was the likely cause of recurring IPv6 prefix delegation failures in the router logs.
- **Messy topology:** wired devices plugged in wherever there was a free port, so some could end up outside the Deco network, missing DNS filtering and network segmentation.

## Topology
```
ISP gateway (bridge mode)
   │  WAN
Deco main node (router, DHCP)
   │  Deco LAN → switch uplink
Netgear GS108Ev4
   ├─ Windows PC
   ├─ Ubuntu server (Pi-hole, Jellyfin, Nextcloud, Tailscale)
   ├─ 3rd PC
   └─ spare ports
Deco satellite nodes (wireless backhaul)
```
- **The Deco has only one LAN port,** so the switch hangs off it and every wired device goes through the switch.
- **Only the Deco connects to the ISP gateway.** Anything plugged into the gateway directly would bypass the router, the DNS filtering and the guest/IoT segmentation.
- **The Deco stays in router mode** so the guest/IoT network keeps working.
- **Satellites take addresses at the top of the `/22` range.** That looked odd at first but is normal Deco behavior.

## Rack layout (top to bottom)
1. **Deco on top of the frame,** not inside it, because metal rails weaken Wi-Fi
2. **Switch on a 1U shelf** (it's a desktop switch, so it won't bolt to the rails)
3. **ISP gateway on the bottom shelf,** with nothing stacked on it because it runs warm
4. **Power at the bottom**

## Cutover order
1. Put the ISP gateway in bridge mode and confirm the Deco gets a public IP on its WAN
2. Move the cables, color-coded by role (WAN, uplink, devices) and labeled at both ends
3. Create DHCP reservations for the router, switch and servers so their addresses never move

## Network inventory
I exported a full device scan with Fing and reconciled it against the Deco's client list:
- **11 active devices:** 3 mesh nodes, the switch, 2 wired PCs/servers, a smart TV, a printer, a phone, and 2 IoT devices
- **Stale entries cleaned up:** offline devices and phantom "new devices" caused by randomized private Wi-Fi MACs. A MAC whose second hex digit is 2, 6, A or E is locally administered, so phones keep appearing as new devices.
- **A mislabeled server:** Fing called the Ubuntu server a Windows laptop named after Pi-hole, because Pi-hole answers reverse DNS lookups for its own IP.
- **Segmentation follow-ups:** IoT devices found on the main network are being moved to the Deco's IoT network. The smart TV stays on the main network because it needs to reach Jellyfin.

## Speed investigation
**Finding:** wired speeds top out around **710–760 Mbps down** on a 1 Gig plan, so the cap is upstream of the PCs.

| Device | Tool | Down | Up | Notes |
|---|---|---|---|---|
| Phone (before the rewire) | Fing | 759 Mbps | 11 Mbps | Double NAT still in place |
| Windows PC, wired | fast.com | 710 Mbps | 120 Mbps | 22 ms idle / 27 ms loaded latency |
| Ubuntu server, wired | `speedtest-cli` (Python) | 424 Mbps | 62 Mbps | **Discarded:** distant server, and the tool is CPU-bound |
| Ubuntu server, wired | Ookla CLI 1.2.0 | 734 Mbps | 33 Mbps | Nearby server, 0% packet loss |

**Ruled out, one variable at a time:**
- **Switch and cabling:** every link negotiates at gigabit. A 100 Mbps link would cap around 94 Mbps.
- **The PCs:** two different wired machines hit the same ceiling.
- **DNS / Pi-hole:** DNS affects how fast connections start, not throughput.
- **Congestion:** the result is the same across days and times of day.
- **Router features:** QoS and the security scanning feature are off, and the firmware is current.
- **Double NAT:** the same cap appeared before and after bridge mode.

**Tool choice mattered.** The Python `speedtest-cli` under-reported by about 40% on the older server CPU. The official Ookla CLI is the reliable repeat test:
```bash
cd /tmp
curl -LO https://install.speedtest.net/app/cli/ookla-speedtest-1.2.0-linux-x86_64.tgz
tar xzf ookla-speedtest-1.2.0-linux-x86_64.tgz
./speedtest
```
Upload of about 33 Mbps is in line with the plan. fast.com's 120 Mbps upload figure was an overestimate.

**Next step: a bypass test** to split the remaining suspects. I'll connect a PC directly to the ISP gateway, power-cycle it so the PC gets the public IP, and test.
- **900+ Mbps direct:** the Deco X10 is the bottleneck, and a higher-end tri-band mesh would fix it.
- **About 700 Mbps direct:** the issue is upstream, so I'd escalate to the ISP with the modem's signal and error statistics.

**Wi-Fi expectations:** the X10 realistically delivers 400–600 Mbps near the main node, and satellites on wireless backhaul usually get 100–250. MoCA isn't an option with a single coax jack, so the renter-friendly backhaul upgrades are powerline adapters or a tri-band mesh with a dedicated backhaul radio.

## Troubleshooting: switch admin page unreachable after the rewire
**Symptom:** after the rewire, the switch's web admin page timed out. Devices plugged into the switch still had normal connectivity.

**Steps:**
1. **Fing** showed the switch as last online a month ago. That was stale cached data, not its current state.
2. **Advanced IP Scanner** found the switch's MAC alive at its expected IP. So the addressing hadn't changed and the Deco was still the DHCP server.
3. **NETGEAR's Switch Discovery Tool** hung after the UAC prompt. That's a known issue on PCs with many virtual network adapters (Docker, WSL and Hyper-V vEthernet).
4. **Isolated the failure with:**
   ```powershell
   Test-NetConnection <switch-ip> -Port 80
   ```
   Ping succeeded but TCP 80 failed. So the device was up but its management web server wasn't answering. That ruled out both a network problem and the browser forcing HTTPS.
5. **Fix:** factory reset the switch. Port 80 came back once it finished booting. I logged in, set a new admin password, and reserved its IP.

**Takeaway:** on this switch line, the management plane can lock up while hardware switching keeps working. Testing ping against the specific service port separates "device down" from "service down" without guessing.

## Related work
- Earlier, I diagnosed recurring TV Wi-Fi drops with Wireshark captures and Deco system logs. I ruled out RF interference, channel choice and the clients themselves, and traced the drops to a radio/AP-sync error in the mesh firmware.
- VLAN trunking wasn't practical on this switch and router combination, so I use the Deco's guest/IoT network for segmentation instead.

## Next improvements
- Run the bypass test and resolve the ~730 Mbps ceiling (router upgrade or ISP escalation)
- Finish moving IoT devices to the IoT network
- A mini UPS so internet and DNS ride through short power blips
- Wired backhaul to a satellite node to stabilize the mesh
