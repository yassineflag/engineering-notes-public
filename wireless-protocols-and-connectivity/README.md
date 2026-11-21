# Wireless Protocols and Connectivity

Notes on the radio links an embedded device uses to reach a network: how each protocol works, how Linux drives it, and how to debug it.

## Notes

### Wi-Fi

- [Wi-Fi for Embedded Linux Engineers](wifi/wifi_v1_2026-11-01.md): bands, channels and regulatory rules (EU/US, 6 GHz, vehicles), OFDM/OFDMA/MCS/MIMO, medium access, aggregation and spatial reuse, frames and addressing, joining, WPA2/WPA3/Enterprise security, power save and roaming, Wi-Fi 7 MLO, mesh, P2P, TDLS and Passpoint; the Linux stack (cfg80211, mac80211, drivers, firmware, offloads), operating modes and interface combinations, station/AP/monitor setups, board integration (buses, device tree, firmware, coexistence, production configuration), performance, power and systematic troubleshooting.

### Bluetooth

- [Bluetooth for Embedded Linux Engineers](bluetooth/bluetooth_v1_2026-11-21.md): Classic (BR/EDR) and Low Energy, versions and the radio (channels, PHYs, power, range), LE advertising and scanning, connection parameters and throughput, addresses and privacy, inquiry, paging and Classic links, HCI and L2CAP with their error codes, ATT and GATT, pairing, bonding and known attacks, the profiles (audio, HID, serial, LE Audio and Auracast, mesh, direction finding, Channel Sounding); the Linux stack (kernel, mgmt, bluetoothd, D-Bus), bluetoothctl, btmgmt and btmon, the board as a central (Bleak) and as a peripheral (a provisioning GATT server), PipeWire audio and RFCOMM; board integration (UART HCI, serdev device tree, firmware, the address), Wi-Fi coexistence, power, production hardening, RF tests and qualification, and troubleshooting by symptom.

### Cellular

- [Cellular Protocols: LTE, 5G NR and Cellular IoT](cellular/cellular_v1_2026-11-02.md): generations, releases and device capabilities, EPS and 5GS architecture, protocol stacks and state machines, NSA vs SA, bands and channel numbers, the LTE and NR physical layer, channels, cell search, system information and random access, scheduling, power control, HARQ and layer 2, carrier aggregation and dual connectivity, RSRP/RSRQ/SINR and link budgets, identities and the SIM (eSIM included), attach and registration with UE capabilities and reject causes, APNs, bearers, PDU sessions and the LTE and 5G QoS models, security and known attacks, RRC states including RRC_INACTIVE, paging, reselection, handover and 4G-5G interworking, DRX, eDRX and PSM, LTE-M, NB-IoT and RedCap, IMS voice and EPS fallback, SMS, positioning, slicing, URLLC and NTN, reading signaling in modem reports and logs, and troubleshooting by layer.

### NTN

- [Non-terrestrial networks](ntn/) (satellite)
