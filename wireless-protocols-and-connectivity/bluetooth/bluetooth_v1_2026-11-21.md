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
