# Wi-Fi for Embedded Linux Engineers

How Wi-Fi works from the radio up to the handshake, how Linux implements it (cfg80211, mac80211, wpa_supplicant, hostapd), and what it takes to put a Wi-Fi chip on an embedded board and debug it when the link is slow, drops or never comes up.

This note covers the **wireless link**: everything below IP. What happens once the device has an address (DHCP, DNS, routing, NAT) is in [Networking Fundamentals for Embedded Linux Engineers](../../networking/fundamentals/fundamentals.md). Router-side Wi-Fi configuration is in [Building a Home Gateway on Linux](../../networking/gateway/gateway.md#10-wi-fi-access-point-hostapd) (hostapd) and [Networking on OpenWrt](../../networking/openwrt/openwrt.md#6-wi-fi-bridges-and-vlans-a-guest-network) (UCI), so this note links there instead of repeating it.

**Assumptions:**
- An example network (section 2): the gateway's access point and a board with an SDIO Wi-Fi module.
- Linux with `iw`, `wpa_supplicant` 2.10+ and `hostapd` 2.10+. BusyBox-based images are covered where they differ.
- Regulatory values are given for the EU (country `FR`) and the US. They change: check your regulator before shipping a product.

**How to read it:**
- **To learn:** Part 1 (sections 1 to 10) in order.
- **To bring up Wi-Fi on Linux:** Part 2 (sections 11 to 15).
- **To integrate a chip in a product:** Part 3 (sections 16 and 17).
- **To debug:** section 18 first, then the section of the step that fails.

## Table of contents

**Part 1: How Wi-Fi works**

- [1. The mental model: what 802.11 changes compared to Ethernet](#1-the-mental-model-what-80211-changes-compared-to-ethernet)
- [2. Example network](#2-example-network)
- [3. Generations and standards](#3-generations-and-standards)
- [4. Bands, channels and regulatory rules](#4-bands-channels-and-regulatory-rules)
- [5. The radio link: OFDM, MCS, MIMO and signal levels](#5-the-radio-link-ofdm-mcs-mimo-and-signal-levels)
- [6. Sharing the air: medium access, QoS and aggregation](#6-sharing-the-air-medium-access-qos-and-aggregation)
- [7. 802.11 frames and addressing](#7-80211-frames-and-addressing)
- [8. Joining a network: scan, authenticate, associate, handshake](#8-joining-a-network-scan-authenticate-associate-handshake)
- [9. Security: WPA2, WPA3, Enterprise and known attacks](#9-security-wpa2-wpa3-enterprise-and-known-attacks)
- [10. Power save, roaming and multi-AP networks](#10-power-save-roaming-and-multi-ap-networks)

**Part 2: How Linux implements it**

- [11. The Linux wireless stack](#11-the-linux-wireless-stack)
- [12. Tools: iw, wpa_cli, hostapd_cli and friends](#12-tools-iw-wpa_cli-hostapd_cli-and-friends)
- [13. Station mode: connecting a device](#13-station-mode-connecting-a-device)
- [14. Access point mode and provisioning](#14-access-point-mode-and-provisioning)
- [15. Monitor mode and over-the-air captures](#15-monitor-mode-and-over-the-air-captures)

**Part 3: Embedded products**

- [16. Integrating a Wi-Fi chip on a board](#16-integrating-a-wi-fi-chip-on-a-board)
- [17. Performance, power and tuning](#17-performance-power-and-tuning)

**Debugging and reference**

- [18. Troubleshooting](#18-troubleshooting)
- [19. Not covered here](#19-not-covered-here)
- [References](#references)

---

## 1. The mental model: what 802.11 changes compared to Ethernet

Planned: shared half-duplex medium, collisions avoided instead of detected, acknowledgments, mobility, security inside the link layer. Vocabulary: STA, AP, BSS, BSSID, SSID, ESS, radio vs interface.

---

## 2. Example network

Planned: the gateway AP and a board with an SDIO module, addresses, channels, security mode.

---

## 3. Generations and standards

Planned: Wi-Fi 4 to 7 (and 8), PHY names (HT, VHT, HE, EHT), peak rates, the amendments that matter (d, e, h, i, k, r, v, w).

---

## 4. Bands, channels and regulatory rules

Planned: 2.4, 5 and 6 GHz channel plans, channel widths and bonding, DFS, power limits, 6 GHz rules (PSC, LPI, AFC), regulatory domains.

---

## 5. The radio link: OFDM, MCS, MIMO and signal levels

Planned: OFDM and subcarriers, MCS table, rate formula with a worked example, spatial streams, beamforming, MU-MIMO, OFDMA, dBm, RSSI/SNR, path loss.

---

## 6. Sharing the air: medium access, QoS and aggregation

Planned: CSMA/CA, backoff, ACKs and retries, hidden nodes and RTS/CTS, WMM access categories, A-MSDU/A-MPDU, rate control, why throughput is far below the PHY rate.

---

## 7. 802.11 frames and addressing

Planned: frame format, frame types and subtypes, the four address fields, information elements, why a station can't be bridged.

---

## 8. Joining a network: scan, authenticate, associate, handshake

Planned: active and passive scanning, beacons, authentication and association, the 4-way handshake, disconnection and reason codes.

---

## 9. Security: WPA2, WPA3, Enterprise and known attacks

Planned: key hierarchy, PSK and SAE, PMF, OWE, 802.1X/EAP, AKM and cipher suites, transition modes, attacks and what stops them.

---

## 10. Power save, roaming and multi-AP networks

Planned: power save mechanisms (TIM/DTIM, U-APSD, TWT), roaming (802.11k/v/r), mesh, Wi-Fi Direct, multi-link operation.

---

## 11. The Linux wireless stack

Planned: nl80211, cfg80211, mac80211, FullMAC vs SoftMAC, firmware, regulatory database, interface types, driver families.

---

## 12. Tools: iw, wpa_cli, hostapd_cli and friends

Planned: daily commands, reading `iw` output, the `iwconfig` to `iw` mapping, rfkill, debugfs.

---

## 13. Station mode: connecting a device

Planned: wpa_supplicant configurations (WPA2, WPA3, Enterprise, hidden, open/OWE), running it under systemd or BusyBox, events, iwd and NetworkManager.

---

## 14. Access point mode and provisioning

Planned: what to add to the gateway note's hostapd config (Wi-Fi 6/6E, roaming, ACS, DFS), AP+STA concurrency, onboarding a headless device.

---

## 15. Monitor mode and over-the-air captures

Planned: monitor interfaces, radiotap, Wireshark filters, decrypting a capture, what a sniffer can't see.

---

## 16. Integrating a Wi-Fi chip on a board

Planned: buses (SDIO, PCIe, USB, SPI), device tree, power sequencing, firmware and NVRAM files, kernel and build-system options, antennas, coexistence, certification.

---

## 17. Performance, power and tuning

Planned: measuring with iperf3, reading `station dump` and `survey dump`, bus and CPU limits, power save vs latency, WoWLAN.

---

## 18. Troubleshooting

Planned: layer-by-layer method, symptom table, reason and status codes, driver and firmware errors.

---

## 19. Not covered here

Planned: topics for later notes.

---

## References

Planned: standards, Linux and hostap documentation, books.
