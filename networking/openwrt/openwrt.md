# Networking on OpenWrt

How Linux networking works on an OpenWrt router, and how to configure, trace and debug it: netifd, UCI, firewall4/nftables, dnsmasq/odhcpd and hostapd, then rebuilding the firmware when the image lacks something.

Companion of [Building a Home Gateway on Linux](../gateway/gateway.md): same functions, but with OpenWrt's components instead of hand-written configs. Concepts explained there (netfilter hooks, nftables policy, prefix delegation, Wi-Fi security modes) are linked, not repeated.

**Assumptions:** OpenWrt **25.12** (25.12.6 at the time of writing: kernel 6.12, firewall4/nftables, `apk`, `config device` syntax) on an **OpenWrt One** (section 1). Debian 13 as build host and LAN laptop.

**Verification status:** syntax, defaults, interface and package names were checked against the `openwrt-25.12` sources (base-files, board files, firewall4, wifi-scripts, odhcpd, dnsmasq), the 25.12.6 release files and the official wiki. Nothing was built or run on an OpenWrt One for this note. Output shown is abridged, and **(verify on device)** marks what depends on your unit.

**How to read it:** to learn, sections 1-2, then 4-7 to build the gateway, then 9-10 to rebuild the firmware. To debug: the packet flow in section 2, then section 8.

## Table of contents

