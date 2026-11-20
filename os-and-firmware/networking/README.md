# Networking

Notes on routing, firewalling, Wi-Fi and network services, with a focus on setups you can reproduce at home or in a lab.

## Notes

### Networking basics

- [Networking Basics: How Networks Work](networking-basics/networking-basics_v1_2026-11-20.md): the OS-neutral concepts from scratch: network types and devices, the OSI and TCP/IP models, MAC and IP addresses, subnetting, IPv6, switching and VLANs, routing, TCP/UDP, DHCP, DNS, NAT, HTTPS, firewalls, VPNs and Wi-Fi, with example architectures from a home LAN to a data center and a cloud VPC. Start here if you're new to networking.

### Linux networking fundamentals

- [Linux Networking Fundamentals for Embedded Engineers](linux-networking-fundamentals/linux-networking-fundamentals_v1_2026-10-29.md): layers, Ethernet and MAC addresses, IP subnetting, ARP, routing, DHCP, DNS, TCP/UDP, ICMP, HTTPS, NAT and firewalls, seen from an embedded device, with an end-to-end packet walk and a bottom-up troubleshooting guide.

### Gateway

- [Building a Home Gateway on Linux](gateway/gateway_v1_2026-10-26.md): routing, NAT, firewall (nftables, iptables, firewalld), DHCP/DNS, IPv6, VLANs, hostapd Wi-Fi, and backup uplinks (Wi-Fi station, LTE, Starlink).

### OpenWrt

- [Networking on OpenWrt](openwrt/openwrt_v1_2026-10-29.md): netifd, UCI, firewall4/nftables, dnsmasq/odhcpd, Wi-Fi and guest networks, IPv6, packet-level troubleshooting, and building/recovering firmware (OpenWrt 25.12 on the OpenWrt One).
