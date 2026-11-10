# The Linux Kernel for Embedded Linux

What the kernel does between the bootloader's jump and the first userspace program on an embedded board, and how to choose, configure, build, install, boot, change and debug it. One board is followed from end to end: the BeagleBone Black, the same board as the [bootloader note](../bootloader/bootloader.md), so the kernel picks up exactly where U-Boot stops.

This note starts at the kernel's first instruction and ends when `init` runs. The cross-compiler is in [Cross-Compilation and Toolchains](../cross-compilation-toolchain/cross-compilation-toolchain.md), how U-Boot loads the kernel is in the bootloader note, and what `init` finds on the root filesystem is in the [rootfs note](../rootfs/). [Buildroot](../../linux-build-systems/buildroot/) and [Yocto](../../linux-build-systems/yocto/) run the same kernel build described here; writing drivers is in [Linux kernel and device driver development](../../../linux-kernel-device-driver-development/).

**Assumptions:**
- The board is a BeagleBone Black (TI AM3358, one Cortex-A8, 512 MiB DDR3L, 4 GB eMMC, microSD slot), booted by mainline U-Boot from the SD card, with the serial console on J1 at 115200 8N1.
- The kernel is mainline Linux 7.1-rc1 configured with `omap2plus_defconfig`, built on an x86_64 Debian or Ubuntu machine with the `arm-none-linux-gnueabihf-` cross-compiler (Arm GNU Toolchain 15.2.Rel1). Any `arm-linux-gnueabihf-` compiler works the same way.
- The root filesystem is ext4 on the SD card's second partition, or NFS during development.
- Network addresses and device names (`/dev/sdX`) are placeholders.

**Conventions:**
- `$` is a shell on the build machine, `#` a root shell on the board, `=>` the U-Boot prompt.
- **Real output** marks text pasted from a command run on the build machine for this note (configuration tools and file inspection only).
- **From source** marks behavior read in the Linux 7.1-rc1 source, cited next to it.
- **Expected** marks console output reconstructed from the format strings in the source, not captured from a board. Values such as addresses, sizes and timestamps in it are placeholders.
- Every other command is generic: it follows the kernel's documentation and was not run for this note.

**Verification status:** the configuration commands of section 5 (`omap2plus_defconfig`, `scripts/config`, `merge_config.sh`, `savedefconfig`, `diffconfig`) and the inspection of existing files (`fdtget` on a DTB, `mkimage -l` and `cpio -t` on an initramfs) were run in November 2026 on Ubuntu 22.04, against a copy of a Linux tree at commit `66edb901bf87` (2 May 2026, just after 7.1-rc1). The files inspected come from an earlier build of that tree for a BeagleBone Black: `zImage` 5,813,512 bytes, `am335x-boneblack.dtb` 98,077 bytes, 860 modules. **The kernel was not built for this note, nothing was booted, and no emulator was used.** Build, install and boot procedures are generic, and console messages are quoted from the source.

**How to read it:**
- **To learn:** Parts 1 to 3 in order. They follow the kernel from its source to `init`.
- **To get a kernel on the board:** sections 5, 6, 9 and 10.
- **To work on the kernel every day:** Part 4.
- **When it doesn't boot:** section 24 lists symptoms and points back to the explanation.

## Table of contents

**Part 1: What the kernel does on an embedded board**