- [1. The project: an OpenWrt One as a gateway](#1-the-project-an-openwrt-one-as-a-gateway)
- [2. Architecture](#2-architecture)
- [3. Linux networking, the OpenWrt way](#3-linux-networking-the-openwrt-way)
- [4. Configuring the network with UCI](#4-configuring-the-network-with-uci)
- [5. Firewall, forwarding and NAT](#5-firewall-forwarding-and-nat)
- [6. Wi-Fi, bridges and VLANs: a guest network](#6-wi-fi-bridges-and-vlans-a-guest-network)
- [7. IPv6](#7-ipv6)
- [8. Troubleshooting](#8-troubleshooting)
- [9. Build, install and recover](#9-build-install-and-recover)
- [10. Extending the image](#10-extending-the-image)
- [11. Quick reference](#11-quick-reference)
- [References](#references)

---

## 1. The project: an OpenWrt One as a gateway

The OpenWrt One is the project's own reference board: officially supported, with a recovery system in a separate NOR flash and a built-in USB serial console.

| Item | Value | Checked in |
|---|---|---|
| Target / profile | `mediatek/filogic`, `openwrt_one` | `target/linux/mediatek/image/filogic.mk` |
| SoC, memory | MT7981B, 1 GB DDR4, 256 MiB SPI-NAND (system) + 16 MiB NOR (recovery) | [Device page](https://openwrt.org/toh/openwrt/one) |
| WAN | 2.5 GbE (Airoha EN8811H PHY, PoE in) → **`eth0`** | DTS `gmac0`; `board.d/02_network`: `ucidef_set_interfaces_lan_wan eth1 eth0` |
| LAN | 1 GbE (SoC PHY) → **`eth1`**, sole port of `br-lan` | Same |
| Wi-Fi | Wi-Fi 6, 2.4 + 5 GHz, `mt7915e` (mt76) | `DEVICE_PACKAGES` |
| Console | Front USB-C, 115200 8N1 | Device page |

There's no switch chip, so no DSA `lan1`…`lanN` ports: an external switch adds wired clients. **Other devices differ:** DSA routers expose `lanN`/`wan` ports, older swconfig devices use `eth0.1`/`eth0.2` with `config switch_vlan`, and each board's `board.d` file decides roles. Check the **exact model and revision** in the [Table of Hardware](https://openwrt.org/toh/start), since revisions of one model can need different images. On a running unit:

```sh
ubus call system board      # board_name "openwrt,one", release, target
cat /etc/board.json         # default lan/wan devices and radios
```

### Topology and address plan

```
              Internet
                 │
       upstream router / ONT        DHCP; ideally public IPv4 + IPv6 prefix delegation
                 │
      eth0  WAN 2.5G                wan:  203.0.113.10/24 (DHCP), gateway 203.0.113.1
                 │                  wan6: DHCPv6 + delegated 2001:db8:1200::/56
        ┌────────┴─────────┐
        │   OpenWrt One    │ )))  SSID "Home"       → br-lan
        │  br-lan br-guest │ )))  SSID "Home-Guest" → br-guest
        └────────┬─────────┘
      eth1  LAN 1G (port of br-lan)
                 │
          switch / PC / NAS 192.168.10.10
```

| Logical interface | Device | IPv4 | IPv6 (with a /56) | Zone |
|---|---|---|---|---|
| `lan` | `br-lan` = `eth1` + Home | 192.168.10.1/24 | 2001:db8:1200:10::/64 | lan |
| `guest` | `br-guest` = Home-Guest | 192.168.30.1/24 | 2001:db8:1200:30::/64 | guest |
| `wan` / `wan6` | `eth0` | DHCP client | DHCPv6 client + PD | wan |

Subnets match the [gateway note](../gateway/gateway.md#networks). The LAN leaves OpenWrt's default 192.168.1.0/24 because upstream ISP boxes often use it. WAN addresses are documentation placeholders: behind an ISP box you get a private one and double NAT.

---

## 2. Architecture

| Component | Role | Reads → produces |
|---|---|---|
| Kernel | Netdevs, bridges, routing, netfilter, conntrack | Runtime state |
| procd | PID 1: starts and supervises services, hotplug, reload triggers | `/etc/init.d/*` → running daemons |
| ubus | Message bus: `network.interface.lan`, `hostapd.phy1-ap0`, `dhcp`… | IPC between all of these |
| UCI | Config format, library, `uci` CLI | `/etc/config/*`, no runtime effect alone |
| netifd | Devices (bridges, VLANs), interfaces, addresses, routes. Protocol handlers: `udhcpc` (`dhcp`), `odhcp6c` (`dhcpv6`), `pppd` (`pppoe`). Adds Wi-Fi netdevs to bridges | `network`, `wireless` → netlink, ubus state, `/tmp/resolv.conf.d/resolv.conf.auto` |
| wifi-scripts + hostapd | netifd's wireless handler configures hostapd (`wpad-basic-mbedtls`) over ubus. hostapd creates `phyN-apM` and runs WPA | `wireless` → AP netdevs |
| dnsmasq | DHCPv4, DNS forwarder and cache | `dhcp` → `/var/etc/dnsmasq.conf.cfg*`, `/tmp/dhcp.leases` |
| odhcpd (`odhcpd-ipv6only`) | RA, DHCPv6, NDP relay, using prefixes from netifd | `dhcp` + ubus → RAs, `/tmp/odhcpd.leases` |
| firewall4 (`fw4`) | ucode program that turns zones and rules into nftables, re-run on interface events | `firewall` + ubus → `table inet fw4` |

As in the [gateway note](../gateway/gateway.md#who-does-what-on-the-gateway), each piece has one owner: a permanent `ip addr add` or a hand-written `nftables.conf` next to netifd and fw4 is overwritten at the next reload.

```
  persistent intent (flash)                runtime state (RAM)
  /etc/config/network  ─┐
  /etc/config/wireless ─┴─► netifd ─netlink─► eth0 eth1 br-lan br-guest, addresses, routes
                              │ ├─ udhcpc, odhcp6c (wan, wan6)
                              │ └─ wifi-scripts ─ubus─► hostapd ─► phy0-ap0 phy1-ap0 phy1-ap1
                              │                        (netifd adds them to br-lan / br-guest)
                              ▼ ubus: network.interface.* (state, addresses, prefixes, DNS)
  /etc/config/dhcp ───────► dnsmasq (DHCPv4, DNS)   odhcpd (RA, DHCPv6) ◄─ delegated prefixes
  /etc/config/firewall ───► fw4 ─nft -f─► table inet fw4 ◄─ re-run on interface events
           ▲
   uci / vi / LuCI          procd starts all of the above and re-triggers them on change
```

From intent to runtime:
1. `uci set` stages a change in `/tmp/.uci/`, and `uci commit` writes `/etc/config/<file>`. **Nothing changes on the network yet.**
2. `service network reload`, or `reload_config` (reloads every service whose file changed), makes the owner re-read. LuCI's *Save & Apply* does the same with an automatic rollback.
3. The owner applies the change and publishes the result on ubus. Later events (link, lease, new prefix) trigger hotplug, and fw4, dnsmasq and odhcpd follow on their own.

### Packet flow: a LAN client to the internet

Client 192.168.10.100 opens HTTPS to 1.1.1.1. It has ARPed 192.168.10.1, so the frame is addressed to the router's MAC:

```
 ① client ─frame to router MAC─► eth1 (wired) or phy1-ap0 (Wi-Fi), ports of br-lan
 ② br-lan  dst MAC = bridge itself → up to IP. (A frame for another LAN host is switched
           here and never reaches IP or fw4.)
 ③ prerouting   conntrack: new flow · dstnat (port forwards, wan ingress only)
        │
    routing decision (ip route get 1.1.1.1 → via 203.0.113.1 dev eth0)
        ├── destination is a router address (192.168.10.1, 203.0.113.10…)
        │      → input → input_lan → dnsmasq / dropbear / uhttpd, replies via output
        ▼
 ③ forward      ct established? accept · else forward_lan → jump accept_to_wan
 ④ postrouting  srcnat_wan: masquerade, source becomes 203.0.113.10:<port>
 ⑤ eth0 ──► 203.0.113.1

 ⑥ reply arrives on eth0 for 203.0.113.10:<port>
    prerouting  conntrack finds the flow, reverses the NAT → dst 192.168.10.100
    forward     ct state established → accept (zone chains not evaluated again)
    br-lan      FDB lookup of the client MAC → eth1 or phy1-ap0
```

- **Bridged** (LAN ↔ LAN, wired ↔ Wi-Fi in `br-lan`): switched by the bridge's MAC table. No routing, no fw4 (`br_netfilter` isn't loaded). Two clients on the same SSID are forwarded inside the AP, before the bridge, unless `isolate` is set.
- **Routed** (LAN → WAN, guest → LAN): filtered in `forward`, source NAT on WAN egress, destination NAT on WAN ingress.
- **Local** (DNS, DHCP, SSH, LuCI, ping to the router): prerouting, then input. **Every router address is local:** a LAN client reaching 203.0.113.10, or a guest reaching 192.168.10.1, goes through `input`, not `forward`.

The generic hook diagram is in the [gateway note](../gateway/gateway.md#how-a-packet-crosses-the-gateway).

---

## 3. Linux networking, the OpenWrt way

### Tools on the image

| Need | Default 25.12 image | Package |
|---|---|---|
| `ip link/addr/route/neigh/rule` | BusyBox `ip` (subset, e.g. no `-br`) | `ip-full` |
| `bridge link/fdb/vlan` | Missing (BusyBox `brctl show`) | `ip-bridge` |
| `ss` | Missing (BusyBox `netstat -lntup`) | `ss` |
| `tcpdump`, `conntrack`, `dig`, `ethtool`, `tracepath`, `iw` | Missing | `tcpdump` (`tcpdump-mini` on small flash), `conntrack`, `bind-dig`, `ethtool`, `iputils-tracepath`, `iw` |
| `nft` | Present (firewall4 dependency) | |
| `ping`, `ping6`, `traceroute`, `nslookup` | BusyBox | `iputils-ping` for `ping -M do` |

```sh
apk update && apk add ip-full ip-bridge ss tcpdump conntrack bind-dig ethtool iputils-tracepath
```

Packages added this way live in the overlay and a manual sysupgrade drops them. `/etc/apk/world` lists the explicitly installed packages (the image's and yours), and section 10 bakes them in. `/tmp` is RAM, so stream long captures to the host:

```sh
ssh root@192.168.10.1 "tcpdump -U -ni br-lan -w - 'not port 22'" | wireshark -k -i -
```

### Interfaces and naming

| Layer | Examples | Defined by | Inspect |
|---|---|---|---|
| Kernel netdev | `eth0`, `eth1`, `phy1-ap0` | Driver, hostapd | `ip link` |
| netifd device | `br-lan`, `eth1.30` | `config device` | `devstatus br-lan` |
| Logical interface | `lan`, `wan`, `wan6`, `guest` | `config interface` | `ifstatus lan`, `ubus call network.interface dump` |

Zones, DHCP pools and SSIDs all reference **logical interfaces**. `wan` and `wan6` are two logical interfaces on the same `eth0`, one per protocol handler. Wi-Fi netdevs are named `<phy>-ap<N>`. Which `phyN` is 5 GHz is **(verify on device)**: check `wifi status`.

### Bridges, routed interfaces and VLANs

`br-lan` holds 192.168.10.1/24, and its ports (`eth1`, `phy1-ap0`) are Layer 2 members without addresses. `br-lan` and `br-guest` are two routed interfaces, so traffic between them crosses routing and the firewall. **Devices that must be filtered from each other need different bridges and subnets**, not two ports of one bridge.

```sh
bridge link                  # which netdev belongs to which bridge
bridge fdb show br br-lan    # which MAC was learned on which port: wired or Wi-Fi?
```

A VLAN is a netifd device attached to an interface or bridge. There are two styles: an 802.1Q device on a port (`eth1.30`, section 6), or bridge VLAN filtering with `config bridge-vlan` (usual on DSA switches, see the [DSA mini-tutorial](https://openwrt.org/docs/guide-user/network/dsa/dsa-mini-tutorial)). `bridge vlan` shows the membership. For VLAN vs bridge vs subnet, see the [gateway note](../gateway/gateway.md#networks).

### Routing

```sh
ip route                     # default via 203.0.113.1 dev eth0 (from wan DHCP)
ip route get 1.1.1.1         # the path the kernel picks for one destination
ip -6 route                  # "default from 2001:db8:1200::/56 via fe80::… dev eth0"
ip rule                      # policy routing rules
```

**IPv6 default routes are source-specific** (`sourcefilter`, on by default for `wan6`): a packet only uses the default route if its source belongs to that uplink's prefix. So a client keeping an address from an expired prefix has IPv6 addressing but no way out. Policy routing uses `config rule` sections (`option src '192.168.30.0/24'`, `option lookup '100'`) plus `option ip4table '100'` on the second uplink. For multi-WAN with health checks, use `mwan3` or `pbr` (principle in the [gateway note](../gateway/gateway.md#failover)).

### ARP, Neighbor Discovery, DHCP and DNS

`ip neigh show dev br-lan`: REACHABLE/STALE are fine, FAILED/INCOMPLETE mean the router asked and got no answer (client gone, wrong VLAN, Wi-Fi associated but not passing data). OpenWrt sets `arp_ignore=1` (`/etc/sysctl.d/10-default.conf`): the router only answers ARP for addresses on the receiving interface. Neighbor Discovery is ICMPv6, so dropping ICMPv6 breaks IPv6 at Layer 2.

A dnsmasq lease gives the client its address, 192.168.10.1 as gateway (option 3) and DNS (option 6). The router resolves through dnsmasq on `127.0.0.1`, and dnsmasq forwards to the servers netifd learned on the WAN:

```sh
cat /tmp/dhcp.leases                       # expiry, MAC, IP, hostname
cat /tmp/resolv.conf.d/resolv.conf.auto    # upstream DNS from wan/wan6
nslookup openwrt.org 127.0.0.1
```

### Connection tracking, NAT and offloading

Same model as any Linux gateway: one conntrack entry per flow, NAT decided on the first packet, replies reversed automatically.

```sh
conntrack -L -s 192.168.10.100      # one client's flows; [UNREPLIED] = no answer came back
conntrack -C; sysctl net.netfilter.nf_conntrack_max
```

**Flow offloading** (`flow_offloading '1'`, or `flow_offloading_hw` for the MT7981 packet engine, in `config defaults`) fast-paths established TCP/UDP flows. Software offload skips the `forward` chain, so counters freeze. Hardware offload also hides packets from `tcpdump`. Both are off by default: keep them off while debugging.

### MTU

Ethernet carries 1500 bytes, PPPoE 1492, WireGuard typically 1420. Symptom of an MTU problem: ping and small pages work, large transfers or some sites hang. The `wan` zone's `mtu_fix '1'` (default) clamps the TCP MSS. UDP (QUIC, VPNs) still depends on ICMP "fragmentation needed" / "packet too big" getting through. From a Linux client:

```sh
ping -M do -s 1472 1.1.1.1   # 1472 + 28 = 1500: must pass on an Ethernet WAN
tracepath 1.1.1.1            # prints the discovered path MTU
```

---

## 4. Configuring the network with UCI

| File | Read by | Holds |
|---|---|---|
| `/etc/config/network` | netifd | Devices, interfaces, routes, rules |
| `/etc/config/wireless` | netifd → hostapd | Radios (`wifi-device`), SSIDs (`wifi-iface`) |
| `/etc/config/dhcp` | dnsmasq, odhcpd | DNS options, DHCPv4 pools, RA/DHCPv6, static leases |
| `/etc/config/firewall` | fw4 | Defaults, zones, forwardings, rules, redirects |

The files reference each other by logical interface name (`dhcp.lan.interface`, the zone's `network`, the SSID's `network`): renaming an interface touches all four.

```sh
uci show network.lan                 # values, including staged changes
uci set network.lan.ip6assign='64'
uci changes                          # staged, not yet in /etc/config
uci revert network                   # drop them
uci commit network                   # write the file (still no runtime effect)
service network reload               # netifd applies the difference
uci show firewall.@zone[1].name      # anonymous sections: type + index
```

Editing the files with `vi` works too, without `uci commit`. Name your own sections (`firewall.guest=zone`) so they don't depend on indexes. Don't add generic Linux configs (`/etc/network/interfaces`, `/etc/dnsmasq.conf`, `/etc/nftables.conf`): nothing reads them, or they fight the owner.

**No CLI rollback.** Before a change that could cut your SSH session, arm one (BusyBox has `setsid`, not `nohup`):

```sh
cp /etc/config/network /tmp/network.good; rm -f /tmp/keep
setsid sh -c 'sleep 180; [ -e /tmp/keep ] || { cp /tmp/network.good /etc/config/network; service network reload; }' &
touch /tmp/keep        # after the change, if you're still connected
```

### LAN bridge, static address and WAN

25.12 writes addresses in CIDR notation as a list. The generated `/etc/config/network`, with this note's changes:

```
config device
	option name 'br-lan'
	option type 'bridge'
	list ports 'eth1'                # Wi-Fi joins at runtime via wireless "network 'lan'"

config interface 'lan'
	option device 'br-lan'
	option proto 'static'
	list ipaddr '192.168.10.1/24'    # default 192.168.1.1/24
	option ip6assign '64'            # default 60 (section 7)
	option ip6hint '10'

config interface 'wan'
	option device 'eth0'
	option proto 'dhcp'              # udhcpc. Override DNS: peerdns '0' + list dns '9.9.9.9'

config interface 'wan6'
	option device 'eth0'
	option proto 'dhcpv6'            # odhcp6c: address + prefix delegation
```

```sh
uci del network.lan.ipaddr; uci add_list network.lan.ipaddr='192.168.10.1/24'
uci commit network && service network reload   # SSH drops: renew the laptop's lease, use .10.1
```

Static WAN: `proto 'static'`, `list ipaddr '203.0.113.10/24'`, `option gateway '203.0.113.1'`, `list dns '9.9.9.9'`. PPPoE: `proto 'pppoe'` with `username`/`password`, and the runtime WAN device becomes `pppoe-wan`.

### DHCP server for the LAN

```
config dhcp 'lan'
	option interface 'lan'
	option start '100'               # offsets in the subnet: .100 to .249
	option limit '150'
	option leasetime '12h'
	option dhcpv6 'server'           # odhcpd
	option ra 'server'

config host
	option name 'nas'
	option mac 'aa:bb:cc:dd:ee:ff'
	option ip '192.168.10.10'
```

dnsmasq serves DHCPv4 and DNS, and odhcpd serves RA and DHCPv6 from the same section (`service dnsmasq restart`, `service odhcpd restart`).

### The routed LAN → WAN result

Nothing else is needed to route. `/etc/sysctl.d/10-default.conf` already enables IPv4 and IPv6 forwarding, `wan` brings the default route, and the default firewall forwards `lan → wan` with masquerade (section 5):

```sh
ifstatus wan | jsonfilter -e '@["ipv4-address"][0].address' -e '@.route[0].nexthop'
ip route get 1.1.1.1 from 192.168.10.100 iif br-lan     # ip-full: the route a LAN packet takes
```

### Intended vs actual state

| Question | Intended (flash) | Actual (runtime) |
|---|---|---|
| `lan` address | `uci get network.lan.ipaddr` | `ip addr show br-lan`, `ifstatus lan` |
| Bridge members | `uci show network \| grep ports` | `bridge link`, `devstatus br-lan` |
| SSID → bridge | `uci show wireless` | `wifi status`, `bridge link` |
| DHCP | `uci show dhcp` | `/var/etc/dnsmasq.conf.cfg*` |
| Firewall | `uci show firewall`, `fw4 print` | `nft list ruleset` |
| Why they differ | | `logread -e netifd`, `logread -e dnsmasq`, warnings from `service firewall reload` |

**Older syntax:** before 21.02, bridges were declared in the interface (`option type 'bridge'`, `option ifname 'eth0.1'`) and switch ports used `config switch_vlan`. Since 21.02, `config device` and `option device`. Before 25.12, `option ipaddr` + `option netmask`. Up to 21.02, fw3/iptables: `/etc/firewall.user` scripts don't apply under fw4.

---

## 5. Firewall, forwarding and NAT

| Traffic | Example | fw4 chain | Controlled by |
|---|---|---|---|
| To the router | LAN → 192.168.10.1:53, WAN → 203.0.113.10:51820 | `input` → `input_<src zone>` | Zone `input`, rules with `src` only |
| From the router | apk, NTP, upstream DNS | `output` → `output_<dest zone>` | Zone `output` |
| Through the router | LAN client → 1.1.1.1 | `forward` → `forward_<src zone>` | `forwarding` sections, rules with `src` + `dest` |

### Zones and forwardings

A zone groups logical interfaces, which fw4 resolves to devices and subnets. The 25.12 default:

| Section | Settings |
|---|---|
| `defaults` | `input REJECT`, `output ACCEPT`, `forward REJECT`, `syn_flood 1` |
| zone `lan` | network `lan`; input, output, forward `ACCEPT` |
| zone `wan` | `wan wan6`; `input REJECT`, `output ACCEPT`, `forward DROP`, `masq 1`, `mtu_fix 1` |
| `forwarding` | `lan → wan` |
| rules from `wan` | Allow-DHCP-Renew, -Ping, -IGMP, -DHCPv6, -MLD, -ICMPv6-Input, -ICMPv6-Forward, -IPSec-ESP, -ISAKMP |

How to read it:
- A zone's `forward` policy only covers traffic **between interfaces of that same zone**.
- Between zones you need a one-way `forwarding` or a rule. Everything else falls to `defaults.forward`.
- `lan → wan` allows new connections outward, and the replies return through `ct state established`.
- `masq 1` is the IPv4 NAT. There's no IPv6 masquerade (`masq6`) by default.
- The default rules allow ping to the router from the WAN, plus some ICMPv6 (echo-request included) forwarded to LAN hosts.

### How fw4 becomes nftables

fw4 generates one `table inet fw4` for both families. In each zone chain, rules are emitted before forwardings, so a `REJECT` rule from `lan` to `wan` wins over the forwarding.

| UCI | nftables |
|---|---|
| `defaults` | Base chains `input`/`forward`/`output` (reject via `handle_reject`) |
| zone `lan` | `input_lan`, `forward_lan`, `output_lan` + `accept_to_lan`, `reject_from_lan`… |
| `forwarding lan → wan` | `jump accept_to_wan` in `forward_lan` |
| zone `masq` | `masquerade` in `srcnat_wan` (postrouting) |
| `redirect` | `dnat` in `dstnat_wan` (prerouting) + `ct status dnat accept` in `forward_wan` |
| zone `mtu_fix` | MSS clamp in `mangle_forward` / `mangle_postrouting` |

```sh
fw4 print                    # ruleset generated from UCI, not loaded
fw4 check                    # the same, validated with nft -c
service firewall reload      # apply, and read the warnings it prints
nft list ruleset             # what the kernel has
```

Abridged (counters on by default, exact text varies):

```
chain forward {
    type filter hook forward priority filter; policy drop;
    ct state vmap { established : accept, related : accept } comment "!fw4: Handle forwarded flows"
    iifname "br-lan" jump forward_lan comment "!fw4: Handle lan IPv4/IPv6 forward traffic"
    jump handle_reject
}
chain forward_lan {
    jump accept_to_wan comment "!fw4: Accept lan to wan forwarding"
    jump accept_to_lan
}
chain accept_to_wan {
    meta nfproto ipv4 oifname "eth0" ct state invalid counter packets 0 bytes 0 drop comment "!fw4: Prevent NAT leakage"
    oifname "eth0" counter packets 5120 bytes 812345 accept comment "!fw4: accept wan IPv4/IPv6 traffic"
}
chain srcnat_wan {
    meta nfproto ipv4 masquerade comment "!fw4: Masquerade IPv4 wan traffic"
}
```

### Permit, deny and port forwarding

Add to `/etc/config/firewall`, then `service firewall reload`:

```
config rule                           # permit: WireGuard on the router. No dest → input_wan
	option name 'Allow-WireGuard'
	option src 'wan'
	option proto 'udp'
	option dest_port '51820'
	option target 'ACCEPT'

config rule                           # deny: no DNS-over-TLS bypass. src + dest → forward_lan
	option name 'Block-DoT'
	option src 'lan'
	option dest 'wan'
	option proto 'tcp'
	option dest_port '853'
	option target 'REJECT'

config redirect                       # port forward: public 8443 → NAS 443
	option name 'NAS-HTTPS'
	option src 'wan'
	option src_dport '8443'
	option dest 'lan'
	option dest_ip '192.168.10.10'
	option dest_port '443'
	option proto 'tcp'
	option target 'DNAT'
```

`REJECT` makes clients fail fast (TCP reset or ICMP unreachable). `DROP` makes them time out.

The redirect allows new TCP connections **from any internet address** to 203.0.113.10:8443. `dstnat_wan` rewrites them to 192.168.10.10:443, `forward_wan` accepts them because `ct status dnat`, and replies return through conntrack. No other port or host is opened. `reflection '1'` (default) adds hairpin NAT for LAN clients that use the public address.

Security implications:
- The NAS's HTTPS service, its authentication and its updates are now your perimeter. Limit sources with `option src_ip` when you know them.
- A redirect does nothing behind CGNAT or double NAT unless upstream forwards too.
- IPv6 needs an allow rule instead (section 7).
- For your own remote access, one WireGuard port on the router exposes far less.

### Temporary experiments vs persistent rules

```sh
nft insert rule inet fw4 forward_lan ip saddr 192.168.10.50 counter drop   # cut one host now
nft -a list chain inet fw4 forward_lan; nft delete rule inet fw4 forward_lan handle <n>
```

Runtime rules vanish on `service firewall reload`, on reboot, and whenever fw4 reloads after a config or interface change. Permanent custom nftables go in `/etc/nftables.d/*.nft` (included in `table inet fw4`), or in a `config include` with `option type 'nftables'` and a `position` (`chain-pre`, `chain-post`, `table-post`…).

### Troubleshooting a dropped connection

1. **Path:** is the destination a router address (`input`) or beyond (`forward`)? `ip route get <dst> from <src> iif br-lan`.
2. **Counters:** `nft list chain inet fw4 forward_<zone>` before and after reproducing. The reject/drop counter that moved is the answer.
3. **Log the zone:** `uci set firewall.@zone[1].log='1'` (check the index), commit, reload, then `logread -f | grep -E 'forward:| in: '`. Prefixes look like `reject wan in:` or `drop wan forward:`.
4. **Trace one flow** through every chain, from a separate table that leaves fw4 untouched:
   ```sh
   nft add table inet trace
   nft add chain inet trace pre '{ type filter hook prerouting priority -350; }'
   nft add rule inet trace pre ip saddr 192.168.10.100 tcp dport 853 meta nftrace set 1
   nft monitor trace            # every chain/rule hit and the final verdict
   nft delete table inet trace
   ```
5. **Conntrack:** `conntrack -L -d <dst>`. `[UNREPLIED]` means it left and no reply came: upstream or return path, not filtering.

---

## 6. Wi-Fi, bridges and VLANs: a guest network

| Concept | OpenWrt / Linux object |
|---|---|
| Radio (one channel) | `config wifi-device 'radio1'` = wiphy `phy1` |
| Access point / BSS (one SSID on one radio) | `config wifi-iface` = netdev `phy1-ap0`, created by hostapd |
| The network it serves | `option network 'lan'` → netifd adds `phy1-ap0` to `br-lan` |

An SSID is a name in beacons, while the netdev is what Linux bridges and routes. Two SSIDs on one radio are two BSSes: two netdevs on one channel, within the driver's limits (`iw list`, see the [gateway note](../gateway/gateway.md#10-wi-fi-access-point-hostapd)). A client must associate, then complete the WPA handshake with hostapd. Only then do its frames come out of `phy1-ap0` as Ethernet into `br-lan`, following the section 2 path. **Associated isn't connected:** a failed handshake or the wrong bridge stops it before DHCP. Encryption values: `sae-mixed` (WPA2/WPA3), `psk2`, `sae`.

Wi-Fi is disabled on first boot. Keep the generated radio sections (their `path` is board-specific), and find the 5 GHz radio with `uci show wireless | grep band` **(verify on device; radio1 = 5 GHz assumed below)**:

```sh
uci set wireless.radio1.country='FR'
uci set wireless.default_radio1.ssid='Home'
uci set wireless.default_radio1.encryption='sae-mixed'
uci set wireless.default_radio1.key='change-me-long-passphrase'
uci set wireless.default_radio1.disabled='0'
uci commit wireless && wifi reload
```

### Guest network

Wi-Fi only, with its own bridge, subnet and zone (structure from the official [guest Wi-Fi guide](https://openwrt.org/docs/guide-user/network/wifi/guestwifi/configuration_command_line_interface)). Append to each file:

```
# /etc/config/network
config device 'guest_dev'
	option name 'br-guest'
	option type 'bridge'
	option bridge_empty '1'           # up even before the SSID joins

config interface 'guest'
	option device 'br-guest'
	option proto 'static'
	list ipaddr '192.168.30.1/24'
	option ip6assign '64'
	option ip6hint '30'

# /etc/config/wireless
config wifi-iface 'guest_radio1'
	option device 'radio1'
	option network 'guest'
	option mode 'ap'
	option ssid 'Home-Guest'
	option encryption 'sae-mixed'
	option key 'another-guest-passphrase'
	option isolate '1'

# /etc/config/dhcp
config dhcp 'guest'
	option interface 'guest'
	option start '100'
	option limit '150'
	option leasetime '2h'
	option ra 'server'
	option dhcpv6 'server'

# /etc/config/firewall
config zone
	option name 'guest'
	list network 'guest'
	option input 'REJECT'
	option output 'ACCEPT'
	option forward 'REJECT'

config forwarding
	option src 'guest'
	option dest 'wan'

config rule
	option name 'Allow-DNS-DHCP-Guest'
	option src 'guest'
	list proto 'tcp'
	list proto 'udp'
	list dest_port '53'
	list dest_port '67'
	list dest_port '547'
	option target 'ACCEPT'

config rule
	option name 'Allow-ICMPv6-Guest'  # RS/NS/NA: IPv6 needs them
	option src 'guest'
	option proto 'icmp'
	list icmp_type 'router-solicitation'
	list icmp_type 'neighbour-solicitation'
	list icmp_type 'neighbour-advertisement'
	list icmp_type 'echo-request'
	option family 'ipv6'
	option target 'ACCEPT'
```

```sh
service network reload && wifi reload && service dnsmasq restart && service odhcpd restart && service firewall reload
```

| Guest 192.168.30.100 → | Chain | Result | Why |
|---|---|---|---|
| 1.1.1.1 | `forward_guest` → `accept_to_wan`, `srcnat_wan` | ✅ NATed | `forwarding guest → wan` |
| 192.168.30.1 ports 53/67 | `input_guest` | ✅ | Allow-DNS-DHCP-Guest |
| 192.168.30.1 ports 22/80 | `input_guest` | ❌ | Zone `input REJECT` |
| 192.168.10.1 (router's LAN IP) | `input_guest`: still local | ❌ except 53/67/547 | Input, not forward |
| 192.168.10.10 (NAS) | `forward_guest` | ❌ | No `guest → lan` forwarding |
| Another guest on Home-Guest | Inside the AP | ❌ | `isolate '1'` |
| LAN → guest client | `forward_lan` | ❌ | No `lan → guest` forwarding (add one if wanted) |

**Client isolation and network isolation are different:**
- `isolate '1'` (hostapd `ap_isolate`) blocks station ↔ station on one BSS, inside the AP.
- The separate bridge, subnet and zone block guest ↔ LAN at Layer 3.

Guests need both. If the guest SSID also runs on the other radio, both BSSes join `br-guest`, and the bridge forwards between them: `isolate` doesn't cover that path.

### Wired guest ports with a VLAN

With a managed switch on `eth1`, the LAN stays untagged and the guest network travels as VLAN 30:

```
config device
	option type '8021q'
	option ifname 'eth1'
	option vid '30'
	option name 'eth1.30'
# then add  list ports 'eth1.30'  to the br-guest device
```

The switch uplink carries VLAN 30 tagged and the LAN untagged, and guest access ports are untagged in VLAN 30 (switch side as in the [gateway note](../gateway/gateway.md#9-switch-configuration)). Check with `ip -d link show eth1.30`, `bridge link` and `tcpdump -eni eth1 vlan 30`. On DSA routers, use `bridge-vlan` sections on one VLAN-filtering bridge instead.

---

## 7. IPv6

The concepts are summarized in the [gateway note](../gateway/gateway.md#ipv6-in-short). On OpenWrt:

| Piece | Who | Config |
|---|---|---|
| WAN address, default route, delegated prefix | `odhcp6c` via `wan6` | `reqaddress 'try'`, `reqprefix 'auto'` (defaults). `reqprefix '56'` asks for a size, but the ISP decides |
| Prefix split into LAN /64s | netifd | `ip6assign '64'` + `ip6hint` (hex subnet ID) |
| ULA `fdxx::/48` | netifd | `globals.ula_prefix`: on LANs even without ISP IPv6 |
| RA + DHCPv6 on LANs | odhcpd | `ra 'server'`, `dhcpv6 'server'`, `ra_flags 'managed-config other-config'` (default) |

```sh
ifstatus wan6               # "ipv6-prefix": 2001:db8:1200::/56, "assigned": { lan, guest }
ip -6 addr show br-lan      # 2001:db8:1200:10::1/64 + ULA + fe80::
ubus call dhcp ipv6leases
```

**ISPs vary; never assume prefix delegation:**

| Upstream gives | Result | What to do |
|---|---|---|
| /56 or /60 | One /64 per LAN, as hinted | Nothing |
| A single /64 | Only one LAN gets global IPv6 | Give it to `lan`. Guest keeps IPv4 + ULA |
| No prefix, only an RA/address on WAN | ULA-only LANs, so clients prefer IPv4 | Relay mode: `ra`/`dhcpv6`/`ndp 'relay'` on the LANs, plus a `dhcp 'wan6'` section with `master '1'` ([IPv6 config](https://openwrt.org/docs/guide-user/network/ipv6/configuration)) |

**Don't carry IPv4 NAT habits over.** LAN hosts have global addresses. Protection comes from the WAN zone's `forward DROP` and the absence of a `wan → lan` forwarding. There's no port forward: open a host with a rule. Because the prefix can change, match only the interface ID with fw4's negative prefix length. The host needs a stable ID (static token or DHCPv6 `hostid`), not a privacy address:

```
config rule
	option name 'Allow-NAS-HTTPS-v6'
	option src 'wan'
	option dest 'lan'
	option dest_ip '::10/-64'         # any prefix, interface ID ::10
	option proto 'tcp'
	option dest_port '443'
	option family 'ipv6'
	option target 'ACCEPT'
```

`masq6` and NPTv6 exist for special cases (e.g. multi-WAN without PD). They break end-to-end addressing and add no security.

**Testing:** on the router, `ping6 -c3 2606:4700:4700::1111`. On a client, `ping -6 2606:4700:4700::1111` and `curl -6 https://ifconfig.co`, which must show the client's own address.

**A client has an IPv6 address but can't reach IPv6 destinations:**
1. **Address:** `ip -6 addr` on the client. Only `fd…`/`fe80…`, or a prefix different from `ifstatus wan6`, means no or stale delegation.
2. **Default route:** `ip -6 route show default` must show `via fe80::… proto ra`. If it's missing, RAs aren't arriving: `tcpdump -ni br-lan 'icmp6 and ip6[40] == 134'`.
3. **Neighbors:** the router's link-local must be REACHABLE on the client (`ip -6 neigh`), and the client on the router (`ip -6 neigh show dev br-lan`).
4. **Router route:** `ip -6 route get 2606:4700:4700::1111 from <client address>` must hit the source-specific default.
5. **Wire:** `tcpdump -ni eth0 icmp6 and host 2606:4700:4700::1111`. The request leaves with the client's own source. No reply means the ISP isn't routing that prefix to you.
6. **Firewall:** `accept_to_wan` counters. ICMPv6 packet-too-big must pass, and the default rules allow it.

---

## 8. Troubleshooting

| # | Step | Commands | Proves | Doesn't prove |
|---|---|---|---|---|
| 1 | Client config | Client: `ip addr`, `ip route`, `resolvectl status` | Address in 192.168.10.0/24, gateway and DNS .1 | That this router issued it (169.254.x = no lease) |
| 2 | Layer 2 | Client `ip neigh`; router `ip neigh show dev br-lan`, `bridge fdb show br br-lan` | Frames reach the bridge on the expected port | Anything above L2 |
| 3 | Addressing / routing | Router `ip route get <client>`, `ip route get 1.1.1.1` | Both directions are routable | That the firewall allows them |
| 4 | Router reachable | `ping 192.168.10.1` | L2 + router **input** | Forwarding |
| 5 | External IP | `ping 1.1.1.1`, `nc -vz 1.1.1.1 443` | Forward, NAT, upstream for that protocol | DNS, large packets, other ports |
| 6 | DNS separately | `nslookup openwrt.org 192.168.10.1`, `dig @9.9.9.9 openwrt.org` | Local resolver vs upstream path | Application reachability |
| 7 | Capture | `tcpdump` at the points below | Where the packet disappears | Why |
| 8 | Firewall / NAT | Counters, zone log, trace (section 5) | Which rule decided | Upstream behaviour |
| 9 | Return path | Reply on `eth0`? `conntrack -L`: ESTABLISHED vs `[UNREPLIED]` | The flow is symmetric | Application errors |

A successful ping only proves ICMP echo on that path: TCP ports, DNS, MTU and IPv6 fail independently.

| Capture on | What you see | Establishes |
|---|---|---|
| `phy1-ap0` | A Wi-Fi client's frames after decryption | The client sends data, not only associates |
| `eth1` | Wired frames, VLAN tags with `-e` | The switch delivers the right VLAN |
| `br-lan` | Everything entering the LAN bridge, untagged | The packet reached L3 |
| `eth0` | Traffic after SNAT (source 203.0.113.10) | It left; a reply did or didn't come back |
| `any` | All interfaces, with name and direction (tcpdump 4.99) | Both sides of the router in one capture |

### Playbooks

**No DHCP lease:** `tcpdump -eni br-lan 'udp port 67 or udp port 68'` and `logread -e dnsmasq`.
- **DISCOVER, no OFFER:** dnsmasq isn't serving `lan` (pool full, `ignore`, config error). Or it detected another DHCP server: it logs `found already running DHCP-server on interface` and stays silent unless `option force '1'`.
- **DISCOVER only on `phy1-ap0`, not on `br-lan`:** bridge membership.
- **Nothing at all:** client, Wi-Fi authentication, or a switch VLAN mismatch.
- More detail: `uci set dhcp.@dnsmasq[0].logdhcp='1'`.

**Reaches the router, not the internet** (the router itself pings 1.1.1.1): `tcpdump -ni any icmp and host 1.1.1.1`.

| Seen | Meaning |
|---|---|
| Request on `br-lan` only | Dropped inside: forward rule, or no route |
| Request on `eth0` from 192.168.10.x | No masquerade: interface not in the `wan` zone, or `masq` off |
| Request on `eth0` from 203.0.113.10, no reply | Upstream |
| Reply on `eth0`, never on `br-lan` | Conntrack or return path |

**DNS fails, IP works:** `tcpdump -ni any port 53`.
- **Queries on `br-lan`, none on `eth0`:** dnsmasq has no upstream (check `resolv.conf.auto`), or it blocks the answer. Rebind protection drops upstream answers with private IPs and logs `possible DNS-rebind attack detected`.
- **Queries leave without answers:** upstream DNS.
- **Client queries 8.8.8.8 directly:** that's forwarding, not dnsmasq.
- Temporary per-query log: `logqueries='1'`.

**Leaves the LAN, never reaches the WAN:** counters and trace (section 5). Also rule out flow offloading and a missing route (`ip route get`).

**Wrong routes:**
- Two default routes (a second uplink without `option metric`).
- A LAN subnet overlapping the WAN subnet (both 192.168.1.0/24 behind an ISP box): `ip route` shows two connected routes. Renumber the LAN.
- Static routes go in `/etc/config/network`: `config route` with `interface`, `target`, `gateway`.

**MTU:** big transfers stall. Look at the SYNs leaving the WAN (`tcpdump -ni eth0 'tcp[tcpflags] & tcp-syn != 0'` shows `mss`), then run the client tests from section 3. **IPv6:** use the workflow in section 7.

**Wi-Fi associated, nothing works:**
```sh
ubus call hostapd.phy1-ap0 get_clients   # "authorized": false = handshake failed
logread -e hostapd
tcpdump -eni phy1-ap0                     # data or DHCP frames from that MAC?
bridge link                               # is phy1-ap0 in the intended bridge?
```
- **Not authorized:** usually the key, or a device that fails WPA3: try `psk2` on a separate SSID.
- **Authorized but no lease:** check the SSID's `network` option.
- **Clients can't see each other:** `isolate` on the main SSID.

**Logs** live in a RAM ring buffer (`logread`, `logread -f`, `logread -e <pattern>`, `dmesg`) and are lost on reboot. To keep them, use remote syslog: `uci set system.@system[0].log_ip='<server>'`.

---

## 9. Build, install and recover

The **build host** (x86_64 Debian 13, glibc, never as root) compiles a toolchain and produces images and `.apk` packages for the **target** (`aarch64_cortex-a53`, musl, BusyBox). Nothing is compiled on the router.

Use a stable tag (`v25.12.x`) for anything you flash. To install your own packages on a running router, build **its exact tag**: release kmods only load on the kernel they were built for. Snapshots (`main`) move daily, without a stable package ABI.

```sh
# Debian 13 dependencies (official list without the optional ccache/mold/clang; make reports anything missing)
sudo apt install bc binutils-gold bison build-essential ecj fastjar file flex g++ gawk gettext git \
  help2man libbsd-dev libelf-dev liblzma-dev libncurses-dev libssl-dev meson mtd-utils ninja-build \
  pbzip2 pigz pkg-config python3-dev python3-setuptools rsync subversion swig texinfo time \
  u-boot-tools unzip wget xsltproc xxd zlib1g-dev zstd

git clone https://git.openwrt.org/openwrt/openwrt.git     # mirror: github.com/openwrt/openwrt
cd openwrt && git checkout v25.12.6
./scripts/feeds update -a && ./scripts/feeds install -a
make menuconfig     # Target System: MediaTek ARM · Subtarget: Filogic 8x0 (MT798x) · Target Profile: OpenWrt One
make download && make -j"$(nproc)" world   # on failure: make -j1 V=s
ls bin/targets/mediatek/filogic/
```

To reproduce the release configuration, save the release's `config.buildinfo` as `.config`. It enables every profile of the subtarget, so narrow it to OpenWrt One in `menuconfig`.

| File (`openwrt-mediatek-filogic-openwrt_one-…`) | Use |
|---|---|
| `squashfs-sysupgrade.itb` | Upgrade from running OpenWrt (`sysupgrade`, LuCI, USB upgrade) |
| `factory.ubi` + `snand-preloader.bin` | Rewrite the whole NAND from the NOR recovery system |
| `initramfs.itb` | Boot from RAM without touching flash: test a build safely |
| `nor-*` | The NOR recovery itself: only when the device page says so |

A **factory** image is whatever the vendor firmware or bootloader accepts for a first install. A **sysupgrade** image upgrades a device already running OpenWrt, and carries board metadata: `sysupgrade` refuses another board's image, and you should never override that with `-F`. The One ships with OpenWrt, so normal installs are sysupgrades.

```sh
sysupgrade -b /tmp/backup-$(date +%F).tar.gz        # /etc/config + files from sysupgrade -l
scp -O root@192.168.10.1:/tmp/backup-*.tar.gz .      # -O: no SFTP server on OpenWrt by default
cat /etc/apk/world                                   # explicitly installed packages, yours included

sha256sum -c sha256sums --ignore-missing             # host: verify the download
scp -O openwrt-25.12.6-mediatek-filogic-openwrt_one-squashfs-sysupgrade.itb root@192.168.10.1:/tmp/
ubus call system board | grep board_name             # router: must be "openwrt,one"
sysupgrade -T /tmp/openwrt-25.12.6-*-sysupgrade.itb  # test only
sysupgrade -v /tmp/openwrt-25.12.6-*-sysupgrade.itb  # keeps config; -n wipes it
```

`owut upgrade` and LuCI's *Attended Sysupgrade* instead request an image with your packages from the public build service (`sysupgrade.openwrt.org`), which receives your package list.

**Access:** `ssh root@192.168.10.1` (dropbear: set `passwd`, keys in `/etc/dropbear/authorized_keys`). Serial: `picocom -b 115200 /dev/ttyACM0` on the front USB-C gives U-Boot, the kernel log and a shell even with a broken network config.

**Recovery, least to most invasive:**
1. **Config mistake:** fix `/etc/config/*` over serial, or restore with `sysupgrade -r backup.tar.gz`.
2. **[Failsafe](https://openwrt.org/docs/guide-user/troubleshooting/failsafe_and_factory_reset):** `f` + Enter on the console when prompted at boot, or a button press while the LED blinks fast. The router comes up at 192.168.1.1 on the LAN port, without DHCP. Set the laptop to 192.168.1.2/24, then `ssh root@192.168.1.1` and run `mount_root` to fix files, or `firstboot -y && reboot` to factory reset.
3. **Broken NAND firmware:** the USB reflash or initramfs recovery boot from the [device page](https://openwrt.org/toh/openwrt/one).
4. **Broken NAND bootloader:** boot switch to NOR, USB stick with `factory.ubi` + `snand-preloader.bin`.
5. **NOR itself:** UART boot with `mtk_uartboot` + TFTP. That's the last resort: the NOR is write-protected by a jumper for a reason.

Button and LED sequences come from the device page and are **(verify on device)**.

---

## 10. Extending the image

| Way | Effort | Notes |
|---|---|---|
| [Firmware Selector](https://firmware-selector.openwrt.org) / `owut` | None | Package list sent to the public build service |
| [Image Builder](https://openwrt.org/docs/guide-user/additional-software/imagebuilder) | Minutes | `openwrt-imagebuilder-25.12.6-mediatek-filogic.Linux-x86_64.tar.zst`, prebuilt packages |
| Full build (section 9) | Hours the first time | Required for kernel, driver or DTS changes |

```sh
# Image Builder (prefix a package with - to remove it)
make image PROFILE=openwrt_one PACKAGES="tcpdump ip-full ip-bridge ss conntrack bind-dig ethtool" FILES=files/
```

Both tools copy a `files/` tree into the root filesystem. Scripts in `files/etc/uci-defaults/` run once at first boot (and are deleted when they exit 0), which is the clean way to ship this note's address plan:

```sh
# files/etc/uci-defaults/90-lan-addressing
uci -q batch <<'EOF'
delete network.lan.ipaddr
add_list network.lan.ipaddr='192.168.10.1/24'
set network.lan.ip6assign='64'
set network.lan.ip6hint='10'
commit network
EOF
```

**One package:** `./scripts/feeds search tcpdump`, then `make package/tcpdump/{clean,compile} V=s` (after one full build). The `.apk` lands in `bin/packages/aarch64_cortex-a53/packages/`.
- **Recipes:** core ones under `package/` (`package/network/config/{netifd,firewall4}`, `package/network/services/{dnsmasq,odhcpd,hostapd}`), feeds under `feeds/<feed>/` (`feeds/packages/net/tcpdump/Makefile`).
- **Sources:** unpacked in `build_dir/target-aarch64_cortex-a53_musl/<pkg>-<version>/`. Patches go in the recipe's `patches/` ([quilt workflow](https://openwrt.org/docs/guide-developer/toolchain/use-patches-with-buildsystem)).
- **Installing your build:** `apk add --allow-untrusted /tmp/<pkg>.apk`, only on firmware from the same tag.

**A missing driver or feature:**
1. Identify the hardware and driver: `dmesg`, `ls /sys/class/net`, `ethtool -i eth1`, `iw list`.
2. Look for a packaged module: `apk search kmod-<name>`, or grep the `CONFIG_` symbol in `package/kernel/linux/modules/*.mk`, then select it under `menuconfig` → *Kernel modules*.
3. Not packaged: `make kernel_menuconfig`, or a new `KernelPackage` definition. A custom kernel can't use release kmods, so build everything it needs from the same tree.

| Change | Location (25.12, mediatek/filogic) |
|---|---|
| Default LAN/WAN roles | `target/linux/mediatek/filogic/base-files/etc/board.d/02_network` |
| Device tree (ports, PHYs, LEDs, buttons) | `target/linux/mediatek/dts/mt7981b-openwrt-one.dts` |
| Image recipe, device packages | `target/linux/mediatek/image/filogic.mk` (`Device/openwrt_one`) |
| Subtarget kernel config | `target/linux/mediatek/filogic/config-6.12` (via `make kernel_menuconfig`) |
| Kernel patches | `target/linux/mediatek/patches-6.12/`; all targets: `target/linux/generic/{backport,pending,hack}-6.12/` |
| Wi-Fi driver | `package/kernel/mt76` |
| Your default config | `files/etc/uci-defaults/` at the top of the build tree |

---

## 11. Quick reference

| Task | Commands |
|---|---|
| Network inspection | `ip link` · `ip addr` · `ifstatus <iface>` · `devstatus <dev>` · `ubus call network.interface dump` · `ubus call system board` |
| UCI | `uci show <file>[.<section>]` · `uci set` / `add_list` / `del` · `uci changes` · `uci revert` · `uci commit` · `service network reload` · `reload_config` |
| DHCP and DNS | `cat /tmp/dhcp.leases` · `ubus call dhcp ipv6leases` · `cat /tmp/resolv.conf.d/resolv.conf.auto` · `nslookup <name> 127.0.0.1` · `logdhcp` / `logqueries` |
| Routing | `ip route` · `ip -6 route` · `ip route get <dst> [from <src> iif <dev>]` · `ip rule` · `ip neigh` |
| Bridges and VLANs | `bridge link` · `bridge fdb show br br-lan` · `bridge vlan` · `ip -d link show eth1.30` |
| Wi-Fi | `wifi status` · `wifi reload` · `ubus call hostapd.<ifname> get_clients` · `iw list` |
| Firewall and NAT | `fw4 print` · `fw4 check` · `service firewall reload` · `nft list ruleset` · `nft list chain inet fw4 forward_lan` · `conntrack -L` · `nft monitor trace` |
| Packet capture | `tcpdump -ni any …` · `tcpdump -eni br-lan 'port 67 or port 68'` · `tcpdump -ni any port 53` · `tcpdump -ni eth0 icmp6` · `ssh … tcpdump -U -w - \| wireshark -k -i -` |
| Logs | `logread` · `logread -f` · `logread -e <pattern>` · `dmesg` |
| Packages | `apk update` · `apk add` · `apk search` · `cat /etc/apk/world` |
| Build | `git checkout v25.12.6` · `./scripts/feeds update -a && ./scripts/feeds install -a` · `make menuconfig` · `make download && make -j$(nproc) world` · `make package/<pkg>/compile V=s` |
| Upgrade and recovery | `sysupgrade -b` / `-T` / `-v` / `-r` · `owut upgrade` · `scp -O` · failsafe: `mount_root`, `firstboot -y` · `picocom -b 115200 /dev/ttyACM0` |

---

## References

- OpenWrt One: <https://openwrt.org/toh/openwrt/one>
- Network configuration: <https://openwrt.org/docs/guide-user/network/network_configuration>
- DSA mini-tutorial: <https://openwrt.org/docs/guide-user/network/dsa/dsa-mini-tutorial>
- Firewall configuration (fw4): <https://openwrt.org/docs/guide-user/firewall/firewall_configuration>
- IPv6 configuration: <https://openwrt.org/docs/guide-user/network/ipv6/configuration>
- Wireless configuration: <https://openwrt.org/docs/guide-user/network/wifi/basic>
- Build system: <https://openwrt.org/docs/guide-developer/toolchain/install-buildsystem>, <https://openwrt.org/docs/guide-developer/toolchain/use-buildsystem>
- Sources: `git.openwrt.org/openwrt/openwrt.git` (branch `openwrt-25.12`), firewall4: `git.openwrt.org/project/firewall4.git`
