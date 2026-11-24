# Wireless Protocols and Connectivity

Notes on the radio links an embedded device uses to reach a network: how each protocol works, how Linux drives it, and how to debug it.

## Notes

### Wi-Fi

- [Wi-Fi for Embedded Linux Engineers](wifi/wifi_v1_2026-11-01.md): bands, channels and regulatory rules (EU/US, 6 GHz, vehicles), OFDM/OFDMA/MCS/MIMO, medium access, aggregation and spatial reuse, frames and addressing, joining, WPA2/WPA3/Enterprise security, power save and roaming, Wi-Fi 7 MLO, mesh, P2P, TDLS and Passpoint; the Linux stack (cfg80211, mac80211, drivers, firmware, offloads), operating modes and interface combinations, station/AP/monitor setups, board integration (buses, device tree, firmware, coexistence, production configuration), performance, power and systematic troubleshooting.

### Bluetooth

- [Bluetooth for Embedded Linux Engineers](bluetooth/bluetooth_v1_2026-11-21.md): Classic (BR/EDR) and Low Energy, versions and the radio (channels, PHYs, power, range), LE advertising and scanning, connection parameters and throughput, addresses and privacy, inquiry, paging and Classic links, HCI and L2CAP with their error codes, ATT and GATT, pairing, bonding and known attacks, the profiles (audio, HID, serial, LE Audio and Auracast, mesh, direction finding, Channel Sounding); the Linux stack (kernel, mgmt, bluetoothd, D-Bus), bluetoothctl, btmgmt and btmon, the board as a central (Bleak) and as a peripheral (a provisioning GATT server), PipeWire audio and RFCOMM; board integration (UART HCI, serdev device tree, firmware, the address), Wi-Fi coexistence, power, production hardening, RF tests and qualification, and troubleshooting by symptom.

### Cellular

- [Cellular Protocols: LTE, 5G NR and Cellular IoT](cellular/cellular_v1_2026-11-02.md): generations, releases and device capabilities, EPS and 5GS architecture, protocol stacks and state machines, NSA vs SA, bands and channel numbers, the LTE and NR physical layer, channels, cell search, system information and random access, scheduling, power control, HARQ and layer 2, carrier aggregation and dual connectivity, RSRP/RSRQ/SINR and link budgets, identities and the SIM (eSIM included), attach and registration with UE capabilities and reject causes, APNs, bearers, PDU sessions and the LTE and 5G QoS models, security and known attacks, RRC states including RRC_INACTIVE, paging, reselection, handover and 4G-5G interworking, DRX, eDRX and PSM, LTE-M, NB-IoT and RedCap, IMS voice and EPS fallback, SMS, positioning, slicing, URLLC and NTN, reading signaling in modem reports and logs, and troubleshooting by layer.

### LoRa

- [LoRa and LoRaWAN for Embedded Linux Engineers](lora/lora_v1_2026-11-22.md): LoRa vs LoRaWAN and the other low-power wide-area options, chirp spread spectrum, spreading factors, time on air, sensitivity and link budgets, EU868 and US915 regional parameters, duty cycle and dwell time, the LoRaWAN architecture and identifiers, Class A, B and C receive windows, frames, ports and MAC commands, OTAA and ABP, keys, counters and known attacks, ADR, confirmed messages and capacity; Linux as a gateway (SX1302 on spidev, the Semtech packet forwarder and UDP protocol, Basic Station), ChirpStack, device firmware, payload design and codecs, AT-command modules on a UART, MQTT uplinks and downlinks; antennas, gateway placement and battery life, production keys, certification and regulation, firmware updates over the air, LoRa P2P, 2.4 GHz, LR-FHSS, satellites, geolocation and relays, and troubleshooting by symptom.

### Satellite communications and NTN

- [Satellite Communications and NTN for Embedded Linux Engineers](satcom-ntn/satcom-ntn_v1_2026-11-24.md): what a device gets from each kind of satellite system, GEO, MEO, LEO and HEO orbits with delays, footprints and passes, bands from UHF to Ka, ITU services and terminal regulation, link budgets (EIRP, G/T, C/N0, rain, scintillation, shadowing), delay, Doppler and visibility, transparent and regenerative payloads, beams, gateways and inter-satellite links, DVB-S2, DVB-S2X and DVB-RCS2, TCP, PEPs, QUIC and CGNAT over satellite; 3GPP NTN from Rel-17 to Rel-19 (NR-NTN and IoT-NTN, bands, timing advance and Doppler pre-compensation, K_offset, HARQ, SIB19/SIB31/SIB32, moving cells, time and location based mobility, discontinuous coverage, satellite RAT types, UE location and cause 78, direct to device) and the operator landscape; on Linux, GNSS with gpsd, PPS and chrony, an Iridium SBD modem with a Python sender and a DirectIP receiver, an NB-IoT NTN module with AT commands, PSM and non-IP data, a router behind a LEO terminal (prefix delegation, WireGuard through CGNAT, TCP tuning, CAKE), pass prediction with Skyfield, link emulation with netem, compact protected payloads and bearer selection; antennas, sky view, GNSS coexistence, power budgets, certification and troubleshooting by symptom.
