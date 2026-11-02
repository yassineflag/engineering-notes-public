# Cellular Protocols: LTE, 5G NR and Cellular IoT

How a cellular network works from the radio up to the IP session: spectrum and the physical layer, how the base station schedules every transmission, how a device finds a cell, registers, authenticates and gets an IP address, how it moves and saves power, and what LTE-M, NB-IoT and RedCap change for IoT devices.

This note covers the **protocols**, as 3GPP specifies them and as a device engineer meets them in modem logs and AT command answers. Driving a modem from Linux (ModemManager, QMI, MBIM, drivers, board integration) belongs in a separate note. Until it exists, the networking side is in [Networking Fundamentals for Embedded Linux Engineers](../../networking/fundamentals/fundamentals.md#20-modem-based-devices-the-ap-and-the-modem) (the AP-modem split, data path, sharing the connection) and a hands-on modem bring-up is in [Building a Home Gateway on Linux](../../networking/gateway/gateway.md#method-2-cellular-modem-quectel-ec25).

**Assumptions:**
- One example network (section 2) is used throughout: a device with an LTE/5G module, an operator's LTE and 5G cells, and its core network.
- LTE means 3GPP E-UTRA from Release 8 on; 5G means NR and the 5G core (5GC) from Release 15 on. 2G and 3G appear only where they still matter: fallback, sunsets and security downgrades.
- Spectrum values are given for the EU and the US. They change, and other regions differ.

**Conventions:**
- **Normative** values come from a 3GPP specification (cited by its TS number) or a regulation.
- **Rule of thumb** marks empirical values that depend on the network, the device and the environment.
- **Operator-dependent** or **vendor-dependent** marks behavior that 3GPP leaves to the network operator or to the modem vendor.

**How to read it:**
- **To learn:** Parts 1 to 3 in order.
- **To understand the radio:** Part 2 (sections 5 to 8).
- **To understand registration, sessions and power saving:** Part 3 (sections 9 to 16).
- **To debug:** section 18 first, then the section of the step that fails.

## Table of contents

**Part 1: Foundations**

- [1. The mental model: what cellular changes compared to Wi-Fi](#1-the-mental-model-what-cellular-changes-compared-to-wi-fi)
- [2. Example network](#2-example-network)
- [3. Generations, 3GPP releases and device categories](#3-generations-3gpp-releases-and-device-categories)
- [4. Spectrum: bands, duplexing and channel numbers](#4-spectrum-bands-duplexing-and-channel-numbers)

**Part 2: The radio interface**

- [5. The physical layer: OFDM, numerology, resource grid, MCS and MIMO](#5-the-physical-layer-ofdm-numerology-resource-grid-mcs-and-mimo)
- [6. Physical channels, cell search and system information](#6-physical-channels-cell-search-and-system-information)
- [7. Scheduling, HARQ and layer 2](#7-scheduling-harq-and-layer-2)
- [8. Signal measurements and link budget](#8-signal-measurements-and-link-budget)

**Part 3: The network and its procedures**

- [9. Network architecture: RAN, EPC, 5GC, NSA and SA](#9-network-architecture-ran-epc-5gc-nsa-and-sa)
- [10. Identities and the SIM](#10-identities-and-the-sim)
- [11. Getting service: PLMN selection, RRC connection, attach and registration](#11-getting-service-plmn-selection-rrc-connection-attach-and-registration)
- [12. Data sessions: APNs, bearers, PDU sessions, QoS and IP](#12-data-sessions-apns-bearers-pdu-sessions-qos-and-ip)
- [13. Security: authentication, keys, ciphering and known attacks](#13-security-authentication-keys-ciphering-and-known-attacks)
- [14. Mobility and power saving: RRC states, paging, handover, DRX, PSM](#14-mobility-and-power-saving-rrc-states-paging-handover-drx-psm)
- [15. Cellular IoT: LTE-M, NB-IoT and RedCap](#15-cellular-iot-lte-m-nb-iot-and-redcap)
- [16. Beyond data: SMS, voice, positioning, slicing, NTN](#16-beyond-data-sms-voice-positioning-slicing-ntn)

**Part 4: Observing and debugging**

- [17. Observing the protocol: AT commands, modem logs and captures](#17-observing-the-protocol-at-commands-modem-logs-and-captures)
- [18. Troubleshooting by layer](#18-troubleshooting-by-layer)
- [19. Not covered here](#19-not-covered-here)
- [References](#references)

---

## 1. The mental model: what cellular changes compared to Wi-Fi

Planned: scheduled vs contended medium, licensed spectrum owned by an operator, identity in the SIM, mobility handled by the network, an IP session instead of a LAN, power states driven by the network. Vocabulary: UE, eNB/gNB, cell, PLMN, core, bearer, APN.

---

## 2. Example network

Planned: the device, an LTE cell on band 20, an NR cell on n78, the operator's core, identities, APNs and addresses, reused from the fundamentals note.

---

## 3. Generations, 3GPP releases and device categories

Planned: 2G to 5G, what each release brought, LTE UE categories, LTE-M, NB-IoT, RedCap, 2G/3G sunsets.

---

## 4. Spectrum: bands, duplexing and channel numbers

Planned: FDD, TDD and SDL, band numbers, EU and US bands, FR1 and FR2, EARFCN and NR-ARFCN, carrier aggregation and EN-DC combinations, why a module comes in regional variants.

---

## 5. The physical layer: OFDM, numerology, resource grid, MCS and MIMO

Planned: OFDMA and SC-FDMA, NR numerologies, frames, slots and resource blocks, modulation and coding, peak rate formula, MIMO layers and beamforming, TDD patterns.

---

## 6. Physical channels, cell search and system information

Planned: synchronization signals, PCI, MIB and SIBs, reference signals, control and shared channels, random access, NR SSB beams and BWPs.

---

## 7. Scheduling, HARQ and layer 2

Planned: downlink and uplink scheduling, SR and BSR, grants, CQI and link adaptation, HARQ, MAC, RLC modes, PDCP, SDAP, carrier aggregation and dual connectivity, latency budget.

---

## 8. Signal measurements and link budget

Planned: RSRP, RSRQ, RSSI, SINR, SS-RSRP, CQI, how modems and AT commands report them, thresholds, noise floor, uplink vs downlink budget, coupling loss.

---

## 9. Network architecture: RAN, EPC, 5GC, NSA and SA

Planned: eNB and gNB, MME, S-GW, P-GW, HSS, AMF, SMF, UPF, UDM, control and user plane stacks, GTP tunnels, NSA option 3x vs SA, where the IP packets actually go.

---

## 10. Identities and the SIM

Planned: IMSI/SUPI/SUCI, IMEI, ICCID, MSISDN, GUTI and TMSI, the USIM and its files, PIN and PUK, form factors, eUICC and remote provisioning.

---

## 11. Getting service: PLMN selection, RRC connection, attach and registration

Planned: PLMN and cell selection, random access, RRC connection setup, LTE attach and 5G registration, tracking areas, EMM/5GMM reject causes.

---

## 12. Data sessions: APNs, bearers, PDU sessions, QoS and IP

Planned: APN and DNN, PDN types, default and dedicated bearers, QCI and 5QI, PDU sessions and QoS flows, IPv4, IPv6 and DNS delivery, MTU, roaming, ESM/5GSM causes.

---

## 13. Security: authentication, keys, ciphering and known attacks

Planned: AKA, the key hierarchy, NAS and AS security, algorithms, SUCI, what is and isn't protected, fake base stations, 2G downgrades, practical consequences for devices.

---

## 14. Mobility and power saving: RRC states, paging, handover, DRX, PSM

Planned: RRC idle, inactive and connected, inactivity timers, paging, cell reselection and handover, DRX, eDRX, PSM and their timers, the cost of keepalives behind CGNAT.

---

## 15. Cellular IoT: LTE-M, NB-IoT and RedCap

Planned: what each one removes, coverage enhancement and repetitions, CIoT optimizations, release assistance, availability and roaming, choosing between them.

---

## 16. Beyond data: SMS, voice, positioning, slicing, NTN

Planned: SMS over NAS and IMS, VoLTE and VoNR, emergency calls, cell-based positioning and GNSS assistance, network slicing, satellite access.

---

## 17. Observing the protocol: AT commands, modem logs and captures

Planned: standard AT commands for registration, signal, sessions and power saving, modem diagnostic logs, GSMTAP and Wireshark, test networks.

---

## 18. Troubleshooting by layer

Planned: a method from SIM to IP, symptoms, reject causes and what they mean, a useful bug report.

---

## 19. Not covered here

Planned: topics for separate notes.

---

## References

Planned: 3GPP specifications, GSMA documents, regulation, books.
