# Root Filesystems for Embedded Linux

The root filesystem is what the kernel mounts as `/` and runs `init` from: the programs, libraries, device handling, startup scripts and configuration of an embedded Linux system. This note covers what a root filesystem must contain, and how to build one by hand around BusyBox and the toolchain's C library. It explains `/dev`, `init`, logins and configuration, then how to turn the tree into an image and get it onto the board. It ends with the differences between a development root filesystem and a production one. The board is the BeagleBone Black used in the [bootloader](../bootloader/bootloader.md) and [kernel](../kernel/kernel.md) notes, so this note starts where section 13 of the kernel note stops: the kernel has mounted the root and runs `/sbin/init`.

The root filesystem is built by hand here, file by file. That's what [Buildroot](../../linux-build-systems/buildroot/) and [Yocto](../../linux-build-systems/yocto/) automate, and knowing every file makes their output readable (section 13). The cross-compiler comes from [Cross-Compilation and Toolchains](../cross-compilation-toolchain/cross-compilation-toolchain.md); the kernel, the device tree and the modules come from the kernel note.

**Assumptions:**
- The board is a BeagleBone Black (TI AM3358, one Cortex-A8, 512 MiB DDR3L, 4 GB eMMC, microSD slot), booted by mainline U-Boot from the SD card. The serial console is on J1 at 115200 8N1 and appears as `ttyS0` in Linux.
- The kernel is the Linux 7.1-rc1 `omap2plus_defconfig` build of the kernel note. Its release string is `7.1.0-rc1-g66edb901bf87`, and `zImage` and `am335x-boneblack.dtb` are on SD card partition 1.
- The root filesystem goes on SD card partition 2 (ext4). During development it can also come over NFS, and section 10 also builds it as an initramfs.
- The cross-compiler is the Arm GNU Toolchain 15.2.Rel1, prefix `arm-none-linux-gnueabihf-`, with glibc. The basic commands come from BusyBox 1.37.0.
- The build machine is x86_64 Debian or Ubuntu. Network addresses (board `192.168.0.50`, build machine `192.168.0.10`) and device names (`/dev/sdX`) are placeholders, the same as in the bootloader note.

**Conventions:**
- `$` is a shell on the build machine, `#` a root shell on the board, `=>` the U-Boot prompt.
- **Expected** marks console output reconstructed from the programs' messages, not captured. Values such as sizes, addresses and timestamps in it are placeholders.
- **Typical** marks sizes and figures given as orders of magnitude, not measured for this note.
- Every command is generic: it follows the documentation of the tool it calls and was not run for this note.

**Verification status:** nothing in this note was run. BusyBox and the other packages weren't built, no image was made, nothing was booted, and no emulator was used. The commands follow the documentation of BusyBox, e2fsprogs, squashfs-tools, mtd-utils, cpio, mmdebstrap, genimage and cryptsetup, and the kernel's own documentation (`Documentation/filesystems/`, `Documentation/admin-guide/`). Kernel behavior is the one described in the kernel note, which checked it in the 7.1-rc1 source. Versions were checked online in November 2026: BusyBox 1.37.0 is the current stable release, and 1.38.0 (May 2026) is still labelled "unstable" on busybox.net. Script and configuration files are written for this note and untested; read them before using them.

**How to read it:**
- **To learn:** Parts 1 to 3 in order. They go from an empty directory to a board that boots to a login prompt.
- **For a quick development root filesystem:** sections 4, 5, 7 and 11 (NFS root).
- **To choose between building by hand, a distribution and a build system:** sections 12 and 13.
- **For a product:** Part 5.
- **When the board doesn't reach a prompt:** section 19, then the symptom tables of section 21.

## Table of contents

**Part 1: What a root filesystem is**

