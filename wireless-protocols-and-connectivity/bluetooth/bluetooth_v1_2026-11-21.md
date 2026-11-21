# Bluetooth for Embedded Linux Engineers

How Bluetooth works from the radio up to GATT and the profiles, how Linux implements it (the kernel's HCI core and drivers, BlueZ's `bluetoothd`, D-Bus), and what it takes to put a Bluetooth controller in a product and debug it when a device isn't found, won't pair, disconnects or is slow.

This note covers both halves of Bluetooth: **Classic** (BR/EDR: audio, serial links, HID) and **Low Energy** (LE: sensors, provisioning, beacons, LE Audio). The board and network are the ones of [Wi-Fi for Embedded Linux Engineers](../wifi/wifi_v1_2026-11-01.md): the board's Wi-Fi and Bluetooth share one combo module. Wi-Fi itself, and what happens once the device has an IP address, are in the Wi-Fi note and in [Linux Networking Fundamentals for Embedded Engineers](../../os-and-firmware/networking/linux-networking-fundamentals/linux-networking-fundamentals_v1_2026-10-29.md). This note links to them instead of repeating them.

**Assumptions:**
- One example setup (section 2) is used throughout: the board with its combo module, a BLE temperature sensor, a phone, a headset and the dev laptop with a USB Bluetooth adapter.
- Linux 6.x with BlueZ 5.66 or later (Debian 12 ships 5.66, Debian 13 ships 5.8x). Features that need a newer kernel or BlueZ say so. Tools that BlueZ deprecated (`hciconfig`, `hcitool`, `gatttool`, `hciattach`) are mentioned only to say what replaces them.
- Specification values are from the Bluetooth Core Specification 5.x and 6.x. Versions and what they added are in section 3.
- Addresses, names, passkeys and UUIDs that aren't standard are placeholders.

**Conventions:**
- `$` is a shell on the dev laptop, `#` a root shell on the board.
- **Normative** values come from the Bluetooth Core Specification, a profile specification or the SIG's Assigned Numbers.
- **Rule of thumb** marks empirical values that depend on the environment, the chips and the application.
- **Controller-dependent** or **stack-dependent** marks behavior that the specification leaves open: support varies between chips, firmware, kernel and BlueZ versions, and phone operating systems.
- **Illustrative** marks a file, script or output that shows the shape of a solution and needs adapting.

**Verification status:** nothing in this note was run for it. No source was downloaded, nothing was built, no board or phone was used. Commands, configuration keys and D-Bus interfaces follow the BlueZ documentation and manual pages, protocol values follow the Core Specification and Assigned Numbers, and kernel behavior follows the kernel documentation, as known in November 2026. Command outputs are illustrative and abridged.

**How to read it:**
- **To learn:** Part 1 (sections 1 to 12) in order.
- **To use Bluetooth from Linux:** Part 2 (sections 13 to 17).
- **To integrate a controller in a product:** Part 3 (sections 18 to 20).
- **To debug:** section 21 first, then the section of the step that fails.

## Table of contents

**Part 1: How Bluetooth works**

