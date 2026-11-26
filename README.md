# Engineering Notes

Public notes I keep while building things: embedded Linux, networking, and whatever I'm digging into at the time.
Every note aims to be practical, so you'll find configs and commands you can reuse rather than theory.

## Topics

| Folder | What's inside |
|---|---|
| [linux-general-usage/](os-and-firmware/linux-general-usage/) | Day-to-day Linux: shell, files, users, packages, services |
| [embedded-linux/](os-and-firmware/embedded-linux/) | [Fundamentals](os-and-firmware/embedded-linux/fundamentals/) (toolchain, bootloader, kernel, rootfs) and [build systems](os-and-firmware/embedded-linux/linux-build-systems/) (Yocto, Buildroot) |
| [linux-kernel-device-driver-development/](os-and-firmware/embedded-linux/linux-kernel-device-driver-development/) | Kernel modules, character devices, platform, I2C and SPI drivers, the device tree, debugging |
| [networking/](os-and-firmware/networking/) | [Networking basics](os-and-firmware/networking/networking-basics/) (OSI, IP, DHCP, DNS, architectures), [Linux networking](os-and-firmware/networking/linux-networking-fundamentals/), routing, firewalls, home and lab network setups |
| [wireless-protocols-and-connectivity/](wireless-protocols-and-connectivity/) | [Wi-Fi](wireless-protocols-and-connectivity/wifi/), [Bluetooth](wireless-protocols-and-connectivity/bluetooth/), [Cellular](wireless-protocols-and-connectivity/cellular/), [LoRa](wireless-protocols-and-connectivity/lora/), [SatCom and NTN](wireless-protocols-and-connectivity/satcom-ntn/) (satellite communications, non-terrestrial networks), [UWB](wireless-protocols-and-connectivity/uwb/) (ultra-wideband ranging and positioning) |
| [linux-systems-programming/](os-and-firmware/linux-systems-programming/) | System calls, processes, threads, IPC, file and socket I/O |
| [software/](software/) | Language notes: [C](software/c/), [Bash](software/bash/) |
| [hardware/](hardware/) | Electronics, boards, buses and peripherals: [analog](hardware/analog/), [digital](hardware/digital/), [signal processing](hardware/signal-processing/), [baseband](hardware/baseband/) (modulation, IQ, coding), [RF](hardware/rf/), [SDR](hardware/sdr/) |

## Latest notes

- [Ultra-Wideband (UWB) for Embedded Linux Engineers](wireless-protocols-and-connectivity/uwb/uwb_v1_2026-11-26.md)
- [Bash: A Quick Reference](software/bash/bash_v1_2026-11-25.md)
- [Satellite Communications and NTN for Embedded Linux Engineers](wireless-protocols-and-connectivity/satcom-ntn/satcom-ntn_v1_2026-11-24.md)
- [LoRa and LoRaWAN for Embedded Linux Engineers](wireless-protocols-and-connectivity/lora/lora_v1_2026-11-22.md)
- [Bluetooth for Embedded Linux Engineers](wireless-protocols-and-connectivity/bluetooth/bluetooth_v1_2026-11-21.md)
- [Networking Basics: How Networks Work](os-and-firmware/networking/networking-basics/networking-basics_v1_2026-11-20.md)
- [Linux Kernel Device Drivers for Embedded Linux](os-and-firmware/embedded-linux/linux-kernel-device-driver-development/device-drivers_v1_2026-11-19.md)
- [Everyday Linux: The Shell, Files, Users, Packages and Services](os-and-firmware/linux-general-usage/everyday-linux/everyday-linux_v1_2026-11-16.md)
- [Buildroot for Embedded Linux](os-and-firmware/embedded-linux/linux-build-systems/buildroot/buildroot_v1_2026-11-15.md)
- [The Yocto Project for Embedded Linux](os-and-firmware/embedded-linux/linux-build-systems/yocto/yocto_v1_2026-11-14.md)
- [Software-Defined Radio: HackRF One and GNU Radio](hardware/sdr/sdr_v1_2026-11-12.md)
- [Root Filesystems for Embedded Linux](os-and-firmware/embedded-linux/fundamentals/04-rootfs/rootfs_v1_2026-11-11.md)
- [The Linux Kernel for Embedded Linux](os-and-firmware/embedded-linux/fundamentals/03-kernel/kernel_v1_2026-11-10.md)
- [Bootloaders for Embedded Linux: U-Boot](os-and-firmware/embedded-linux/fundamentals/02-bootloader/bootloader_v2_2026-11-09.md)
- [Cross-Compilation and Toolchains for Embedded Linux](os-and-firmware/embedded-linux/fundamentals/01-cross-compilation-toolchain/cross-compilation-toolchain_v2_2026-11-08.md)
- [Cellular Protocols: LTE, 5G NR and Cellular IoT](wireless-protocols-and-connectivity/cellular/cellular_v1_2026-11-02.md)
- [Wi-Fi for Embedded Linux Engineers](wireless-protocols-and-connectivity/wifi/wifi_v1_2026-11-01.md)
- [Linux Networking Fundamentals for Embedded Engineers](os-and-firmware/networking/linux-networking-fundamentals/linux-networking-fundamentals_v1_2026-10-29.md)
- [Networking on OpenWrt](os-and-firmware/networking/openwrt/openwrt_v1_2026-10-29.md)
- [Building a Home Gateway on Linux](os-and-firmware/networking/gateway/gateway_v1_2026-10-26.md)

## Conventions

- One topic per folder, one note per Markdown file.
- Commands assume a Debian/Ubuntu-like system unless a note says otherwise.
- Example addresses, interface names and passwords are placeholders. Adapt them to your setup.