- [1. The mental model: what the kernel needs and what init does](#1-the-mental-model-what-the-kernel-needs-and-what-init-does)
- [2. The example board, root filesystem and files](#2-the-example-board-root-filesystem-and-files)
- [3. The directory layout: what goes where](#3-the-directory-layout-what-goes-where)

**Part 2: Building one by hand**

- [4. BusyBox: the basic commands in one binary](#4-busybox-the-basic-commands-in-one-binary)
- [5. The C library and other shared libraries](#5-the-c-library-and-other-shared-libraries)
- [6. /dev: devtmpfs, mdev and udev](#6-dev-devtmpfs-mdev-and-udev)
- [7. init: inittab and the startup scripts](#7-init-inittab-and-the-startup-scripts)
- [8. Users, logins and file ownership](#8-users-logins-and-file-ownership)
- [9. Configuration: mounts, hostname, network, time and logs](#9-configuration-mounts-hostname-network-time-and-logs)

**Part 3: Putting it on the board**

- [10. Filesystem images: ext4, squashfs, UBIFS and initramfs](#10-filesystem-images-ext4-squashfs-ubifs-and-initramfs)
- [11. Installing: SD card, eMMC and NFS root](#11-installing-sd-card-emmc-and-nfs-root)

**Part 4: Beyond the minimal system**

- [12. Init systems: BusyBox init, SysV init, systemd and others](#12-init-systems-busybox-init-sysv-init-systemd-and-others)
- [13. Distributions and build systems](#13-distributions-and-build-systems)
- [14. Adding software](#14-adding-software)

**Part 5: Production and debugging**

- [15. Read-only root and persistent data](#15-read-only-root-and-persistent-data)
- [16. Updates](#16-updates)
- [17. Security](#17-security)
- [18. Size](#18-size)
- [19. When init doesn't start or a program doesn't run](#19-when-init-doesnt-start-or-a-program-doesnt-run)
- [20. Other architectures: arm64 and the Raspberry Pi 4](#20-other-architectures-arm64-and-the-raspberry-pi-4)
- [21. Troubleshooting by symptom](#21-troubleshooting-by-symptom)
- [22. Not covered here](#22-not-covered-here)
- [References](#references)

## 1. The mental model: what the kernel needs and what init does

*Planned: What the kernel needs from the root filesystem, the jobs of PID 1, the three ways to get a root filesystem.*

## 2. The example board, root filesystem and files

*Planned: The example board, kernel, toolchain, BusyBox version, directories and variables.*

## 3. The directory layout: what goes where

*Planned: The FHS directories and what the kernel itself depends on, the merged /usr question, creating the skeleton.*

## 4. BusyBox: the basic commands in one binary

*Planned: What BusyBox is, downloading and configuring it, the tc build failure, building and installing, static or dynamic.*

## 5. The C library and other shared libraries

*Planned: How dynamic linking finds libraries, finding and copying the right ones, glibc details, choosing a C library.*

## 6. /dev: devtmpfs, mdev and udev

*Planned: Device nodes, the four ways to manage /dev, mdev configuration, module autoloading, firmware, /dev/pts and /dev/shm.*

## 7. init: inittab and the startup scripts

*Planned: BusyBox init, its default behavior, inittab, the rcS and rcK scripts, service scripts, console output.*

## 8. Users, logins and file ownership

*Planned: passwd, group and shadow, how a login works, passwords, setuid BusyBox, file ownership on the board and fakeroot.*

## 9. Configuration: mounts, hostname, network, time and logs

*Planned: fstab, mtab, hostname and hosts, profile, network with ifupdown and udhcpc, DNS, time without a battery, logs, modules at boot.*

## 10. Filesystem images: ext4, squashfs, UBIFS and initramfs

*Planned: The filesystem formats and how to choose, building ext4, squashfs, UBIFS and initramfs images.*

## 11. Installing: SD card, eMMC and NFS root

*Planned: Copying to the SD card, kernel modules, root= and PARTUUID, whole-card images with genimage, the eMMC, NFS root.*

## 12. Init systems: BusyBox init, SysV init, systemd and others

*Planned: BusyBox init, SysV init, systemd, OpenRC, runit and s6 compared; choosing; a systemd unit for an application.*

## 13. Distributions and build systems

*Planned: Debian with mmdebstrap, Buildroot and Yocto, and how each piece of this note maps to their options.*

## 14. Adding software

*Planned: Cross-compiling programs and libraries into the tree: prefix and DESTDIR, staging, pkg-config, libtool, stripping; Dropbear as an example.*

## 15. Read-only root and persistent data

*Planned: Why a read-only root, what writes to the root, a data partition, an overlay on /etc, a whole-root overlay from an initramfs.*

## 16. Updates

*Planned: Package updates against image updates, A/B slots and rollback, SWUpdate, RAUC, Mender, OSTree, keeping kernel and modules together.*

## 17. Security

*Planned: Logins and passwords, SSH host keys, services, users and capabilities, mount options, dm-verity, licenses and CVEs.*

## 18. Size

*Planned: Where the space goes, keeping only the needed modules, BusyBox, the C library, compression, measuring.*

## 19. When init doesn't start or a program doesn't run

*Planned: The init failure messages, checking binaries on the build machine, init=/bin/sh, startup scripts, program errors, tools on the board.*

## 20. Other architectures: arm64 and the Raspberry Pi 4

*Planned: What changes for a 64-bit board: loader, lib64, console names, the Pi's boot partition, firmware, Debian arm64.*

## 21. Troubleshooting by symptom

*Planned: Symptom tables: building, first boot, login and shell, devices, network, programs, filesystems.*

## 22. Not covered here

*Planned: What this note leaves to other notes.*

## References

*Planned: Documentation, tools, standards and books.*
