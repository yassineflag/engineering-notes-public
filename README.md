# Engineering Notes

Public notes I keep while building things: embedded Linux, networking, and whatever I'm digging into at the time.
Every note aims to be practical, so you'll find configs and commands you can reuse rather than theory.

## Topics

| Folder | What's inside |
|---|---|
| [linux-general-usage/](linux-general-usage/) | Day-to-day Linux: shell, files, users, packages, services |
| [embedded-linux/](embedded-linux/) | [Fundamentals](embedded-linux/fundamentals/) (toolchain, bootloader, kernel, rootfs) and [build systems](embedded-linux/linux-build-systems/) (Yocto, Buildroot) |
| [linux-kernel-device-driver-development/](linux-kernel-device-driver-development/) | Kernel modules, character/platform drivers, device tree |
| [networking/](networking/) | Routing, firewalls, Wi-Fi, home and lab network setups |
| [wireless-protocols-and-connectivity/](wireless-protocols-and-connectivity/) | [Wi-Fi](wireless-protocols-and-connectivity/wifi/), [Bluetooth](wireless-protocols-and-connectivity/bluetooth/), [Cellular](wireless-protocols-and-connectivity/cellular/), [NTN](wireless-protocols-and-connectivity/ntn/) (satellite / non-terrestrial networks) |
| [linux-systems-programming/](linux-systems-programming/) | System calls, processes, threads, IPC, file and socket I/O |
| [software/](software/) | Language notes: [C](software/c/), [Bash](software/bash/) |
| [hardware/](hardware/) | Electronics, boards, buses and peripherals: [analog](hardware/analog/), [digital](hardware/digital/), [signal processing](hardware/signal-processing/), [baseband](hardware/baseband/) (modulation, IQ, coding), [RF](hardware/rf/), [SDR](hardware/sdr/) |

## Latest notes

- [Software-Defined Radio: HackRF One and GNU Radio](hardware/sdr/sdr_v1_2026-11-12.md)
- [Root Filesystems for Embedded Linux](embedded-linux/fundamentals/rootfs/rootfs_v1_2026-11-11.md)
- [The Linux Kernel for Embedded Linux](embedded-linux/fundamentals/kernel/kernel_v1_2026-11-10.md)
- [Bootloaders for Embedded Linux: U-Boot](embedded-linux/fundamentals/bootloader/bootloader_v2_2026-11-09.md)
- [Cross-Compilation and Toolchains for Embedded Linux](embedded-linux/fundamentals/cross-compilation-toolchain/cross-compilation-toolchain_v2_2026-11-08.md)
- [Cellular Protocols: LTE, 5G NR and Cellular IoT](wireless-protocols-and-connectivity/cellular/cellular_v1_2026-11-02.md)
- [Wi-Fi for Embedded Linux Engineers](wireless-protocols-and-connectivity/wifi/wifi_v1_2026-11-01.md)
- [Networking Fundamentals for Embedded Linux Engineers](networking/fundamentals/fundamentals_v1_2026-10-29.md)
- [Networking on OpenWrt](networking/openwrt/openwrt_v1_2026-10-29.md)
- [Building a Home Gateway on Linux](networking/gateway/gateway_v1_2026-10-26.md)

## Conventions

- One topic per folder, one note per Markdown file.
- Commands assume a Debian/Ubuntu-like system unless a note says otherwise.
- Example addresses, interface names and passwords are placeholders. Adapt them to your setup.
