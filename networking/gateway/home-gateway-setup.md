# Building a Home Gateway on Linux

How to turn a small Linux box (mini-PC, Raspberry Pi with a USB NIC, or an old x86 machine) into a full home gateway: routing, NAT, firewall, DHCP, DNS, IPv6, network segmentation, and Wi-Fi access points.

The goal is to reproduce everything a consumer ISP box does, but in a way you can understand, audit and extend.

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
| Wi-Fi | Access points broadcasting one SSID per network | hostapd or dedicated APs |

---

## 2. Target topology

```
                 Internet
                    │
             ┌──────┴──────┐
             │  ISP modem  │  (bridge mode / ONT if possible)
             └──────┬──────┘
                    │ eth0 (WAN): public IP via DHCP or PPPoE
             ┌──────┴──────┐
             │   Gateway   │  Linux: routing, NAT, firewall, DHCP, DNS
             └──────┬──────┘
                    │ eth1 (LAN trunk): VLAN 10/20/30 tagged
             ┌──────┴──────┐
             │ Managed     │
             │ switch      │
             └─┬────┬────┬─┘
               │    │    │
            PC/NAS  AP1  AP2   (APs get a trunk port and map one SSID per VLAN)
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
- **Access points**: dedicated APs (OpenWrt-capable, UniFi, Omada...) are far better than a Wi-Fi card in the gateway. Most PCIe/USB Wi-Fi cards have poor AP-mode support. Section 9 covers both options.
- CPU: any modern x86 or ARM64 board routes 1 Gbit/s with nftables without trouble.

OS: any Debian/Ubuntu server works. Install the packages:

```bash
sudo apt install nftables dnsmasq hostapd iproute2 conntrack tcpdump
sudo systemctl enable --now systemd-networkd nftables
# Disable anything else managing interfaces (NetworkManager, ifupdown, netplan renderers)
```

---

## 4. Interfaces and VLANs (systemd-networkd)

Layout: `eth0` = WAN, `eth1` = trunk to the switch. Each VLAN goes into its own bridge (`br-lan`, `br-iot`, `br-guest`), so a local Wi-Fi interface can join the same network later.

### WAN

`/etc/systemd/network/10-wan.network`

```ini
[Match]
Name=eth0

[Network]
DHCP=yes
IPv6AcceptRA=yes

[DHCPv4]
UseDNS=no            # we run our own resolver

[DHCPv6]
UseDNS=no
PrefixDelegationHint=::/56   # ask the ISP for a /56 (ISP-dependent, /60 or /64 also common)

