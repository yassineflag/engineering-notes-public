# Bootloaders for Embedded Linux: U-Boot

A beginner's path from power-on to a running Linux kernel: what a bootloader does, how to build U-Boot, how to use its command line in QEMU, and how to put it on a BeagleBone Black or a Raspberry Pi 4.

**You need:** the PC, toolchains and settings files from the [cross-compilation note](../cross-compilation-toolchain/cross-compilation-toolchain.md) (sections 3 and 4). No board is needed until section 9. For a board, you also need a 3.3 V USB-to-serial cable.

**How to read it:**
- **Parts 1 and 2 are essential.** Read them in order and type the commands.
- **Part 3 is optional.** Each section says when you need it. If something fails, start at [section 10](#10-troubleshooting).

**How commands are labelled:**

| Label | Meaning |
|---|---|
| **Expected (QEMU)** | **Not run for this note.** Written for Ubuntu 22.04 (GCC 11.4, QEMU 6.2) and U-Boot v2026.10. The output is what U-Boot's source code and documentation say you should see, shortened. Dates, addresses, sizes and timings will differ |
| **Expected (board)** | **Not run on real hardware.** The commands are what you should type, and the output is what you should see, based on the board's documentation and U-Boot's source |
| **Illustrative** | Shows the form of a command, file or message. Not meant to be typed as is |

## Table of contents

**Part 1: Concepts (essential)**

- [1. What a bootloader does](#1-what-a-bootloader-does)
- [2. The words you need](#2-the-words-you-need)
- [3. How each board boots](#3-how-each-board-boots)

**Part 2: A working example (essential)**

- [4. Get and build U-Boot](#4-get-and-build-u-boot)
- [5. First boot in QEMU](#5-first-boot-in-qemu)
- [6. The environment](#6-the-environment)
- [7. Boot Linux from U-Boot](#7-boot-linux-from-u-boot)
- [8. Make it boot by itself](#8-make-it-boot-by-itself)
- [9. Run it on a board](#9-run-it-on-a-board)

**Part 3: Going further (optional)**

- [10. Troubleshooting](#10-troubleshooting)
- [11. Memory and load addresses](#11-memory-and-load-addresses)
- [12. Kernel image formats and FIT](#12-kernel-image-formats-and-fit)
- [13. The device tree in the bootloader](#13-the-device-tree-in-the-bootloader)
- [14. Booting over the network](#14-booting-over-the-network)
- [15. The environment from Linux](#15-the-environment-from-linux)
- [16. Safe updates and verified boot](#16-safe-updates-and-verified-boot)
- [17. Debugging U-Boot](#17-debugging-u-boot)
- [18. Where the rest is covered](#18-where-the-rest-is-covered)
- [References](#references)

---

## 1. What a bootloader does

Planned: why a board needs a bootloader, its four steps, and why it matters when something fails.

---

## 2. The words you need

Planned: boot ROM, SPL, MLO, U-Boot proper, firmware, serial console, environment, bootcmd, bootargs, device tree, load address, kernel image, initramfs, TFTP, defconfig.

---

## 3. How each board boots

Planned: the boot chain of the BeagleBone Black, the Raspberry Pi 4 and QEMU, side by side, and what QEMU can and can't prove.

---

## 4. Get and build U-Boot

Planned: install the build tools, clone U-Boot v2026.10, extend the settings files, build for QEMU.

---

## 5. First boot in QEMU

Planned: a script to start QEMU, the options it uses, the first prompt, and the first commands.

---

## 6. The environment

Planned: read, set, run and save variables; quoting; saving to QEMU's flash.

---

## 7. Boot Linux from U-Boot

Planned: load Debian's installer kernel and initramfs over TFTP and start it with bootz or booti.

---

## 8. Make it boot by itself

Planned: bootcmd, a boot script made with mkimage, and standard boot with extlinux.conf.

---

## 9. Run it on a board

Planned: the serial console, then U-Boot on a BeagleBone Black SD card and on a Raspberry Pi 4, booting Linux by hand and automatically.

---

## 10. Troubleshooting

Planned: symptoms, causes and fixes, from a silent serial port to a kernel panic.

---

## 11. Memory and load addresses

Planned: RAM start and load address variables on each board, and how to avoid overlapping loads.

---

## 12. Kernel image formats and FIT

Planned: zImage, Image, uImage and FIT; recognising them; a FIT example.

---

## 13. The device tree in the bootloader

Planned: what U-Boot changes in the device tree, the fdt command, and overlays.

---

## 14. Booting over the network

Planned: TFTP and an NFS root from the PC, with the BeagleBone's netboot variable.

---

## 15. The environment from Linux

Planned: fw_printenv and fw_setenv, fw_env.config, and a redundant environment.

---

## 16. Safe updates and verified boot

Planned: bootcount and A/B updates, signed FIT images, and locking down the command line.

---

## 17. Debugging U-Boot

Planned: debug messages, and GDB with QEMU across U-Boot's relocation.

---

## 18. Where the rest is covered

Planned: links to the kernel, root filesystem and build system notes, and what isn't covered.

---

## References

Planned: U-Boot, Linux, QEMU and board documentation.
