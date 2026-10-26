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
| [hardware/](hardware/) | Electronics, boards, buses and peripherals. [Communications](hardware/communications/): [analog](hardware/communications/analog/), [digital](hardware/communications/digital/), [signal processing](hardware/communications/signal-processing/), [RF](hardware/communications/rf/), [SDR](hardware/communications/sdr/) (IQ, modulation) |

## Latest notes

- [Building a Home Gateway on Linux](networking/gateway/gateway.md)

## Conventions

- One topic per folder, one note per Markdown file.
- Commands assume a Debian/Ubuntu-like system unless a note says otherwise.
- Example addresses, interface names and passwords are placeholders. Adapt them to your setup.