[IPv6AcceptRA]
UseDNS=no
```

> **PPPoE**: many fiber/DSL ISPs require PPPoE. Use `pppd` with the `rp-pppoe` plugin on top of `eth0` (often on a VLAN like `eth0.835` for some ISPs) and treat `ppp0` as the WAN in the firewall. Remember the MSS clamping rule in section 6.

### LAN trunk and VLANs

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

---

## 5. Kernel settings

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

## 6. Firewall and NAT (nftables)

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

### Notes on the rules

- **IPv6 has no NAT here, and that's fine.** LAN devices get public IPv6 addresses, but the `forward` chain only lets in replies (`established,related`). Unsolicited inbound IPv6 is dropped by the policy. This is what protects you, not NAT.
- **Guest client isolation** (guests not seeing each other) happens on the access point (`ap_isolate`, see section 9), because traffic between two guests on the same SSID never reaches the gateway.
- **Don't expose SSH or the admin UI on the WAN.** Use a VPN (WireGuard) for remote access instead of port forwards.
- **UPnP/NAT-PMP**: leave it off. If a game console needs it, run `miniupnpd` restricted to that console's IP only.
- **Hairpin NAT** (reaching your public IP from inside): easier to solve with local DNS (`nas.home.arpa` → private IP) than with extra NAT rules.

---

## 7. DHCP and DNS (dnsmasq)

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

---

## 8. Switch configuration

On the managed switch:

| Port | Mode | VLANs |
|---|---|---|
| Uplink to gateway `eth1` | Trunk | 10, 20, 30 tagged |
| Access points | Trunk | 10, 20, 30 tagged (or 10 untagged for AP management) |
| PC / NAS | Access | 10 untagged |
| Smart TV, wired IoT | Access | 20 untagged |

Move the switch's own management IP into VLAN 10, and change its default admin password.

---

## 9. Access points

### Option A: dedicated APs (recommended)

Run the APs as "dumb APs": they bridge Wi-Fi to the wired network, and the gateway does all the routing, DHCP and firewalling.

On each AP:

1. **Disable** its DHCP server, firewall and NAT (on OpenWrt: delete the `wan` interface, disable `dnsmasq` and `firewall`).
2. Give it a static management IP in VLAN 10 (e.g. `192.168.10.2`, `.3`, ...) with the gateway as its gateway/DNS.
3. Create one bridge per VLAN on the uplink port and map each SSID to one:

| SSID | VLAN | Security | Extras |
|---|---|---|---|
| `Home` | 10 | WPA2/WPA3 (SAE mixed) | 802.11r/k/v for roaming |
| `Home-IoT` | 20 | WPA2-PSK (many IoT devices don't support WPA3) | 2.4 GHz only is fine |
| `Home-Guest` | 30 | WPA2/WPA3 | **Client isolation on** |

### Option B: Wi-Fi card in the gateway (hostapd)

Only if your card supports AP mode well (`iw list` → "Supported interface modes" contains `AP`). Mediatek MT7921/MT7922 and some Atheros chips work, while most Intel cards are limited to 2.4 GHz in AP mode.

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

### Wi-Fi planning tips

- **2.4 GHz**: only channels 1, 6 or 11, 20 MHz wide. Wider channels on 2.4 GHz only cause interference.
- **5 GHz**: 80 MHz wide. Channels 36-48 avoid DFS; DFS channels (52-144) are less crowded but the AP must go quiet briefly on radar detection.
- **Multiple APs**: same SSID and password on all, different non-overlapping channels, **wired backhaul** whenever possible.
- **Don't max out transmit power.** Medium power with more APs gives better roaming than one loud AP.
- Use a Wi-Fi analyzer app to check neighboring networks before picking channels.

---

## 10. Hardening checklist

- [ ] SSH: key-only auth (`PasswordAuthentication no`), `PermitRootLogin no`, listen on the LAN IP only.
- [ ] No service listening on the WAN: `sudo ss -lntup` and check every line.
- [ ] Automatic security updates (`unattended-upgrades`).
- [ ] Time sync (`systemd-timesyncd` or `chrony`). TLS and logs depend on it.
- [ ] Default passwords changed on the switch, the APs and the ISP modem.
- [ ] Back up `/etc/systemd/network`, `/etc/nftables.conf`, `/etc/dnsmasq.d`, `/etc/hostapd` (ideally in a private git repo).
- [ ] Remote access through WireGuard, not port forwards.
- [ ] Firmware updates on APs and switch.

---

## 11. Verification

From the gateway:

```bash
ip -br addr                       # all bridges up with the right IPs
ip -6 route                       # delegated prefix routes on each bridge
sudo nft list ruleset             # rules loaded
sudo conntrack -L | head          # NAT and connection tracking in action
journalctl -k | grep "nft "       # dropped packets (from the log rules)
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
| Guest | another guest | ❌ blocked (AP isolation) |
| Internet | any LAN IP (IPv4 or IPv6) | ❌ blocked |

For the last one, scan your public IPs from outside (e.g. `nmap -Pn <wan-ip>` from a VPS, or an online IPv6 port scanner). Nothing should answer except your explicit port forwards.

---

## 12. Troubleshooting

| Symptom | Likely cause |
|---|---|
| Clients get no IP | dnsmasq not listening on the bridge, firewall missing UDP 67, VLAN not tagged on the switch port |
| IP but no internet | `ip_forward` off, missing `masquerade`, WAN has no default route |
| Internet by IP but not by name | DNS port 53 not allowed in `input`, dnsmasq has no upstream (`no-resolv` without `server=`) |
| Some websites hang (IPv4) | MTU/MSS issue with PPPoE: check the MSS clamping rule |
| IPv6 doesn't work | ICMPv6 blocked, ISP doesn't delegate a prefix (check `networkctl status eth0`), wrong `PrefixDelegationHint` |
| Wi-Fi clients on wrong network | SSID mapped to the wrong VLAN, or AP port not a trunk |

Handy capture commands:

```bash
sudo tcpdump -ni br-lan port 67 or port 68     # DHCP
sudo tcpdump -ni eth0 icmp6                    # IPv6 neighbor discovery / RAs on WAN
sudo tcpdump -ni eth0 -e vlan                  # verify VLAN tags
```

---

## References

- nftables wiki: <https://wiki.nftables.org>
- systemd-networkd: `man systemd.network`, `man systemd.netdev`
- dnsmasq manual: <https://thekelleys.org.uk/dnsmasq/docs/dnsmasq-man.html>
- hostapd example config: <https://w1.fi/cgit/hostap/plain/hostapd/hostapd.conf>
- RFC 4890: recommendations for filtering ICMPv6 in firewalls
- RFC 8375: `home.arpa` for home networks
