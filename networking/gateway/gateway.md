# Building a Home Gateway on Linux

How to turn a small Linux box (mini-PC, Raspberry Pi with a USB NIC, or an old x86 machine) into a full home gateway: routing, NAT, firewall, DHCP, DNS, IPv6, network segmentation, Wi-Fi access point, and backup uplinks (Wi-Fi, LTE, satellite).

The goal is to reproduce everything a consumer ISP box does, but in a way you can understand, audit and extend.

## Table of contents

- [1. What a home gateway actually does](#1-what-a-home-gateway-actually-does)
   - [Ways to build one](#ways-to-build-one)
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

All of them do the same things under the hood. Each step below lists more than one method when there's a real choice.

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

> If the ISP modem can't be put in bridge mode, you'll have double NAT. It still works, but forward the modem's DMZ to the gateway's WAN IP so that port forwards keep working.

---

## 3. Hardware checklist

- **Two NICs minimum**: one WAN, one LAN. Intel NICs (i210/i225/i226) are the safe choice.
- **Managed switch** that supports 802.1Q VLANs, if you want segmentation (strongly recommended).
- **Wi-Fi card** with good AP-mode support (see section 10), plus a second one for 2.4 GHz IoT if needed.
- CPU: any modern x86 or ARM64 board routes 1 Gbit/s with nftables without trouble.

OS: any Debian/Ubuntu server works. Install the packages:

```bash
sudo apt install nftables dnsmasq hostapd iproute2 conntrack tcpdump   # swap nftables for iptables-persistent if using Method 2 in section 7
sudo systemctl enable --now systemd-networkd nftables
# Disable anything else managing interfaces (NetworkManager, ifupdown, netplan renderers)
```

---

## 4. Quick start: everything by hand

Before writing any config file, you can build the whole gateway live with plain commands (as root). It's the best way to understand each piece, and handy for testing. **Nothing survives a reboot**: sections 5 to 8 make it permanent.

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
    ip link add link eth1 name eth1.$1 type vlan id $1
    ip link add name br-$2 type bridge
    ip link set eth1.$1 master br-$2
    ip addr add 192.168.$1.1/24 dev br-$2
    ip link set eth1.$1 up
    ip link set br-$2 up
done

# --- Routing ---
sysctl -w net.ipv4.ip_forward=1

# --- Firewall + NAT ---
iptables -A INPUT -i eth0 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -i eth0 -j DROP                       # router unreachable from the internet
iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A FORWARD -i br-+ -o eth0 -j ACCEPT           # all networks -> internet
iptables -A FORWARD -i br-lan -o br-iot -j ACCEPT       # LAN -> IoT
iptables -P FORWARD DROP                                # anything else between networks: blocked
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

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
ip -d link show eth1.10     # VLAN details (id, parent)
bridge link                 # which ports belong to which bridge
bridge fdb show br br-lan   # MAC addresses learned by the bridge
ip neigh                    # ARP / neighbor table
iptables -L -v -n           # firewall rules with packet counters
iptables -t nat -L -v -n    # NAT rules
```

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
UseDNS=no            # we run our own resolver
RouteMetric=100      # main WAN; backup uplinks use 200 (section 11)

[DHCPv6]
UseDNS=no
PrefixDelegationHint=::/56   # ask the ISP for a /56 (ISP-dependent, /60 or /64 also common)

[IPv6AcceptRA]
UseDNS=no
```

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
SubnetId=10
Announce=yes
```

Repeat for the other VLANs:

| VLAN netdev | Bridge | Address | `SubnetId` |
|---|---|---|---|
| `vlan20` | `br-iot` | `192.168.20.1/24` | `20` |
| `vlan30` | `br-guest` | `192.168.30.1/24` | `30` |

Apply and check:

```bash
sudo networkctl reload
networkctl list
ip -br addr
```

> `DHCPPrefixDelegation=` requires systemd ≥ 253 (Debian 13, Ubuntu 24.04). On older versions, use `dhcpcd` or `wide-dhcpv6-client` for prefix delegation.

### Method 2: ifupdown (classic Debian)

The traditional `/etc/network/interfaces` way. Needs `apt install vlan bridge-utils`. IPv6 prefix delegation isn't built in, so add `dhcpcd` for it.

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

# Anti-spoofing: drop packets arriving on an interface they couldn't be routed back through
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
flush ruleset

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

`/etc/iptables/rules.v6`: the same `*filter` table with three changes. It has no `*nat` table. ICMPv6 must be accepted (`-A INPUT -p ipv6-icmp -j ACCEPT`, and the same in `FORWARD`). The DHCP client rule becomes `-A INPUT -i eth0 -s fe80::/10 -p udp --sport 547 --dport 546 -j ACCEPT`.

```bash
sudo iptables-restore --test < /etc/iptables/rules.v4   # syntax check
sudo netfilter-persistent reload
sudo iptables -S && sudo iptables -t nat -S
```

> On current distros, the `iptables` command is usually **iptables-nft**: it accepts the old syntax but stores rules in nftables (`iptables -V` shows `nf_tables`). To convert old rules, use `iptables-restore-translate -f rules.v4`.

### Method 3: firewalld (zones)

Good if you prefer commands over rule files (default on Fedora/RHEL). Each interface goes into a zone, and **policies** control traffic between zones (firewalld ≥ 0.9).

```bash
sudo firewall-cmd --permanent --zone=external --change-interface=eth0     # external zone = masquerade on
sudo firewall-cmd --permanent --zone=internal --change-interface=br-lan   # DNS/DHCP/SSH allowed
sudo firewall-cmd --permanent --new-zone=iot
sudo firewall-cmd --permanent --zone=iot --change-interface=br-iot
sudo firewall-cmd --permanent --zone=iot --add-service={dns,dhcp}

# Allow internal -> internet (add another ingress zone for iot, guest, ...)
sudo firewall-cmd --permanent --new-policy=lan-out
sudo firewall-cmd --permanent --policy=lan-out --add-ingress-zone=internal --add-ingress-zone=iot
sudo firewall-cmd --permanent --policy=lan-out --add-egress-zone=external
sudo firewall-cmd --permanent --policy=lan-out --set-target=ACCEPT

# Port forward
# sudo firewall-cmd --permanent --zone=external --add-forward-port=port=8443:proto=tcp:toport=443:toaddr=192.168.10.10

sudo firewall-cmd --reload
```

> **ufw** also works, but routing and NAT have to be added by hand in `/etc/ufw/before.rules`, so it fits a server better than a router.

### Notes on the rules

- **IPv6 has no NAT here, and that's fine.** LAN devices get public IPv6 addresses, but the forward chain only lets in replies (`established,related`). Unsolicited inbound IPv6 is dropped by the policy. This is what protects you, not NAT.
- **Guest client isolation** (guests not seeing each other) happens in hostapd (`ap_isolate`, see section 10), because the Wi-Fi driver delivers traffic between two clients on the same SSID directly, before the firewall sees it.
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

Check the card first: `iw list` → "Supported interface modes" must contain `AP`. Mediatek MT7921/MT7922 and Atheros chips work well, while most Intel cards are limited to 2.4 GHz in AP mode.

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
country_code=FR            # your country: sets legal channels and power
ieee80211d=1

# 5 GHz, 80 MHz on channel 36 (36-48)
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
ieee80211w=1               # management frame protection: optional for WPA2, required by SAE

# Guest network on a second BSS, bridged to the guest VLAN
bss=wlan0_1
bridge=br-guest
ssid=Home-Guest
wpa=2
wpa_key_mgmt=WPA-PSK SAE
rsn_pairwise=CCMP
wpa_passphrase=another-guest-passphrase
ieee80211w=1
ap_isolate=1               # guests can't talk to each other
```

```bash
sudo systemctl unmask hostapd
sudo systemctl enable --now hostapd
journalctl -u hostapd -f
```

**IoT on 2.4 GHz:** one radio works on one band. Most IoT devices are 2.4 GHz only, so give `Home-IoT` a second radio (e.g. `wlan2`) with its own file, `/etc/hostapd/wlan2.conf`: `interface=wlan2`, `bridge=br-iot`, `hw_mode=g`, `channel=6`, `wpa_key_mgmt=WPA-PSK`. Start it with `sudo systemctl enable --now hostapd@wlan2`.

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

Use a **second** Wi-Fi adapter (`wlan1`). A single radio can't follow the upstream network's channel and serve your own AP at the same time.

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
RouteMetric=200      # backup: used only when eth0 has no route
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
mmcli -m 0                                         # SIM state, operator, signal, bearer path
sudo mmcli -m 0 --simple-connect="apn=internet"    # your carrier's APN
mmcli -b 0                                         # bearer: IP address, prefix, DNS

sudo ip link set wwan0 up
sudo ip addr add <ip>/<prefix> dev wwan0           # values from mmcli -b 0
sudo ip route add default dev wwan0 metric 200     # backup route
```

ModemManager alone doesn't reconnect after a drop. For an unattended link, Quectel's `quectel-CM` tool (from Quectel support) connects, configures `wwan0` and reconnects automatically.

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

That doesn't cover "cable up, but the ISP is down". For that, a small watchdog checks the internet **through `eth0`** and pushes a preferred route via the backup while it fails:

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

`wwan0` is point-to-point, so it needs no gateway address. For a `wlan1` backup, write `default via <its gateway> dev wlan1 metric 50`. Run the script as a systemd service (`Restart=always`). On OpenWrt, `mwan3` does the same with a UI.

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
journalctl -k | grep -E "nft |ipt "   # dropped packets (from the log rules)
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
| Backup WAN never takes over | Same route metric on both WANs, or the backup interface is missing from the firewall's WAN rules |

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
