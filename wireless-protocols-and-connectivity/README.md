# Wireless Protocols and Connectivity

Notes on the radio links an embedded device uses to reach a network: how each protocol works, how Linux drives it, and how to debug it.

## Notes

### Wi-Fi

- [Wi-Fi for Embedded Linux Engineers](wifi/wifi.md): bands, channels and regulatory rules (EU/US, 6 GHz, vehicles), OFDM/OFDMA/MCS/MIMO, medium access, aggregation and spatial reuse, frames and addressing, joining, WPA2/WPA3/Enterprise security, power save and roaming, Wi-Fi 7 MLO, mesh, P2P, TDLS and Passpoint; the Linux stack (cfg80211, mac80211, drivers, firmware, offloads), operating modes and interface combinations, station/AP/monitor setups, board integration (buses, device tree, firmware, coexistence, production configuration), performance, power and systematic troubleshooting.

### Bluetooth

- [Bluetooth](bluetooth/)

### Cellular

- [Cellular Protocols: LTE, 5G NR and Cellular IoT](cellular/cellular.md): generations, releases and device categories, bands and channel numbers, the LTE and NR physical layer, cell search and system information, scheduling, HARQ and layer 2, RSRP/RSRQ/SINR and link budgets, EPC and 5GC architecture, NSA vs SA, identities and the SIM (eSIM included), attach and registration with reject causes, APNs, bearers, PDU sessions, QoS and IP configuration, roaming, security and known attacks, RRC states, paging, handover, PSM and eDRX, LTE-M, NB-IoT and RedCap, SMS, VoLTE, positioning, slicing and NTN, observing the protocol with AT commands and modem logs, and troubleshooting by layer.

### NTN

- [Non-terrestrial networks](ntn/) (satellite)
