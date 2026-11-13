# Wireless Protocols and Connectivity

Notes on the radio links an embedded device uses to reach a network: how each protocol works, how Linux drives it, and how to debug it.

## Notes

### Wi-Fi

- [Wi-Fi for Embedded Linux Engineers](wifi/wifi_v1_2026-11-01.md): bands, channels and regulatory rules (EU/US, 6 GHz, vehicles), OFDM/OFDMA/MCS/MIMO, medium access, aggregation and spatial reuse, frames and addressing, joining, WPA2/WPA3/Enterprise security, power save and roaming, Wi-Fi 7 MLO, mesh, P2P, TDLS and Passpoint; the Linux stack (cfg80211, mac80211, drivers, firmware, offloads), operating modes and interface combinations, station/AP/monitor setups, board integration (buses, device tree, firmware, coexistence, production configuration), performance, power and systematic troubleshooting.

### Bluetooth

- [Bluetooth](bluetooth/)

### Cellular

- [Cellular Protocols: LTE, 5G NR and Cellular IoT](cellular/cellular_v1_2026-11-02.md): generations, releases and device capabilities, EPS and 5GS architecture, protocol stacks and state machines, NSA vs SA, bands and channel numbers, the LTE and NR physical layer, channels, cell search, system information and random access, scheduling, power control, HARQ and layer 2, carrier aggregation and dual connectivity, RSRP/RSRQ/SINR and link budgets, identities and the SIM (eSIM included), attach and registration with UE capabilities and reject causes, APNs, bearers, PDU sessions and the LTE and 5G QoS models, security and known attacks, RRC states including RRC_INACTIVE, paging, reselection, handover and 4G-5G interworking, DRX, eDRX and PSM, LTE-M, NB-IoT and RedCap, IMS voice and EPS fallback, SMS, positioning, slicing, URLLC and NTN, reading signaling in modem reports and logs, and troubleshooting by layer.

### NTN

- [Non-terrestrial networks](ntn/) (satellite)
