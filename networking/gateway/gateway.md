# Building a Home Gateway on Linux

How to turn a small Linux box (mini-PC, Raspberry Pi with a USB NIC, or an old x86 machine) into a full home gateway: routing, NAT, firewall, DHCP, DNS, IPv6, network segmentation, Wi-Fi access point, and backup uplinks (Wi-Fi, LTE, satellite).

The goal is to reproduce everything a consumer ISP box does, but in a way you can understand, audit and extend.

**Assumptions:** Debian 13 / Ubuntu 24.04 (systemd ≥ 253), `eth0` = WAN, `eth1` = LAN trunk to a managed switch, `wlan0` = Wi-Fi AP. Your interface names will differ: check with `ip -br link`.

**How to read it:**
- **To learn:** sections 1-3 (what and why), 4 (build it live by hand), then 5-10 (make it permanent), 11 (extra uplinks).
- **To debug:** the packet path in section 7, then 13 (verification) and 14 (troubleshooting).

## Table of contents

- [1. What a home gateway actually does](#1-what-a-home-gateway-actually-does)
   - [Ways to build one](#ways-to-build-one)
   - [Who does what on the gateway](#who-does-what-on-the-gateway)
- [2. Target topology](#2-target-topology)
   - [Networks](#networks)
- [3. Hardware checklist](#3-hardware-checklist)
- [4. Quick start: everything by hand](#4-quick-start-everything-by-hand)
   - [Old commands vs ip](#old-commands-vs-ip)
- [5. Interfaces and VLANs](#5-interfaces-and-vlans)
   - [Method 1: systemd-networkd (recommended)](#method-1-systemd-networkd-recommended)
   - [Method 2: ifupdown (classic Debian)](#method-2-ifupdown-classic-debian)
- [6. Kernel settings](#6-kernel-settings)
- [7. Firewall and NAT](#7-firewall-and-nat)
   - [How a packet crosses the gateway](#how-a-packet-crosses-the-gateway)
   - [Method 1: nftables (recommended)](#method-1-nftables-recommended)
   - [Method 2: iptables (classic)](#method-2-iptables-classic)
   - [Method 3: firewalld (zones)](#method-3-firewalld-zones)
   - [Notes on the rules](#notes-on-the-rules)
- [8. DHCP and DNS (dnsmasq)](#8-dhcp-and-dns-dnsmasq)
- [9. Switch configuration](#9-switch-configuration)
- [10. Wi-Fi access point (hostapd)](#10-wi-fi-access-point-hostapd)
   - [Wi-Fi planning tips](#wi-fi-planning-tips)
- [11. Other WAN uplinks and backup](#11-other-wan-uplinks-and-backup)
   - [Method 1: Wi-Fi station (client)](#method-1-wi-fi-station-client)
   - [Method 2: Cellular modem (Quectel EC25)](#method-2-cellular-modem-quectel-ec25)
   - [Method 3: Satellite (Starlink)](#method-3-satellite-starlink)
   - [Failover](#failover)
- [12. Hardening checklist](#12-hardening-checklist)
- [13. Verification](#13-verification)
- [14. Troubleshooting](#14-troubleshooting)
- [References](#references)

---

## 1. What a home gateway actually does

| Function | What it means | Tool used here |
|---|---|---|
| Routing | Forward packets between the LAN(s) and the internet | Linux kernel (`ip_forward`) |
| NAT (IPv4) | Hide all private LAN addresses behind the single public WAN IP | nftables `masquerade` |
| Stateful firewall | Allow outbound traffic and its replies, drop unsolicited inbound traffic | nftables |
| DHCP server | Hand out IP address, gateway and DNS to clients | dnsmasq |
| DNS resolver / cache | Resolve names for clients, plus local names like `nas.home.arpa` | dnsmasq |
| IPv6 | Get a delegated prefix from the ISP, advertise it on the LAN | systemd-networkd |
| Segmentation | Separate trusted devices, IoT and guests into VLANs | 802.1Q VLANs + bridges |
| Wi-Fi | Access point broadcasting one SSID per network | hostapd |

### Ways to build one

| Approach | Examples | When to pick it |
|---|---|---|
| **DIY Linux** (this note) | Debian/Ubuntu + nftables/iptables + dnsmasq | You want to learn and control every piece |
| **Router OS (Linux)** | OpenWrt, VyOS | Same building blocks, packaged with a UI/CLI. OpenWrt also runs on consumer routers |
| **Router OS (BSD)** | OPNsense, pfSense | Web UI, firewall is `pf` instead of netfilter, many plugins |
| **ISP box / consumer router** | The box you already have | Zero effort, little control |

They all provide the same functions, but not with the same internals: BSD systems use a different network stack and firewall (`pf`), so Linux commands and debugging habits don't carry over 1:1. Each step below lists more than one method when there's a real choice.

### Who does what on the gateway

Each piece of configuration should have **one owner**. Most integration bugs come from two tools fighting over the same thing: two DHCP clients installing default routes, a modem manager rewriting `/etc/resolv.conf` behind dnsmasq, NetworkManager and systemd-networkd both managing `eth0`.

| Component | Owns |
|---|---|
| Kernel | Network devices, IP stack, bridging, routing, forwarding, netfilter |
| `iproute2` (`ip`, `bridge`) | Inspecting and changing all of the above by hand (not persistent) |
| systemd-networkd | Persistent interfaces, VLANs, bridges, addresses, WAN DHCP client, IPv6 RA and prefix delegation |
| nftables | Firewall and NAT policy |
| dnsmasq | DHCP server and DNS forwarder for the LANs |
| hostapd | Wi-Fi access point |
| ModemManager / modem tool | Cellular modem control and data session (section 11) |
| systemd | Starting, ordering and restarting all the services above |

---

## 2. Target topology

```
                 Internet
                    │
             ┌──────┴──────┐
             │  ISP modem  │  (bridge mode / ONT if possible)
             └──────┬──────┘
                    │ eth0 (WAN): public IP via DHCP or PPPoE
                    │ (or Wi-Fi, LTE, Starlink: section 11)
             ┌──────┴──────┐
             │   Gateway   │  Linux: routing, NAT, firewall, DHCP, DNS
             │             │)))  wlan0 (hostapd): one SSID per network
             └──────┬──────┘
                    │ eth1 (LAN trunk): VLAN 10/20/30 tagged
             ┌──────┴──────┐
             │ Managed     │
             │ switch      │
             └─┬────┬────┬─┘
               │    │    │
            PC/NAS  TV  printer
```

### Networks

| VLAN | Name | Subnet | Gateway | Purpose | Can reach |
|---|---|---|---|---|---|
| 10 | LAN | 192.168.10.0/24 | 192.168.10.1 | Trusted devices: laptops, phones, NAS | Internet, IoT |
| 20 | IoT | 192.168.20.0/24 | 192.168.20.1 | TVs, cameras, smart plugs | Internet only |
| 30 | Guest | 192.168.30.0/24 | 192.168.30.1 | Visitors | Internet only, clients isolated |

**VLAN, bridge and subnet are three different things**, mapped 1:1:1 here by choice:

- **VLAN**: a separate Layer 2 broadcast domain, identified by an 802.1Q tag on trunk links.
- **Bridge**: a software switch in the gateway that forwards Ethernet frames between its member ports (`vlan10`, `wlan0`...).
- **Subnet**: the Layer 3 address range (`192.168.10.0/24`) whose gateway IP lives on the bridge.

Where tags exist: **tagged** on the `eth1` ↔ switch trunk. **Untagged** inside the gateway after `vlan10`/`vlan20`/`vlan30` strip them, on switch access ports, and on Wi-Fi. When debugging, `tcpdump -e -ni eth1` shows tags, `tcpdump -ni br-lan` doesn't.

> If the ISP modem can't be put in bridge mode, you'll have double NAT. It still works, but forward the modem's DMZ to the gateway's WAN IP so that port forwards keep working.

---

## 3. Hardware checklist

- **Two NICs minimum**: one WAN, one LAN. Intel NICs (i210/i225/i226) are the safe choice.
- **Managed switch** that supports 802.1Q VLANs, if you want segmentation (strongly recommended).
- **Wi-Fi card** with good AP-mode support (see section 10), plus a second one for 2.4 GHz IoT if needed.
- **CPU**: a recent x86 mini-PC usually routes and NATs 1 Gbit/s with a ruleset like this one. Small ARM boards vary a lot (NIC driver, USB NICs, interrupts, conntrack load). Measure instead of assuming: run `iperf3 -s` on a WAN-side machine and `iperf3 -c <server> -t 30` (add `-R` for the other direction) from a LAN client, and watch the gateway's CPU with `top` during the test.

OS: any Debian/Ubuntu server works. Install the packages:

```bash
sudo apt install nftables dnsmasq hostapd iw iproute2 conntrack tcpdump   # swap nftables for iptables-persistent if using Method 2 in section 7
sudo systemctl enable --now systemd-networkd nftables
# Only one tool may manage the interfaces: disable NetworkManager and ifupdown.
# Ubuntu: move the YAML files out of /etc/netplan. Netplan generates networkd files such as
# /run/systemd/network/10-netplan-eth0.network, which sort before 10-wan.network and take precedence.
```

---

## 4. Quick start: everything by hand

Before writing any config file, you can build the whole gateway live with plain commands (as root). It's the best way to understand each piece, and handy for testing. **Nothing survives a reboot**: sections 5 to 8 make it permanent.

> **Lab only.** Run this on a test machine, from its console or from `br-lan` (the `DROP` policy cuts SSH from anywhere else). Needs `apt install iptables dhcpcd dnsmasq`. Stop `systemd-networkd` and NetworkManager first, so nothing else reconfigures the interfaces underneath you. IPv4 gets a complete policy. IPv6 is simply closed: nothing is forwarded, and the gateway only answers ICMPv6. The firewall uses `iptables` here on purpose, one readable command per rule. The permanent setup in section 7 uses nftables.

```bash
# --- WAN ---
ip link set eth0 up
dhcpcd eth0                                    # or: dhclient eth0
# Static WAN instead of DHCP:
# ip addr add 203.0.113.10/24 dev eth0
# ip route add default via 203.0.113.1

# --- LAN: one VLAN + one bridge per network ---
ip link set eth1 up
for net in "10 lan" "20 iot" "30 guest"; do
    set -- $net                                # $1 = VLAN id, $2 = name
    ip link add link eth1 name vlan$1 type vlan id $1   # same names as section 5
    ip link add name br-$2 type bridge
    ip link set vlan$1 master br-$2
    ip addr add 192.168.$1.1/24 dev br-$2
    ip link set vlan$1 up
    ip link set br-$2 up
done

# --- Routing ---
sysctl -w net.ipv4.ip_forward=1

# --- Firewall + NAT (IPv4): traffic to the gateway itself ---
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -m conntrack --ctstate INVALID -j DROP
iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -p icmp -j ACCEPT
iptables -A INPUT -i eth0 -p udp --sport 67 --dport 68 -j ACCEPT        # WAN DHCP replies (renewals can arrive as broadcast)
iptables -A INPUT -i br-+ -p udp -m multiport --dports 53,67 -j ACCEPT   # DNS + DHCP from the LANs
iptables -A INPUT -i br-+ -p tcp --dport 53 -j ACCEPT
iptables -A INPUT -i br-lan -p tcp --dport 22 -j ACCEPT                  # SSH from the trusted LAN only
iptables -P INPUT DROP                                  # everything else, WAN included: dropped

# --- Firewall + NAT (IPv4): traffic through the gateway ---
iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A FORWARD -m conntrack --ctstate INVALID -j DROP
iptables -A FORWARD -i br-+ -o eth0 -j ACCEPT           # all networks -> internet
iptables -A FORWARD -i br-lan -o br-iot -j ACCEPT       # LAN -> IoT
iptables -P FORWARD DROP                                # anything else between networks: blocked
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

# --- IPv6: closed (no forwarding), only what the gateway itself needs ---
ip6tables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
ip6tables -A INPUT -i lo -j ACCEPT
ip6tables -A INPUT -p ipv6-icmp -j ACCEPT               # neighbor discovery + RAs: IPv6 breaks without it
ip6tables -A INPUT -i eth0 -s fe80::/10 -p udp --sport 547 --dport 546 -j ACCEPT   # WAN DHCPv6 replies (dhcpcd)
ip6tables -P INPUT DROP
ip6tables -P FORWARD DROP

# --- DHCP + DNS (foreground, Ctrl+C to stop; stop the dnsmasq service first) ---
dnsmasq --no-daemon --bind-dynamic \
    --interface=br-lan --interface=br-iot --interface=br-guest \
    --dhcp-range=192.168.10.100,192.168.10.250,12h \
    --dhcp-range=192.168.20.100,192.168.20.250,12h \
    --dhcp-range=192.168.30.100,192.168.30.250,2h
```

Check what you built:

```bash
ip -br addr                 # interfaces and their IPs
ip route                    # routing table: default route via the WAN
ip route get 1.1.1.1        # which interface and gateway the kernel picks for a destination
ip -d link show vlan10      # VLAN details (id, parent)
bridge link                 # which ports belong to which bridge
bridge fdb show br br-lan   # MAC addresses learned by the bridge
ip neigh                    # ARP / neighbor table
iptables -L -v -n           # firewall rules with packet counters
iptables -t nat -L -v -n    # NAT rules
```

Undo everything: `iptables -P INPUT ACCEPT; iptables -P FORWARD ACCEPT; iptables -F; iptables -t nat -F` (policies first, so the flush never leaves an empty `DROP` chain; same with `ip6tables`, without the NAT part). Then `ip link del br-lan` (and the other bridges and `vlanXX`), or simply reboot.

More routing with `ip route` / `ip rule`:

```bash
# Static route: reach another network through a router on the LAN (e.g. a VPN box)
ip route add 10.8.0.0/24 via 192.168.10.5
ip route del 10.8.0.0/24

# Policy routing (dual WAN): send IoT traffic out of a second uplink eth2 (also add a MASQUERADE on eth2)
ip route add default via 198.51.100.1 dev eth2 table 100
ip rule add from 192.168.20.0/24 table 100
```

### Old commands vs `ip`

`ifconfig`, `route`, `brctl`, `vconfig`, `arp` and `netstat` come from deprecated packages (`net-tools`, `bridge-utils`, `vlan`) that are often not installed anymore. You'll still meet them in old tutorials and on embedded boards (BusyBox):

| Task | Old way | Modern (`iproute2`) |
|---|---|---|
| Show interfaces | `ifconfig -a` | `ip addr` / `ip -br a` |
| Set IP and bring up | `ifconfig br-lan 192.168.10.1 netmask 255.255.255.0 up` | `ip addr add 192.168.10.1/24 dev br-lan` + `ip link set br-lan up` |
| Show routes | `route -n` / `netstat -rn` | `ip route` |
| Default route | `route add default gw 203.0.113.1` | `ip route add default via 203.0.113.1` |
| Create VLAN | `vconfig add eth1 10` | `ip link add link eth1 name eth1.10 type vlan id 10` |
| Create bridge | `brctl addbr br-lan` | `ip link add name br-lan type bridge` |
| Add port to bridge | `brctl addif br-lan eth1.10` | `ip link set eth1.10 master br-lan` |
| Show bridges | `brctl show` | `bridge link` |
| ARP table | `arp -n` | `ip neigh` |
| Listening ports | `netstat -tulpn` | `ss -tulpn` |

`ifconfig` and `route` can't do multiple IPs per interface cleanly, policy routing (`ip rule`) or routing tables, so stick to `ip` for anything new.

---

## 5. Interfaces and VLANs

Layout: `eth0` = WAN, `eth1` = trunk to the switch. Each VLAN goes into its own bridge (`br-lan`, `br-iot`, `br-guest`), so a local Wi-Fi interface can join the same network later.

### Method 1: systemd-networkd (recommended)

Handles VLANs, bridges, DHCP client and IPv6 prefix delegation in one tool.

#### WAN

`/etc/systemd/network/10-wan.network`

```ini
[Match]
Name=eth0

[Network]
DHCP=yes
IPv6AcceptRA=yes

[DHCPv4]
# We run our own resolver
UseDNS=no
# Main WAN; backup uplinks use 200 (section 11)
RouteMetric=100

[DHCPv6]
UseDNS=no
# Ask the ISP for a /56 (a request, not a guarantee: see "Prerequisites and checks" below)
PrefixDelegationHint=::/56

[IPv6AcceptRA]
UseDNS=no
# Request a prefix even if the ISP's RA doesn't set the "managed" flag
DHCPv6Client=always
```

> systemd files don't support comments after a value: `RouteMetric=100  # main WAN` fails to parse, and the line is **silently ignored**. Keep comments on their own lines.

> **PPPoE**: many fiber/DSL ISPs require PPPoE. Use `pppd` with the `rp-pppoe` plugin on top of `eth0` (often on a VLAN like `eth0.835` for some ISPs) and treat `ppp0` as the WAN in the firewall. Remember the MSS clamping rule in section 7.

#### LAN trunk and VLANs

`/etc/systemd/network/20-lan-trunk.network`

```ini
[Match]
Name=eth1

[Network]
VLAN=vlan10
VLAN=vlan20
VLAN=vlan30
LinkLocalAddressing=no
```

For each VLAN, create a VLAN netdev, a bridge netdev, and attach one to the other. Example for VLAN 10:

`/etc/systemd/network/30-vlan10.netdev`

```ini
[NetDev]
Name=vlan10
Kind=vlan

[VLAN]
Id=10
```

`/etc/systemd/network/30-vlan10.network`

```ini
[Match]
Name=vlan10

[Network]
Bridge=br-lan
```

`/etc/systemd/network/40-br-lan.netdev`

```ini
[NetDev]
Name=br-lan
Kind=bridge
```

`/etc/systemd/network/40-br-lan.network`

```ini
[Match]
Name=br-lan

[Network]
Address=192.168.10.1/24
ConfigureWithoutCarrier=yes
IPv6SendRA=yes
DHCPPrefixDelegation=yes

[DHCPPrefixDelegation]
UplinkInterface=eth0
SubnetId=1
Announce=yes
```

Repeat for the other VLANs:

| VLAN netdev | Bridge | Address | `SubnetId` |
|---|---|---|---|
| `vlan20` | `br-iot` | `192.168.20.1/24` | `2` |
| `vlan30` | `br-guest` | `192.168.30.1/24` | `3` |

`SubnetId` is **hexadecimal** and picks which /64 of the delegated prefix a LAN gets. It must fit in the bits between the delegated length and /64: `0`-`ff` for a /56, `0`-`f` for a /60, only `0` for a /64. Small IDs like `1`, `2`, `3` work with a /56, a /60 or a /62. (`10` would mean 0x10, which doesn't fit in a /60.)

Apply and check:

```bash
sudo networkctl reload
networkctl list
ip -br addr
```

> `DHCPPrefixDelegation=` requires systemd ≥ 253 (Debian 13, Ubuntu 24.04). On older versions, use `dhcpcd` or `wide-dhcpv6-client` for prefix delegation.

#### IPv6 in short

| Mechanism | Where | What it does |
|---|---|---|
| Link-local (`fe80::/10`) | Every interface | Always present, used for neighbor discovery and to talk to the next router |
| RA (Router Advertisement) | ISP → WAN, gateway → LANs | "I'm your router, here's the prefix" |
| SLAAC | LAN clients | Clients build their own global address from the RA prefix. Used here, so no DHCPv6 server on the LANs |
| DHCPv6 (address) | Optional | Stateful address assignment, the IPv6 equivalent of DHCPv4. Not needed here |
| DHCPv6-PD | Gateway ↔ ISP | The ISP **delegates a prefix** (e.g. a /56), and the gateway cuts one /64 per LAN (`SubnetId`) |

IPv6 is **routed, not NATed**: protection comes from the forward chain (section 7).

**Prerequisites and checks:**
- The ISP must actually delegate a prefix. `PrefixDelegationHint=::/56` is a request, not a guarantee: you may get a /60, a /64, or nothing at all.
- **With a /64**, only `SubnetId=0` fits, so only one LAN can have IPv6. Give `br-lan` `SubnetId=0` and leave the others IPv4-only.
- **With no prefix**, the LANs simply stay IPv4-only: clients get no global IPv6 address, so they don't try IPv6 for internet traffic. Nothing else breaks.
- `networkctl status eth0` shows the delegated prefix. `ip -6 addr show br-lan` must show a global address (`2xxx:...`), and a LAN client must reach `curl -6 ifconfig.co`. If a bridge has none, check `journalctl -u systemd-networkd` for prefix-delegation errors.
- **The prefix can change** (ISP reconnect, router reboot on some ISPs). networkd announces the new one, but clients may keep using the old address until its lifetime expires, so expect a period of broken IPv6 after a change. Never hardcode delegated IPv6 addresses in the firewall or DNS.

### Method 2: ifupdown (classic Debian)

The traditional `/etc/network/interfaces` way. Needs `apt install vlan bridge-utils`. IPv6 prefix delegation isn't built in, so add `dhcpcd` for it. VLANs are named `eth1.10` instead of `vlan10`: the `vlan` package creates them from that `parent.id` naming.

```
auto eth0
iface eth0 inet dhcp

auto eth1.10
iface eth1.10 inet manual

auto br-lan
iface br-lan inet static
    address 192.168.10.1/24
    bridge_ports eth1.10
    bridge_stp off

# Repeat eth1.20 -> br-iot and eth1.30 -> br-guest
```

Apply with `sudo systemctl restart networking`.

> Ubuntu's **netplan** is a third option. It writes systemd-networkd config for you, so Method 1 is the same thing without the YAML layer.

---

## 6. Kernel settings

`/etc/sysctl.d/90-gateway.conf`

```ini
# Routing
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1

# Anti-spoofing: drop packets arriving on an interface they couldn't be routed back through.
# Strict (1) suits a single WAN. With a backup WAN (section 11), use 2 (loose) on both lines:
# strict mode drops replies that arrive on the WAN that isn't currently preferred.
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

# A router should not accept or send ICMP redirects
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv6.conf.all.accept_redirects = 0

# Hardening
net.ipv4.tcp_syncookies = 1
net.ipv4.conf.all.log_martians = 1

# Busy homes (many devices, torrents, game consoles) can exhaust the default
net.netfilter.nf_conntrack_max = 131072
```

```bash
sudo sysctl --system
# nf_conntrack_max only exists once the nf_conntrack module is loaded. At boot, that can happen
# after systemd-sysctl has run, and the setting is then skipped. Load the module early:
echo nf_conntrack | sudo tee /etc/modules-load.d/conntrack.conf
```

> With `forwarding = 1`, the kernel ignores Router Advertisements on the WAN. That's fine here because systemd-networkd handles RAs itself (`IPv6AcceptRA=yes`). If you use another tool, set `net.ipv6.conf.eth0.accept_ra = 2`.

---

## 7. Firewall and NAT

All methods below drive the same kernel firewall (netfilter). Pick **one** and don't mix them.

| Method | Status | Notes |
|---|---|---|
| 1. nftables | Modern default (Debian 10+, Ubuntu 20.04+) | One file, IPv4 + IPv6 together, sets and maps |
| 2. iptables | Classic, still everywhere | Separate rules for IPv4 (`iptables`) and IPv6 (`ip6tables`) |
| 3. firewalld / ufw | Frontends | Generate nftables/iptables rules from zones/commands |

### How a packet crosses the gateway

Every method hooks into the same netfilter points. Knowing this path is what lets you debug "it pings the gateway but not the internet":

```
 packet in (any interface)
     │
 PREROUTING ─── nat: DNAT (port forwards), DNS redirect
     │
 routing decision ── for the gateway itself? ──► INPUT ──► local service (dnsmasq, sshd)
     │ no                                                        │ its replies
 FORWARD ─── filter: LAN → WAN allowed? inter-VLAN policy        ▼
     │                                                         OUTPUT
     ▼                                                           │
 POSTROUTING ◄───────────────────────────────────────────────────┘
     │   nat: masquerade (source NAT) out of the WAN
     ▼
 packet out
```

- **Connection tracking** sees the first packet of each connection. NAT rules only run for that first packet, and replies are un-NATed automatically.
- **Debug order**, LAN client → internet:
  1. `tcpdump -ni br-lan` (does the request arrive?)
  2. FORWARD counters (`nft list ruleset`, or `iptables -L FORWARD -v`)
  3. `tcpdump -ni eth0` (does it leave with the WAN IP? does a reply come back?)
  4. `conntrack -L` (is the reply matched to the connection?)

  No reply on `eth0` means the problem is upstream. A reply on `eth0` that never reaches `br-lan` means it's a gateway problem: conntrack, firewall or routing.

These are the rules every gateway should have:

1. **Default deny inbound** to the router itself and to the LAN.
2. **Allow established/related** traffic (replies to connections started from inside).
3. **Drop invalid** packets.
4. **Allow essential ICMP/ICMPv6**. IPv6 breaks without neighbor discovery and "packet too big".
5. **Router services** (DNS, DHCP) reachable from internal networks only; **SSH from the trusted LAN only**.
6. **Outbound** internet access for every internal network.
7. **Inter-VLAN policy**: LAN → IoT allowed, IoT/Guest → LAN denied.
8. **NAT/masquerade** IPv4 out of the WAN.
9. **Port forwards** only where needed, explicitly.
10. **MSS clamping** when the WAN MTU is below 1500 (PPPoE, tunnels).

### Method 1: nftables (recommended)

`/etc/nftables.conf`

```nft
#!/usr/sbin/nft -f
flush ruleset        # deletes EVERY table, not only these ones: see the warning below

define WAN   = "eth0"
define LAN   = "br-lan"
define IOT   = "br-iot"
define GUEST = "br-guest"

table inet filter {

    chain input {
        type filter hook input priority filter; policy drop;

        ct state established,related accept
        ct state invalid drop
        iif "lo" accept

        # ICMP: ping and error messages (rate-limited ping)
        icmp type echo-request limit rate 10/second accept
        icmp type { destination-unreachable, time-exceeded, parameter-problem } accept

        # ICMPv6: required for IPv6 to work at all (NDP, RA, PMTU)
        icmpv6 type { destination-unreachable, packet-too-big, time-exceeded, parameter-problem,
                      nd-router-solicit, nd-router-advert, nd-neighbor-solicit, nd-neighbor-advert,
                      mld-listener-query, mld-listener-report, mld2-listener-report } accept
        icmpv6 type echo-request limit rate 10/second accept

        # WAN: DHCPv4 and DHCPv6 client replies from the ISP
        iifname $WAN udp sport 67 udp dport 68 accept
        iifname $WAN ip6 saddr fe80::/10 udp sport 547 udp dport 546 accept

        # Internal networks: DHCP and DNS served by the gateway
        iifname { $LAN, $IOT, $GUEST } udp dport 67 accept
        iifname { $LAN, $IOT, $GUEST } meta l4proto { tcp, udp } th dport 53 accept

        # Administration: trusted LAN only
        iifname $LAN tcp dport 22 accept

        limit rate 5/minute log prefix "nft input drop: "
    }

    chain forward {
        type filter hook forward priority filter; policy drop;

        # Clamp TCP MSS to the path MTU (needed with PPPoE, harmless otherwise)
        tcp flags syn tcp option maxseg size set rt mtu

        ct state established,related accept
        ct state invalid drop

        # ICMPv6 errors must pass for IPv6 path MTU discovery
        icmpv6 type { destination-unreachable, packet-too-big, time-exceeded, parameter-problem } accept

        # Every internal network can reach the internet
        iifname { $LAN, $IOT, $GUEST } oifname $WAN accept

        # Trusted LAN can reach IoT devices (e.g. to cast or manage a camera); not the reverse
        iifname $LAN oifname $IOT accept

        # Inbound connections that match a port forward (DNAT) in the nat table
        iifname $WAN ct status dnat accept

        # Everything else is dropped: IoT -> LAN, Guest -> anything internal, unsolicited WAN -> LAN (IPv4 and IPv6)
        limit rate 5/minute log prefix "nft forward drop: "
    }

    chain output {
        type filter hook output priority filter; policy accept;
    }
}

table ip nat {

    chain prerouting {
        type nat hook prerouting priority dstnat; policy accept;

        # Port forward example: public 8443 -> NAS HTTPS
        # iifname $WAN tcp dport 8443 dnat to 192.168.10.10:443

        # Optional: force IoT/guest DNS through the gateway, even if devices hardcode 8.8.8.8
        # (IPv4 only: this table doesn't see DNS sent to IPv6 servers)
        iifname { $IOT, $GUEST } meta l4proto { tcp, udp } th dport 53 redirect to :53
    }

    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;

        oifname $WAN masquerade
    }
}
```

Load and check:

```bash
sudo nft -c -f /etc/nftables.conf   # syntax check first
sudo systemctl restart nftables
sudo nft list ruleset
```

> **`flush ruleset` owns the whole firewall.** It also deletes tables created by Docker, libvirt, fail2ban or VPN tools. That's fine on a dedicated gateway. On a shared machine, give your tables their own names (e.g. `table inet gw`), replace `flush ruleset` with `table inet gw` + `flush table inet gw`, and leave the other tables alone. With several tables, a packet must be accepted by every base chain on its hook: a `drop` in any of them wins.

`nft -c` only checks syntax, not whether you'll keep access. When changing the firewall **remotely**, schedule an automatic rollback first:

```bash
# 1. Save the current, working ruleset (with a flush on top, so it fully replaces)
sudo sh -c '{ echo "flush ruleset"; nft list ruleset; } > /root/nftables.good.conf'
# 2. Roll back automatically in 2 minutes unless cancelled
sudo systemd-run --unit=fw-rollback --on-active=120 /usr/sbin/nft -f /root/nftables.good.conf
# 3. Apply the new rules
sudo nft -c -f /etc/nftables.conf && sudo nft -f /etc/nftables.conf
# 4. Still connected? Cancel the rollback
sudo systemctl stop fw-rollback.timer
```

### Method 2: iptables (classic)

Same policy, written for iptables. Install `iptables-persistent`, which loads these files at boot. `br-+` matches every interface starting with `br-`.

`/etc/iptables/rules.v4`

```
*filter
:INPUT DROP [0:0]
:FORWARD DROP [0:0]
:OUTPUT ACCEPT [0:0]

-A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
-A INPUT -m conntrack --ctstate INVALID -j DROP
-A INPUT -i lo -j ACCEPT
-A INPUT -p icmp --icmp-type echo-request -m limit --limit 10/second -j ACCEPT
-A INPUT -i eth0 -p udp --sport 67 --dport 68 -j ACCEPT
-A INPUT -i br-+ -p udp --dport 67 -j ACCEPT
-A INPUT -i br-+ -p udp --dport 53 -j ACCEPT
-A INPUT -i br-+ -p tcp --dport 53 -j ACCEPT
-A INPUT -i br-lan -p tcp --dport 22 -j ACCEPT
-A INPUT -m limit --limit 5/min -j LOG --log-prefix "ipt input drop: "

-A FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
-A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
-A FORWARD -m conntrack --ctstate INVALID -j DROP
-A FORWARD -i br-+ -o eth0 -j ACCEPT
-A FORWARD -i br-lan -o br-iot -j ACCEPT
-A FORWARD -i eth0 -m conntrack --ctstate DNAT -j ACCEPT
-A FORWARD -m limit --limit 5/min -j LOG --log-prefix "ipt forward drop: "
COMMIT

*nat
:PREROUTING ACCEPT [0:0]
:INPUT ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
# Port forward example: public 8443 -> NAS HTTPS
# -A PREROUTING -i eth0 -p tcp --dport 8443 -j DNAT --to-destination 192.168.10.10:443
-A POSTROUTING -o eth0 -j MASQUERADE
COMMIT
```

`/etc/iptables/rules.v6`: same policy for IPv6, with no NAT. ICMP rules use `ipv6-icmp` (`--icmp-type` doesn't exist in ip6tables), and the WAN DHCP rule becomes DHCPv6:

```
*filter
:INPUT DROP [0:0]
:FORWARD DROP [0:0]
:OUTPUT ACCEPT [0:0]

-A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
-A INPUT -m conntrack --ctstate INVALID -j DROP
-A INPUT -i lo -j ACCEPT
-A INPUT -p ipv6-icmp -j ACCEPT
-A INPUT -i eth0 -s fe80::/10 -p udp --sport 547 --dport 546 -j ACCEPT
-A INPUT -i br-+ -p udp --dport 53 -j ACCEPT
-A INPUT -i br-+ -p tcp --dport 53 -j ACCEPT
-A INPUT -i br-lan -p tcp --dport 22 -j ACCEPT
-A INPUT -m limit --limit 5/min -j LOG --log-prefix "ip6t input drop: "

-A FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
-A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
-A FORWARD -m conntrack --ctstate INVALID -j DROP
-A FORWARD -p ipv6-icmp --icmpv6-type destination-unreachable -j ACCEPT
-A FORWARD -p ipv6-icmp --icmpv6-type packet-too-big -j ACCEPT
-A FORWARD -p ipv6-icmp --icmpv6-type time-exceeded -j ACCEPT
-A FORWARD -p ipv6-icmp --icmpv6-type parameter-problem -j ACCEPT
-A FORWARD -i br-+ -o eth0 -j ACCEPT
-A FORWARD -i br-lan -o br-iot -j ACCEPT
-A FORWARD -m limit --limit 5/min -j LOG --log-prefix "ip6t forward drop: "
COMMIT
```

```bash
sudo iptables-restore --test < /etc/iptables/rules.v4   # syntax check
sudo ip6tables-restore --test < /etc/iptables/rules.v6
sudo netfilter-persistent reload
sudo iptables -S && sudo iptables -t nat -S
```

> On current distros, the `iptables` command is usually **iptables-nft**: it accepts the old syntax but stores rules in nftables (`iptables -V` shows `nf_tables`). To convert old rules, use `iptables-restore-translate -f rules.v4`.

### Method 3: firewalld (zones)

Good if you prefer commands over rule files (default on Fedora/RHEL). Each interface goes into a zone, and **policies** control traffic between zones (firewalld ≥ 0.9).

```bash
sudo firewall-cmd --permanent --zone=external --change-interface=eth0     # external zone: masquerade on
sudo firewall-cmd --permanent --zone=external --remove-service=ssh        # external allows SSH by default
sudo firewall-cmd --permanent --zone=internal --change-interface=br-lan   # defaults: ssh, mdns, samba-client, dhcpv6-client
sudo firewall-cmd --permanent --zone=internal --add-service={dns,dhcp}    # not in internal's defaults
sudo firewall-cmd --permanent --new-zone=iot
sudo firewall-cmd --permanent --zone=iot --change-interface=br-iot
sudo firewall-cmd --permanent --zone=iot --add-service={dns,dhcp}

# Allow internal -> internet (repeat --add-ingress-zone for guest, ...)
sudo firewall-cmd --permanent --new-policy=lan-out
sudo firewall-cmd --permanent --policy=lan-out --add-ingress-zone=internal
sudo firewall-cmd --permanent --policy=lan-out --add-ingress-zone=iot
sudo firewall-cmd --permanent --policy=lan-out --add-egress-zone=external
sudo firewall-cmd --permanent --policy=lan-out --set-target=ACCEPT

# Allow LAN -> IoT, not the reverse
sudo firewall-cmd --permanent --new-policy=lan-to-iot
sudo firewall-cmd --permanent --policy=lan-to-iot --add-ingress-zone=internal
sudo firewall-cmd --permanent --policy=lan-to-iot --add-egress-zone=iot
sudo firewall-cmd --permanent --policy=lan-to-iot --set-target=ACCEPT

# Port forward
# sudo firewall-cmd --permanent --zone=external --add-forward-port=port=8443:proto=tcp:toport=443:toaddr=192.168.10.10

sudo firewall-cmd --reload
```

> **ufw** also works, but routing and NAT have to be added by hand in `/etc/ufw/before.rules`, so it fits a server better than a router.

### Notes on the rules

- **IPv6 has no NAT here, and that's fine.** LAN devices get public IPv6 addresses, but the forward chain only lets in replies (`established,related`). Unsolicited inbound IPv6 is dropped by the policy. This is what protects you, not NAT.
- **Guest client isolation** (guests not seeing each other) happens in hostapd (`ap_isolate`, see section 10), because the Wi-Fi driver delivers traffic between two clients on the same SSID directly, before the firewall sees it. It only covers Wi-Fi guests: wired guest ports on VLAN 30 need port isolation on the switch.
- **Traffic inside one bridge** (wired PC ↔ Wi-Fi phone on `br-lan`) normally never reaches the firewall. If the `br_netfilter` module is loaded (`lsmod | grep br_netfilter`, Docker loads it), it does go through the forward hook and the drop policy can block it. Test this. If it's blocked and nothing else needs `br_netfilter`, set `net.bridge.bridge-nf-call-iptables = 0` and `net.bridge.bridge-nf-call-ip6tables = 0`.
- **Don't expose SSH or the admin UI on the WAN.** Use a VPN (WireGuard) for remote access instead of port forwards.
- **UPnP/NAT-PMP**: leave it off. If a game console needs it, run `miniupnpd` restricted to that console's IP only.
- **Hairpin NAT** (reaching your public IP from inside): easier to solve with local DNS (`nas.home.arpa` → private IP) than with extra NAT rules.

---

## 8. DHCP and DNS (dnsmasq)

`/etc/dnsmasq.d/gateway.conf`

```ini
# Only listen on internal networks, never on the WAN
interface=br-lan
interface=br-iot
interface=br-guest
bind-dynamic

# DNS behaviour
domain-needed        # don't forward plain names without a dot upstream
bogus-priv           # don't forward reverse lookups for private ranges
no-resolv            # ignore /etc/resolv.conf, use the servers below
server=1.1.1.1
server=9.9.9.9
cache-size=10000

# Local domain: hosts get name.home.arpa (RFC 8375)
domain=home.arpa
local=/home.arpa/
expand-hosts

# DHCP ranges, one per network (router and DNS default to the interface IP)
dhcp-authoritative
dhcp-range=set:lan,192.168.10.100,192.168.10.250,255.255.255.0,12h
dhcp-range=set:iot,192.168.20.100,192.168.20.250,255.255.255.0,12h
dhcp-range=set:guest,192.168.30.100,192.168.30.250,255.255.255.0,2h

# Static leases (keep them below .100)
dhcp-host=aa:bb:cc:dd:ee:ff,nas,192.168.10.10
dhcp-host=11:22:33:44:55:66,printer,192.168.10.11

# Log DHCP for debugging (comment out once stable)
log-dhcp
```

```bash
sudo systemctl restart dnsmasq
journalctl -u dnsmasq -f
```

> On Ubuntu, `systemd-resolved` listens on port 53 (127.0.0.53). `bind-dynamic` avoids the conflict, but you can also set `DNSStubListener=no` in `/etc/systemd/resolved.conf`.

IPv6 addressing on the LAN uses SLAAC from the RAs sent by systemd-networkd, so no DHCPv6 server is needed. Clients still use the gateway's IPv4 address for DNS.

> **Alternatives:** dnsmasq covers a home network well. For bigger setups, split the roles: **Kea** for DHCP (successor of the now end-of-life ISC `dhcpd`) and **Unbound** as a validating DNS resolver. **Pi-hole** / **AdGuard Home** add ad-blocking on top.

---

## 9. Switch configuration

On the managed switch:

| Port | Mode | VLANs |
|---|---|---|
| Uplink to gateway `eth1` | Trunk | 10, 20, 30 tagged |
| PC / NAS | Access | 10 untagged |
| Smart TV, wired IoT | Access | 20 untagged |

Move the switch's own management IP into VLAN 10, and change its default admin password.

---

## 10. Wi-Fi access point (hostapd)

The gateway's own Wi-Fi card broadcasts one SSID per network, and `hostapd` attaches each SSID to the matching bridge (`br-lan`, `br-guest`...). Wi-Fi clients then land on the same networks as wired ones.

AP support depends on the exact chipset, driver, firmware and regulatory rules, so check your card instead of trusting the brand:

```bash
iw list | grep -A 10 "Supported interface modes"     # must contain "AP"
iw list | grep -A 8 "valid interface combinations"   # how many APs (SSIDs) / AP + station at once
iw list | grep -E "MHz.*(no IR|disabled)"            # "no IR" = can't start an AP on that channel
```

Many client cards can only start an AP on 2.4 GHz because their 5 GHz channels are marked `no IR`.

The config below is an example for a card that supports 5 GHz AP mode, 80 MHz VHT and two SSIDs at once (`#{ AP } <= 2` or more in the interface combinations). If yours doesn't, hostapd refuses to start: adjust the band, channel width and `ht_capab` to what `iw list` reports. It was written for the hostapd 2.10+ shipped by Debian/Ubuntu.

| SSID | Bridge | Security |
|---|---|---|
| `Home` | `br-lan` | WPA2/WPA3 mixed mode |
| `Home-Guest` | `br-guest` | WPA2/WPA3, clients isolated |
| `Home-IoT` | `br-iot` | WPA2 only (many IoT devices don't support WPA3), 2.4 GHz |

`/etc/hostapd/hostapd.conf`

```ini
interface=wlan0
bridge=br-lan
driver=nl80211
# Your country: sets legal channels and power
country_code=FR
ieee80211d=1

# 5 GHz, 80 MHz on channel 36 (36-48). ht_capab must match the card's "HT Capabilities" in iw list
hw_mode=a
channel=36
ieee80211n=1
ieee80211ac=1
wmm_enabled=1
ht_capab=[HT40+][SHORT-GI-20][SHORT-GI-40]
vht_oper_chwidth=1
vht_oper_centr_freq_seg0_idx=42

# Main network: WPA2/WPA3 transition mode
ssid=Home
wpa=2
wpa_key_mgmt=WPA-PSK SAE
rsn_pairwise=CCMP
wpa_passphrase=change-me-long-passphrase
# Management frame protection: optional for WPA2 clients, forced for WPA3 (SAE) clients
ieee80211w=1
sae_require_mfp=1

# Guest network on a second BSS, bridged to the guest VLAN
bss=wlan0_1
bridge=br-guest
ssid=Home-Guest
wpa=2
wpa_key_mgmt=WPA-PSK SAE
rsn_pairwise=CCMP
wpa_passphrase=another-guest-passphrase
ieee80211w=1
sae_require_mfp=1
# Guests can't talk to each other
ap_isolate=1
```

Security modes, as set above:

| Mode | `wpa_key_mgmt` | `ieee80211w` (MFP) | Use for |
|---|---|---|---|
| WPA2 only | `WPA-PSK` | `0` or `1` | Old/IoT devices that fail on anything newer |
| WPA2/WPA3 transition | `WPA-PSK SAE` | `1` + `sae_require_mfp=1`: WPA2 clients may skip MFP, WPA3 clients must use it | Mixed household devices (used above) |
| WPA3 only | `SAE` | `2` (required) | Networks where every device supports WPA3 |

MFP (management frame protection, 802.11w) stops forged disconnect frames. WPA3 requires it. In transition mode, hostapd only enforces it for SAE clients when `sae_require_mfp=1` is set.

```bash
# Debian ships hostapd.service masked; it reads /etc/hostapd/hostapd.conf
sudo systemctl unmask hostapd
sudo systemctl enable --now hostapd
journalctl -u hostapd -f
```

**IoT on 2.4 GHz:** one radio works on one band. Most IoT devices are 2.4 GHz only, so give `Home-IoT` a second radio (e.g. `wlan2`) with its own file. `hostapd@wlan2.service` reads `/etc/hostapd/wlan2.conf`. Write it as a complete single-SSID config: copy the first block above, drop the `bss=` part and the 5 GHz lines (`ieee80211ac`, `vht_*`), then set `interface=wlan2`, `bridge=br-iot`, `hw_mode=g`, `channel=6`, `ssid=Home-IoT`, `wpa_key_mgmt=WPA-PSK`, `ieee80211w=0`, and remove `sae_require_mfp`. Start it with `sudo systemctl enable --now hostapd@wlan2`.

### Wi-Fi planning tips

- **2.4 GHz**: only channels 1, 6 or 11, 20 MHz wide. Wider channels on 2.4 GHz only cause interference.
- **5 GHz**: 80 MHz wide. Channels 36-48 avoid DFS; DFS channels (52-144) are less crowded but the AP must go quiet briefly on radar detection.
- **Placement**: put the gateway (or its antennas) central and high. Walls and floors cost more signal than distance.
- Use a Wi-Fi analyzer app to check neighboring networks before picking channels.

---

## 11. Other WAN uplinks and backup

Ethernet to an ISP modem is the usual WAN, but any interface that reaches the internet can play that role, either **instead of** `eth0` or **next to it as a backup**.

| Uplink | Interface | Good for | Watch out |
|---|---|---|---|
| Ethernet | `eth0` | Main connection | |
| Wi-Fi station | `wlan1` | Campsite, hotel, phone hotspot | Needs a second Wi-Fi radio |
| Cellular (Quectel EC25) | `wwan0` | Backup, rural sites, vehicles | CGNAT (no port forwards), data caps |
| Satellite (Starlink) | `eth0` | Rural without fiber/DSL | CGNAT on IPv4, higher latency |

The firewall must treat the new interface as a WAN:

- **nftables**: turn the define into a set, `define WAN = { "eth0", "wwan0" }`. Every `$WAN` rule, masquerade included, then covers both.
- **iptables**: duplicate each `-i eth0` / `-o eth0` rule for the new interface.
- **firewalld**: `firewall-cmd --permanent --zone=external --add-interface=wwan0`.

### Method 1: Wi-Fi station (client)

Use a **second** Wi-Fi adapter (`wlan1`). Some radios can run AP and station at once (see `valid interface combinations`), but only on one channel. Your AP would then have to follow whatever channel the upstream network uses, which hostapd doesn't do on its own. Needs `apt install wpasupplicant`.

`/etc/wpa_supplicant/wpa_supplicant-wlan1.conf`

```ini
ctrl_interface=/run/wpa_supplicant
country=FR

network={
    ssid="Upstream-WiFi"
    psk="upstream-passphrase"
}
```

`/etc/systemd/network/11-wan-wifi.network`

```ini
[Match]
Name=wlan1

[Network]
DHCP=yes

[DHCPv4]
# Backup: used only when eth0 has no default route
RouteMetric=200
```

```bash
sudo systemctl enable --now wpa_supplicant@wlan1
sudo networkctl reload
# By hand: wpa_supplicant -B -i wlan1 -c /etc/wpa_supplicant/wpa_supplicant-wlan1.conf && dhcpcd wlan1
```

### Method 2: Cellular modem (Quectel EC25)

The EC25 (mini-PCIe card or USB dongle) shows up as `/dev/cdc-wdm0` (QMI control), `wwan0` (data interface) and `/dev/ttyUSB0-3` (AT commands on `ttyUSB2`).

```bash
lsusb | grep 2c7c                                  # Quectel vendor ID (EC25 = 2c7c:0125)
sudo apt install modemmanager libqmi-utils
mmcli -L                                           # list modems
mmcli -m 0                                         # SIM state, operator, signal
sudo mmcli -m 0 --simple-connect="apn=internet,ip-type=ipv4"   # your carrier's APN
mmcli -m 0 | grep -i bearer                        # bearer path: the index is not always 0
mmcli -b <N>                                       # interface, IPv4 method, address, prefix, gateway, DNS, MTU

# If mmcli -b reports "method: static" (usual with QMI):
sudo ip addr flush dev wwan0
sudo ip addr add <address>/<prefix> dev wwan0
sudo ip link set wwan0 mtu <mtu> up
sudo ip route add default via <gateway> dev wwan0 metric 200   # backup route
# If it reports "method: dhcp", run a DHCP client on wwan0 instead
```

ModemManager only opens the session: it doesn't configure `wwan0`, and the values change at every reconnect. Your DNS stays dnsmasq with its fixed upstream servers (section 8), so the carrier's DNS isn't needed.

The modem has two separate sides: a **control plane** (QMI/MBIM/AT: SIM, registration, opening the data session) and a **data plane** (`wwan0`, a normal Linux interface that needs IP, route, DNS and firewall). Opening the session doesn't configure Linux. Pick **one** owner for the whole chain, and never let two tools manage `wwan0`:

| Approach | Who does what | Good for |
|---|---|---|
| **Manual** (above) | ModemManager opens the session, you run the `ip` commands again after every reconnect | Learning, lab |
| **Managed** | One tool does session + IP + route + reconnect: `quectel-CM` (from Quectel support), or NetworkManager with ModemManager | Unattended gateway |

If you use NetworkManager for the modem, restrict it to that device and keep it away from DNS, so it doesn't fight systemd-networkd and dnsmasq. In `/etc/NetworkManager/conf.d/modem-only.conf`: `[keyfile]` `unmanaged-devices=except:type:gsm`, and `[main]` `dns=none`. NetworkManager gives mobile routes a high metric by default, so the connection stays a backup.

An unattended setup must survive these events: loss of registration, dropped data session, modem reset (the USB device disappears and comes back), new IP after reconnect. ModemManager alone doesn't reconnect after a drop, which is why the manual way is lab-only.

> These commands assume a USB modem in **QMI** mode (`qmi_wwan` driver, `/dev/cdc-wdm0`). In **MBIM** mode (`cdc_mbim`), or on PCIe/5G modems (`mhi` driver, `wwan` subsystem, RMNET), the device names and tools differ. Check with `mmcli -m 0` and `ls /sys/class/net`.

Quick checks with AT commands (`picocom /dev/ttyUSB2`, stop ModemManager first since it uses the same port):

| Command | Meaning |
|---|---|
| `AT+CPIN?` | `READY` = SIM unlocked |
| `AT+CSQ` | Signal 0-31 (above ~15 is fine, 99 = unknown) |
| `AT+COPS?` | Registered operator |
| `AT+CGDCONT=1,"IP","internet"` | Set the APN |

### Method 3: Satellite (Starlink)

To the gateway, Starlink is just an Ethernet WAN. Enable **bypass mode** on the Starlink router (in the Starlink app), which turns off its Wi-Fi and routing, and connect its Ethernet port (Ethernet adapter on Gen 2 routers) to `eth0`. On most plans you get:

- **IPv4 by DHCP in `100.64.0.0/10`**: that's CGNAT, so inbound port forwards won't work over IPv4. For remote access, use IPv6 or a tunnel through a VPS (WireGuard, Tailscale).
- **An IPv6 /56 by prefix delegation**: the section 5 config works as is.

### Failover

With `RouteMetric=100` on `eth0` and `200` on the backup, the kernel uses the lowest-metric default route. If `eth0` loses its link, its route disappears and traffic moves to the backup by itself (open connections restart).

That doesn't cover "cable up, but the ISP is down". The watchdog below shows the principle: it checks the internet **through `eth0`** and pushes a preferred route via the backup while that check fails. **It's a demonstration, not a production mechanism:**

```sh
#!/bin/sh
# /usr/local/bin/wan-watchdog: prefer wwan0 while eth0 can't reach the internet
while sleep 10; do
    if ping -I eth0 -c 3 -W 2 9.9.9.9 >/dev/null 2>&1; then
        ip route del default dev wwan0 metric 50 2>/dev/null     # eth0 OK: back to normal
    else
        ip route replace default dev wwan0 metric 50             # eth0 down: backup wins
    fi
done
```

`default dev wwan0` without a gateway only works when `wwan0` is in raw-IP mode (`ip -d link show wwan0` shows `POINTOPOINT,NOARP`). Otherwise, and for a `wlan1` backup, write `default via <its gateway> dev <iface> metric 50`.

**Set `rp_filter` to 2 (loose) first** (section 6). In strict mode, while the backup route is preferred, the kernel drops the replies to the watchdog's own `ping -I eth0`. The watchdog then never sees `eth0` recover and stays on the backup.

Its limits:
- One probe target, and ICMP only: a filtered ping looks like an outage.
- No hysteresis: a flaky line makes it flap between WANs.
- It assumes the backup is already connected (a modem session may need to be re-opened first).
- Open connections break at every switch (NAT state is tied to the old WAN IP).

A real setup uses a dedicated manager with several probe targets, "N failures to switch, M successes to switch back", and control over the backup link. On OpenWrt, `mwan3` does this. Either way, test it by unplugging the WAN, cutting upstream (modem up, ISP down), resetting the modem, and checking recovery.

---

## 12. Hardening checklist

- [ ] SSH: key-only auth (`PasswordAuthentication no`), `PermitRootLogin no`, listen on the LAN IP only.
- [ ] No service listening on the WAN: `sudo ss -lntup` and check every line.
- [ ] Automatic security updates (`unattended-upgrades`).
- [ ] Time sync (`systemd-timesyncd` or `chrony`). TLS and logs depend on it.
- [ ] Default passwords changed on the switch and the ISP modem.
- [ ] Back up `/etc/systemd/network`, the firewall rules (`/etc/nftables.conf` or `/etc/iptables/`), `/etc/dnsmasq.d`, `/etc/hostapd` (ideally in a private git repo).
- [ ] Remote access through WireGuard, not port forwards.
- [ ] Firmware updates on the switch (and the LTE modem, if any).

---

## 13. Verification

From the gateway:

```bash
ip -br addr                       # all bridges up with the right IPs
ip -6 route                       # delegated prefix routes on each bridge
sudo nft list ruleset             # rules loaded (iptables: sudo iptables -S)
sudo conntrack -L | head          # NAT and connection tracking in action
journalctl -k | grep -E "nft |ip6?t "   # dropped packets (from the log rules)
```

From a client on each network:

```bash
ip addr / ipconfig                # got 192.168.X.1xx + a public IPv6 (2xxx:)
ping 1.1.1.1                      # routing + NAT
dig example.com @192.168.10.1     # DNS through the gateway
curl -4 ifconfig.co               # shows the WAN IPv4
curl -6 ifconfig.co               # shows the client's own IPv6
```

Segmentation tests:

| From | To | Expected |
|---|---|---|
| LAN | IoT device | ✅ works |
| IoT | LAN device | ❌ blocked |
| Guest | LAN / IoT device | ❌ blocked |
| Guest | another guest | ❌ blocked (`ap_isolate`) |
| Internet | any LAN IP (IPv4 or IPv6) | ❌ blocked |

For the last one, scan your public IPs from outside (e.g. `nmap -Pn <wan-ip>` from a VPS, or an online IPv6 port scanner). Nothing should answer except your explicit port forwards.

Recovery tests. A gateway that only works after manual commands isn't finished:

| Test | Expected |
|---|---|
| Reboot, touch nothing | Everything above works again |
| Boot with the WAN unplugged, plug it in later | LANs get DHCP and local DNS meanwhile; internet and IPv6 come back without intervention |
| Restart one service (`dnsmasq`, `hostapd`, `nftables`) | It comes back without breaking the others |

The config already helps here: `ConfigureWithoutCarrier=yes` keeps LAN addresses even with no cable or WAN, and dnsmasq's `bind-dynamic` picks up interfaces that appear after it starts.

---

## 14. Troubleshooting

| Symptom | Likely cause |
|---|---|
| Clients get no IP | dnsmasq not listening on the bridge, firewall missing UDP 67, VLAN not tagged on the switch port |
| IP but no internet | `ip_forward` off, missing `masquerade`, WAN has no default route |
| Internet by IP but not by name | DNS port 53 not allowed in `input`, dnsmasq has no upstream (`no-resolv` without `server=`) |
| Some websites hang (IPv4) | MTU/MSS issue with PPPoE: check the MSS clamping rule |
| IPv6 doesn't work | ICMPv6 blocked, ISP doesn't delegate a prefix (check `networkctl status eth0`), wrong `PrefixDelegationHint` |
| Wi-Fi clients on wrong network | Wrong `bridge=` for that SSID in the hostapd config |
| Backup WAN never takes over | Same route metric on both WANs (check `ip route`: a comment after `RouteMetric=` silently disables it), or the backup interface is missing from the firewall's WAN rules |
| Failover never switches back | `rp_filter = 1` drops the watchdog's probe replies on `eth0`: use 2 (section 6) |

Handy capture commands:

```bash
sudo tcpdump -ni br-lan port 67 or port 68     # DHCP
sudo tcpdump -ni eth0 icmp6                    # IPv6 neighbor discovery / RAs on WAN
sudo tcpdump -ni eth0 -e vlan                  # verify VLAN tags
```

---

## References

- nftables wiki: <https://wiki.nftables.org>
- iptables: `man iptables`, `man iptables-extensions`
- firewalld policies: <https://firewalld.org/documentation/man-pages/firewalld.policy.html>
- systemd-networkd: `man systemd.network`, `man systemd.netdev`
- dnsmasq manual: <https://thekelleys.org.uk/dnsmasq/docs/dnsmasq-man.html>
- hostapd example config: <https://w1.fi/cgit/hostap/plain/hostapd/hostapd.conf>
- RFC 4890: recommendations for filtering ICMPv6 in firewalls
- RFC 8375: `home.arpa` for home networks
