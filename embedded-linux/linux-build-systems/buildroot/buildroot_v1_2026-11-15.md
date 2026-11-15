# Buildroot for Embedded Linux

Buildroot builds a complete embedded Linux system with one configuration file and one `make`: the cross-toolchain, the bootloader, the kernel, the packages, the root filesystem and the card image. This note covers how Buildroot is organized, how to build a first image in QEMU and then for the board, and how to make the system your own in a separate tree that holds your configuration, your board files, a package for your program, kernel and BusyBox fragments and the card layout. It ends with what a product needs: an SDK, reproducible builds, license and vulnerability reports, a read-only root and updates. The board is the BeagleBone Black used in the fundamentals notes.

Each piece that the fundamentals notes build by hand is built here by a Buildroot package: the [toolchain](../../fundamentals/cross-compilation-toolchain/cross-compilation-toolchain_v2_2026-11-08.md), [U-Boot](../../fundamentals/bootloader/bootloader_v2_2026-11-09.md), the [kernel](../../fundamentals/kernel/kernel_v1_2026-11-10.md) and the [root filesystem](../../fundamentals/rootfs/rootfs_v1_2026-11-11.md). Section 13 of the rootfs note maps each hand-built piece to a Buildroot option; this note explains where those options go. The [Yocto note](../yocto/yocto_v1_2026-11-14.md) covers the other build system and compares the two (its section 1).

**Assumptions:**
- The release is Buildroot 2025.02, the long-term support (LTS) release, maintained for three years with point releases (2025.02.x). Buildroot also makes a regular release every three months (2026.02, 2026.05...), each maintained for about three months; section 19 explains the choice.
- The board is a BeagleBone Black (TI AM3358, one Cortex-A8, 512 MiB RAM, 4 GB eMMC, microSD slot), with the serial console on J1 at 115200 8N1, `ttyS0` in Linux.
- The first build runs in QEMU with `qemu_arm_vexpress_defconfig`, an emulated 32-bit Arm board, which needs no hardware. The board build starts from `beaglebone_defconfig`.
- The build machine is x86_64 Debian or Ubuntu, with 4 or more cores, 8 GiB of RAM and 20 GB of free disk space to be comfortable (section 3).
- Network addresses (board `192.168.0.50`) and device names (`/dev/sdX`) are placeholders, the same as in the other notes.

**Conventions:**
- `$` is a shell on the build machine, `#` a root shell on the board.
- **Expected** marks output reconstructed from the tools' messages, not captured. Versions, counts and sizes in it are placeholders.
- **From memory** marks file contents recalled from earlier releases and not re-read for this note. Check them in your copy of Buildroot.
- **Illustrative** marks a file that shows the shape of a solution and needs adapting.
- **Typical** marks sizes and durations given as orders of magnitude, not measured.

**Verification status:** nothing in this note was run. No source was downloaded, nothing was built, no image was booted. The option names, `make` targets and file formats follow the Buildroot manual and source tree as known from the 2024 and 2025 releases; they were not re-read line by line for this note. The release schedule and the LTS policy were checked online in November 2026. Three points could not be confirmed:
- **The contents of `beaglebone_defconfig` and `board/beaglebone/` in 2025.02.** The defconfig has been in Buildroot for many years, but its exact kernel, U-Boot version and boot files change between releases. Section 7 shows how to read them.
- **Whether 2026.08 was released.** Releases up to 2026.05 were confirmed. It doesn't change anything in this note.
- **The exact output of the tools.** Every output block is Expected.

The external tree, the package and the configuration files are written for this note and untested; read them before using them.

**How to read it:**
- **To learn:** Parts 1 to 3 in order. They go from the concepts to your own image on the board.
- **For a first image quickly:** sections 3 to 5.
- **To add your own software:** sections 9 and 11.
- **For a product:** Part 5.
- **When a build fails:** section 17, then the symptom tables of section 20.

## Table of contents

**Part 1: What Buildroot is**

- [1. The mental model: one configuration, one make, one image](#1-the-mental-model-one-configuration-one-make-one-image)
- [2. The vocabulary](#2-the-vocabulary)

**Part 2: A first build**

- [3. The build machine](#3-the-build-machine)
- [4. Getting Buildroot](#4-getting-buildroot)
- [5. Building and running an image in QEMU](#5-building-and-running-an-image-in-qemu)
- [6. The output directory: where everything goes](#6-the-output-directory-where-everything-goes)

**Part 3: Making it yours**

- [7. The BeagleBone Black: beaglebone_defconfig](#7-the-beaglebone-black-beaglebone_defconfig)
- [8. Configuration: menuconfig, defconfigs and the toolchain](#8-configuration-menuconfig-defconfigs-and-the-toolchain)
- [9. Your own tree: br2-external](#9-your-own-tree-br2-external)
- [10. Customizing the root filesystem](#10-customizing-the-root-filesystem)
- [11. Your own package](#11-your-own-package)
- [12. The kernel and the bootloader](#12-the-kernel-and-the-bootloader)
- [13. The card image: genimage](#13-the-card-image-genimage)

**Part 4: Working with it every day**

- [14. Rebuilding: what make tracks and what it doesn't](#14-rebuilding-what-make-tracks-and-what-it-doesnt)
- [15. Working on source code: OVERRIDE_SRCDIR](#15-working-on-source-code-override_srcdir)
- [16. The SDK: building applications outside Buildroot](#16-the-sdk-building-applications-outside-buildroot)
- [17. When a build fails](#17-when-a-build-fails)

**Part 5: Production**

- [18. Reproducible builds, licenses, vulnerabilities and updates](#18-reproducible-builds-licenses-vulnerabilities-and-updates)
- [19. Releases and upgrades](#19-releases-and-upgrades)
- [20. Troubleshooting by symptom](#20-troubleshooting-by-symptom)
- [21. Not covered here](#21-not-covered-here)
- [References](#references)

## 1. The mental model: one configuration, one make, one image

## 2. The vocabulary

## 3. The build machine

## 4. Getting Buildroot

## 5. Building and running an image in QEMU

## 6. The output directory: where everything goes

## 7. The BeagleBone Black: beaglebone_defconfig

## 8. Configuration: menuconfig, defconfigs and the toolchain

## 9. Your own tree: br2-external

## 10. Customizing the root filesystem

## 11. Your own package

## 12. The kernel and the bootloader

## 13. The card image: genimage

## 14. Rebuilding: what make tracks and what it doesn't

## 15. Working on source code: OVERRIDE_SRCDIR

## 16. The SDK: building applications outside Buildroot

## 17. When a build fails

## 18. Reproducible builds, licenses, vulnerabilities and updates

## 19. Releases and upgrades

## 20. Troubleshooting by symptom

## 21. Not covered here

## References

---

Version 1 · 2026-11-15