- [1. The mental model: what the kernel does on an embedded board](#1-the-mental-model-what-the-kernel-does-on-an-embedded-board)
- [2. The example board, kernel and files](#2-the-example-board-kernel-and-files)
- [3. Which kernel: mainline, stable, LTS or vendor](#3-which-kernel-mainline-stable-lts-or-vendor)

**Part 2: Configuring and building**

- [4. The source tree and Kbuild](#4-the-source-tree-and-kbuild)
- [5. Configuring: defconfig, menuconfig and fragments](#5-configuring-defconfig-menuconfig-and-fragments)
- [6. Building](#6-building)
- [7. What the build produces](#7-what-the-build-produces)
- [8. Version strings and module vermagic](#8-version-strings-and-module-vermagic)

**Part 3: From the bootloader to init**

- [9. Installing the kernel, the DTB and the modules](#9-installing-the-kernel-the-dtb-and-the-modules)
- [10. The kernel command line](#10-the-kernel-command-line)
- [11. The boot, stage by stage, and its log](#11-the-boot-stage-by-stage-and-its-log)
- [12. The device tree from the kernel's side](#12-the-device-tree-from-the-kernels-side)
- [13. Root filesystem, initramfs and init](#13-root-filesystem-initramfs-and-init)

**Part 4: Working on the kernel**

- [14. Modules on the target](#14-modules-on-the-target)
- [15. Out-of-tree modules](#15-out-of-tree-modules)
- [16. Keeping your changes: patches, fragments and a board DTS](#16-keeping-your-changes-patches-fragments-and-a-board-dts)
- [17. Iterating fast](#17-iterating-fast)

**Part 5: Debugging and production**

- [18. When the console stays silent](#18-when-the-console-stays-silent)
- [19. Reading an oops and a panic](#19-reading-an-oops-and-a-panic)
- [20. Looking inside a running kernel](#20-looking-inside-a-running-kernel)
- [21. Size and boot time](#21-size-and-boot-time)
- [22. Maintenance and security](#22-maintenance-and-security)
- [23. Other architectures: arm64 and the Raspberry Pi 4](#23-other-architectures-arm64-and-the-raspberry-pi-4)
- [24. Troubleshooting by symptom](#24-troubleshooting-by-symptom)
- [25. Not covered here](#25-not-covered-here)
- [References](#references)

## 1. The mental model: what the kernel does on an embedded board

*Planned: the kernel's inputs (image, DTB, command line, initramfs) and who provides them, what it must find on its own, the boot stages from the decompressor to init, and the three things that must agree: configuration, device tree, root filesystem.*

## 2. The example board, kernel and files

*Planned: the BeagleBone Black, the 7.1-rc1 tree, omap2plus_defconfig, the toolchain, ARCH and CROSS_COMPILE, and the files the board needs.*

## 3. Which kernel: mainline, stable, LTS or vendor

*Planned: where kernels come from, version numbers, and how to choose for a product.*

## 4. The source tree and Kbuild

*Planned: the directories a board touches, Kconfig and Makefile pairs, tristate, depends on and select.*

## 5. Configuring: defconfig, menuconfig and fragments

*Planned: defconfig and savedefconfig, menuconfig, scripts/config and its traps, diffconfig, config fragments with merge_config.sh, /proc/config.gz.*

## 6. Building

*Planned: host packages, out-of-tree builds with O=, the make targets, parallel builds, rebuilding one directory.*

## 7. What the build produces

*Planned: vmlinux, Image, zImage and uImage, System.map, DTBs, modules and their index files, kernel compression.*

## 8. Version strings and module vermagic

*Planned: kernelversion and kernelrelease, LOCALVERSION and the Git suffix, vermagic and why a module refuses to load.*

## 9. Installing the kernel, the DTB and the modules

*Planned: the boot partition, modules_install with INSTALL_MOD_PATH, stripping, dtbs_install, headers_install.*

## 10. The kernel command line

*Planned: bootargs and /chosen, CONFIG_CMDLINE and the omap2plus ttyO2 trap, the parameters that matter, unknown parameters passed to init.*

## 11. The boot, stage by stage, and its log

*Planned: the silent decompressor, start_kernel, the console, driver probing, mounting root and starting init, with the expected messages.*

## 12. The device tree from the kernel's side

*Planned: machine matching, driver matching by compatible, status and aliases, /proc/device-tree, a board DTS that includes the BeagleBone Black's.*

## 13. Root filesystem, initramfs and init

*Planned: root=, rootwait, NFS, the built-in initramfs, rdinit and init, the init search order, devtmpfs and /dev/console.*

## 14. Modules on the target

*Planned: modprobe and insmod, modules.dep, autoloading by modalias, parameters, built-in versus module.*

## 15. Out-of-tree modules

*Planned: a minimal Kbuild module, building against the kernel's build directory, Module.symvers, installing into updates/.*

## 16. Keeping your changes: patches, fragments and a board DTS

*Planned: patches with Git, keeping configuration and device tree changes in your own repository, moving to a new release.*

## 17. Iterating fast

*Planned: incremental builds, building one module, TFTP and NFS boot, copying modules to the board.*

## 18. When the console stays silent

*Planned: earlycon for the AM335x, keep_bootcon, DEBUG_LL, reading the log buffer after a reset.*

## 19. Reading an oops and a panic

*Planned: the parts of an oops, decode_stacktrace.sh and faddr2line, panic=, pstore and ramoops.*

## 20. Looking inside a running kernel

*Planned: dmesg and log levels, sysfs and bound drivers, deferred probe, debugfs, dynamic debug, ftrace, magic SysRq.*

## 21. Size and boot time

*Planned: where the size goes, trimming the configuration, compression, initcall_debug and asynchronous probing.*

## 22. Maintenance and security

*Planned: following stable updates, CVEs, hardening options, module signing.*

## 23. Other architectures: arm64 and the Raspberry Pi 4

*Planned: what changes on 64-bit Arm, Image and booti, the Raspberry Pi kernel fork and its firmware.*

## 24. Troubleshooting by symptom

*Planned: a table from the last console line to the cause and the section.*

## 25. Not covered here

*Planned: what other notes cover.*

## References

*Planned: kernel documentation, source files and manuals used.*