- [1. The mental model: two radios under one name](#1-the-mental-model-two-radios-under-one-name)
- [2. Example setup](#2-example-setup)
- [3. Versions, specifications and the SIG](#3-versions-specifications-and-the-sig)
- [4. The radio: band, channels, PHYs, power and range](#4-the-radio-band-channels-phys-power-and-range)
- [5. LE advertising and scanning](#5-le-advertising-and-scanning)
- [6. LE connections: parameters, data length and throughput](#6-le-connections-parameters-data-length-and-throughput)
- [7. Addresses and privacy](#7-addresses-and-privacy)
- [8. Classic (BR/EDR): inquiry, paging, piconets and links](#8-classic-bredr-inquiry-paging-piconets-and-links)
- [9. The host stack: HCI and L2CAP](#9-the-host-stack-hci-and-l2cap)
- [10. ATT and GATT](#10-att-and-gatt)
- [11. Pairing, bonding and security](#11-pairing-bonding-and-security)
- [12. Profiles: Classic, LE, LE Audio, mesh and positioning](#12-profiles-classic-le-le-audio-mesh-and-positioning)

**Part 2: Bluetooth on Linux**

- [13. The Linux Bluetooth stack](#13-the-linux-bluetooth-stack)
- [14. Tools: bluetoothctl, btmgmt, btmon and friends](#14-tools-bluetoothctl-btmgmt-btmon-and-friends)
- [15. The board as a central: scan, pair, connect, read a sensor](#15-the-board-as-a-central-scan-pair-connect-read-a-sensor)
- [16. The board as a peripheral: advertising and a GATT server](#16-the-board-as-a-peripheral-advertising-and-a-gatt-server)
- [17. Classic on Linux: audio, serial and networking](#17-classic-on-linux-audio-serial-and-networking)

**Part 3: Embedded products**

- [18. Integrating a Bluetooth controller on a board](#18-integrating-a-bluetooth-controller-on-a-board)
- [19. Coexistence, power and performance](#19-coexistence-power-and-performance)
- [20. Production: configuration, hardening, RF tests and qualification](#20-production-configuration-hardening-rf-tests-and-qualification)

**Debugging and reference**

- [21. Troubleshooting](#21-troubleshooting)
- [22. Not covered here](#22-not-covered-here)
- [References](#references)

---

## 1. The mental model: two radios under one name

"Bluetooth" is two different technologies that share a band, a name, a security model and a host interface:

| | Classic (BR/EDR) | Low Energy (LE) |
|---|---|---|
| Since | 1.0 (1999), EDR in 2.0 | 4.0 (2010) |
| Made for | Continuous streams: audio, serial cables, keyboards and mice | Small, infrequent data; years on a coin cell. LE Audio (section 12) now also carries audio |
| Channels | 79 × 1 MHz, frequency hopping 1600 times per second | 40 × 2 MHz: 3 primary advertising channels, 37 data channels |
| Finding devices | Inquiry: the searcher asks, devices in inquiry scan answer (section 8) | Advertising: devices broadcast, scanners listen (section 5) |
| Connecting | Paging, then the link stays up, often for hours | Connection events at a negotiated interval, the radio sleeps in between (section 6) |
| Data model | Profiles over RFCOMM and L2CAP channels (serial port, A2DP, HFP...) | Mostly GATT: a table of attributes the peer reads, writes and subscribes to (section 10) |
| Typical peak current | Tens of mA while streaming | A few mA for a few ms, a few µA asleep |
| Typical users | Headsets, car kits, game controllers | Sensors, wearables, smart locks, beacons, provisioning a device's Wi-Fi |

A **dual-mode** controller (every phone, laptop and combo module) does both. A **single-mode** LE device (most sensors and nRF/ESP32-class chips used as peripherals) does only LE. A Classic-only device can't talk to an LE-only one: being "Bluetooth" isn't enough, the two need a common transport.

**Controller and host.** Every Bluetooth stack is split in two, with a standard interface between them:

```
 ┌──────────── Host (Linux: kernel + bluetoothd) ─────────────┐
 │ Profiles and apps:  A2DP  HFP  HID  SPP   GATT services     │
 │ GAP (roles, discovery, connection, security policy)         │
 │ RFCOMM  SDP  AVDTP ...        ATT/GATT     SMP              │
 │ L2CAP: channels, segmentation, flow control                 │
 └──────────────────────────────┬──────────────────────────────┘
                                │ HCI: commands, events, data
                                │ (UART, USB, SDIO, or in-chip)
 ┌──────────────────────────────┴──────────────────────────────┐
 │ Controller (the chip's firmware)                             │
 │ Link Manager (BR/EDR, LMP)   Link Layer (LE)                 │
 │ Baseband: hopping, timing, packets, encryption, retries      │
 │ Radio: 2.4 GHz PHY                                           │
 └──────────────────────────────────────────────────────────────┘
```

- The **controller** handles everything time-critical: hopping, timing, acknowledgments and retransmissions, encryption of packets. Its firmware comes from the chip vendor and you don't change it, beyond loading a patch at boot (section 18).
- The **host** decides what to do: whom to connect to, which profile to run, which keys to store. On Linux, the kernel implements HCI, L2CAP, SMP and the socket interfaces; `bluetoothd` implements GAP policy, GATT, the profiles and the D-Bus API (section 13).
- **HCI** (host controller interface) is the standard protocol between them (section 9). Because it's standard, any host can drive any controller, and `btmon` can show everything that crosses it. It's the single most useful debugging point.
- On a microcontroller (nRF52, ESP32), host and controller often run on the same chip and HCI is a function call. Such a chip can also run controller-only firmware and be used by Linux over a UART (section 18).

**Vocabulary:**

| Term | Meaning |
|---|---|
| Adapter, `hci0` | Linux's name for a local controller |
| BD_ADDR | Bluetooth device address, 48 bits, written `00:00:5E:00:53:16`. LE devices often use random addresses (section 7) |
| Central / peripheral | LE connection roles: the central initiates and runs the connection's timing (the phone, the board reading a sensor), the peripheral advertises and accepts (the sensor). Before 5.3 the spec said master/slave; old tools and logs still do |
| Broadcaster / observer | LE roles without connections: advertise only (a beacon), scan only |
| Client / server | GATT roles, independent of the connection role: the server holds the data (usually the peripheral), the client reads it |
| Pairing / bonding | Pairing creates keys for one connection; bonding stores them for later reconnections (section 11) |
| Profile | A specification of how to use the protocols for one use case: A2DP (stereo audio), HFP (calls), HOGP (keyboards over LE)... |
| Service, characteristic | GATT building blocks: a service groups characteristics; a characteristic is a value with properties (read, write, notify) (section 10) |
| UUID | Identifier of a service or characteristic: 16-bit for SIG-defined ones (`0x180F` Battery), 128-bit for your own |
| SIG | Bluetooth Special Interest Group: writes the specifications, runs qualification, owns the trademark (section 3) |

**Bluetooth is a short-range, low-rate link that must be designed for disconnection.** Devices move out of range, phones kill background apps, the link drops on interference. Products that work well reconnect quickly and keep state on both ends; products that work badly assume the link is always there.

---

## 2. Example setup

The board is the one of the [Wi-Fi note](../wifi/wifi_v1_2026-11-01.md#2-example-network): its combo module has Wi-Fi on SDIO and Bluetooth on a UART. The product use cases:

- **Provisioning:** a phone app sends the Wi-Fi credentials to the board over BLE before it has a network ([Wi-Fi note, provisioning](../wifi/wifi_v1_2026-11-01.md#provisioning-a-headless-device)). The board is a **peripheral** and **GATT server** (section 16).
- **Sensor gateway:** the board reads a BLE temperature sensor and forwards the values over Wi-Fi. The board is a **central** and **GATT client** (section 15).
- **Audio:** the dev laptop streams to a headset, to show Classic audio and LE Audio on Linux (section 17).

```mermaid
flowchart LR
    subgraph BOARD["Board"]
        hci0["hci0: combo module, UART HCI<br/>Bluetooth 5.x, dual mode<br/>00:00:5E:00:53:16"]
        wlan0["wlan0: Wi-Fi<br/>192.168.10.21"]
    end
    sensor["BLE sensor<br/>peripheral, LE only<br/>random static address"]
    phone["Phone<br/>provisioning app"]
    headset["Headset<br/>BR/EDR + LE Audio"]
    laptop["Dev laptop<br/>hci1: USB adapter<br/>btmon, Wireshark"]
    hci0 -.-|"LE: board is central"| sensor
    phone -.-|"LE: board is peripheral"| hci0
    laptop -.-|"BR/EDR A2DP, LE Audio"| headset
    wlan0 ---|"Wi-Fi to the gateway"| laptop
```

| Device | Adapter | Address | Role in the examples |
|---|---|---|---|
| Board | `hci0` on `/dev/ttymxc1` (UART with RTS/CTS) | `00:00:5E:00:53:16` (public) | Dual-mode Bluetooth 5.x controller in the Wi-Fi + Bluetooth combo module. Central for the sensor, peripheral for the phone |
| BLE sensor | | `F0:00:00:00:53:01` (random static) | LE-only, advertises as `TempSensor-01`, Environmental Sensing service (`0x181A`) with a Temperature characteristic (`0x2A6E`) and Battery service (`0x180F`) |
| Phone | | Resolvable private addresses, changing (section 7) | Runs the provisioning app |
| Headset | | `00:00:5E:00:53:70` | Dual-mode: A2DP/HFP over BR/EDR, LE Audio when both sides support it |
| Dev laptop | `hci1`: USB adapter (`btusb`) | `00:00:5E:00:53:64` | Runs `btmon`, Wireshark, `bluetoothctl`, PipeWire |

The public addresses use the documentation block `00:00:5E:00:53:xx` (RFC 7042) as placeholders; the sensor's random static address is a placeholder of the right form (two top bits set, section 7). The UUIDs of the provisioning service (section 16) are placeholders too: generate your own with `uuidgen`.

---

## 3. Versions, specifications and the SIG

The **Bluetooth SIG** writes the **Core Specification** (radio, link layer, HCI, L2CAP, ATT/GATT, SMP, GAP), the **profile and service specifications** (A2DP, HFP, Heart Rate, LE Audio...) and the **Assigned Numbers** document (UUIDs, company identifiers, AD types, appearance values, codec IDs). Since 2023 the core version numbers follow a regular release train; the SIG publishes a feature summary with each release.

| Version | Year | What it brought (main points) |
|---|---|---|
| 1.1, 1.2 | 2001, 2003 | First widely deployed Classic; 1.2 added adaptive frequency hopping (AFH) and eSCO |
| 2.0 + EDR | 2004 | Enhanced Data Rate: 2 and 3 Mb/s |
| 2.1 + EDR | 2007 | Secure Simple Pairing (SSP): numeric comparison, passkey, Just Works. Sniff subrating |
| 3.0 + HS | 2009 | Bulk data over a Wi-Fi link (AMP). Hardly used, later removed |
| 4.0 | 2010 | **Low Energy**: advertising, connections, ATT/GATT, SMP. "Bluetooth Smart" |
| 4.1 | 2013 | LE devices in several roles at once, LE credit-based L2CAP channels, coexistence signaling with LTE |
| 4.2 | 2014 | **LE Secure Connections** (ECDH P-256), **Data Length Extension** (251-byte packets), link-layer privacy |
| 5.0 | 2016 | **LE 2M** and **LE Coded** PHYs (speed, range), **extended advertising**, periodic advertising, channel selection algorithm #2, up to +20 dBm |
| 5.1 | 2019 | Direction finding (angle of arrival and departure), GATT caching |
| 5.2 | 2019 | **Isochronous channels** (base of LE Audio), Enhanced ATT (EATT), LE power control |
| 5.3 | 2021 | Connection subrating, channel classification by the peripheral, periodic advertising enhancements. Inclusive terminology (central/peripheral) |
| 5.4 | 2023 | Periodic advertising with responses (PAwR, for electronic shelf labels), encrypted advertising data, advertising coding selection |
| 6.0 | 2024 | **Channel Sounding** (secure distance measurement), decision-based advertising filtering, monitoring advertisers, frame space update |
| 6.1 | 2025 | Resolvable private address updates at randomized times, controller power-saving improvements |
| 6.2 | 2025 | Shorter connection intervals (below 7.5 ms), isochronous data over USB HCI, Channel Sounding hardening |

Newer releases and in-progress work (higher data throughput for LE, operation above 2.4 GHz) are listed on the SIG's specifications page: check it rather than this table for anything after 6.2.

**What a version number means.** Almost everything after 4.0 is **optional**. A "Bluetooth 5.3" controller is a controller qualified against Core 5.3: it may support 2M but not Coded, or no extended advertising at all. The version tells you which features *can* be present; the controller's feature bits tell you which *are* (`btmgmt info`, `bluetoothctl show`, or the HCI LE Read Local Supported Features command in `btmon`, section 14). And a feature works only if the **controller, the host stack and the peer** all support it: a phone that doesn't do LE Coded can't connect to your sensor over Coded, whatever your chip does.

**Qualification.** Selling a product with Bluetooth, or using the name and logo, requires the product to be listed through the SIG's **Qualification Program** (membership, a declaration, a listing fee). Products built on an already qualified module or controller and host stack mostly reference the existing qualifications (section 20). This is separate from radio regulation (FCC, CE/RED), which is required whatever the trademark.

---

## 4. The radio: band, channels, PHYs, power and range

Both halves use the 2400 to 2483.5 MHz ISM band, the same as 2.4 GHz Wi-Fi, Zigbee/Thread and microwave ovens. In this band, a device must cope with interference: Classic by hopping, LE by spreading over channels and retrying.

### Channels

| | BR/EDR | LE |
|---|---|---|
| Channels | 79 × 1 MHz, center 2402 + *k* MHz, *k* = 0..78 | 40 × 2 MHz, center 2402 + 2*k* MHz, *k* = 0..39 |
| Hopping | 1600 hops/s along a pseudo-random sequence derived from the central's address and clock | One channel per connection event (or per isochronous subevent), chosen by a channel selection algorithm |
| Avoiding busy channels | AFH: the central marks bad channels and excludes them from the hop set (at least 20 must remain) | Channel map: the central excludes bad data channels (at least 2 must remain); 5.3 lets the peripheral report bad channels |
| Discovery channels | 32 inquiry/page channels, hopped quickly | 3 primary advertising channels |

The three LE primary advertising channels are placed between the usual 2.4 GHz Wi-Fi channels:

| LE channel index | RF channel | Center (MHz) | Position |
|---|---|---|---|
| 37 | 0 | 2402 | Below Wi-Fi channel 1 (2412) |
| 38 | 12 | 2426 | Between Wi-Fi channels 1 and 6 (2437) |
| 39 | 39 | 2480 | Above Wi-Fi channel 11 (2462) and 13 (2472) |

Data channels 0 to 36 fill the rest. Logs and HCI commands use the **channel index** (0 to 39), not the RF channel number: index 37 is 2402 MHz, at the bottom of the band.

### PHYs and modulation

| PHY | Modulation | Symbol rate | Raw data rate | Notes |
|---|---|---|---|---|
| BR (Basic Rate) | GFSK | 1 Msym/s | 1 Mb/s | Classic baseline |
| EDR 2 Mb/s | π/4-DQPSK | 1 Msym/s | 2 Mb/s | Packet header still in GFSK |
| EDR 3 Mb/s | 8DPSK | 1 Msym/s | 3 Mb/s | Shortest range of the Classic rates |
| LE 1M | GFSK | 1 Msym/s | 1 Mb/s | Mandatory for LE; all legacy advertising |
| LE 2M | GFSK | 2 Msym/s | 2 Mb/s | 5.0, optional. Faster, shorter airtime, slightly less range |
| LE Coded S=2 | GFSK + FEC | 1 Msym/s | 500 kb/s | 5.0, optional. About 2× the range of 1M (rule of thumb) |
| LE Coded S=8 | GFSK + FEC | 1 Msym/s | 125 kb/s | About 4× the range of 1M in open space (rule of thumb), 8× the airtime |

Coded PHY trades airtime (energy, latency, channel occupancy) for range. It is mostly used between devices you control: phone support for Coded PHY is partial and varies by model.

### Transmit power and power classes

| Class | Max power | Typical use |
|---|---|---|
| 1 | 100 mW (+20 dBm) | Laptops and USB adapters with an amplifier, industrial modules |
| 1.5 (LE) | 10 mW (+10 dBm) | Many LE modules |
| 2 | 2.5 mW (+4 dBm) | Phones, headsets, most combo modules |
| 3 | 1 mW (0 dBm) | Small LE devices |

The class is a **maximum**; the controller and regulation can set less. In the EU, ETSI EN 300 328 limits the band to 20 dBm EIRP (10 dBm EIRP per MHz... for wide-band modulations: the certification lab checks the details). The antenna gain counts in EIRP: a +20 dBm chip with a 3 dBm-gain antenna exceeds 20 dBm EIRP. Since 5.2, LE power control lets two connected devices ask each other to lower or raise their power.

### Range and link budget

Receiver sensitivity (rule of thumb, from datasheets): about -95 dBm for LE 1M, -92 dBm for LE 2M, -100 to -105 dBm for LE Coded S=8, -90 dBm for BR and a few dB worse for EDR 3 Mb/s. The specification only requires -70 dBm for LE 1M; real chips are far better.

The **link budget** is TX power plus antenna gains minus sensitivity. With 0 dBm TX, -95 dBm sensitivity and 0 dBi antennas, 95 dB of path loss is allowed. At 2.44 GHz, free-space loss is about 40 dB at 1 m and adds 20 dB per decade of distance (60 dB at 10 m, 80 dB at 100 m). In free space that's several hundred meters; indoors it's typically 10 to 30 m (rule of thumb), because:

- every wall costs 3 to 15 dB, concrete and metal far more;
- the human body absorbs 2.4 GHz: a phone in a back pocket or a wearable on the far side of the body loses 10 to 20 dB;
- small devices have poor antennas (-5 to -10 dBi is common for chip antennas in a plastic case near a battery);
- multipath fading makes the signal drop by 10 to 20 dB at some positions and frequencies: hopping and channel diversity exist because of it.

**RSSI** (received signal strength indication) is reported in advertising reports and per connection. Use it to compare positions, not as a distance meter: the same distance gives values 10 to 20 dB apart depending on orientation, bodies and channel. For distance, Channel Sounding (section 12) or several samples with a calibrated model are needed.

---
