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

When the kernel's boot ends, it has mounted a filesystem as `/`, mounted devtmpfs on its `/dev`, and started one program, `init`, as process 1. The [kernel note](../kernel/kernel.md) (section 13) covers how it gets there. The **root filesystem** is that filesystem: a directory tree, stored where the kernel can mount it (an SD card partition, eMMC, NFS, or RAM), holding `init` and everything `init` needs. From this point on, the kernel only answers system calls; everything the system does is decided by files in the root filesystem.

**What must be in it**, the minimum for a BeagleBone Black that boots to a shell:

| Item | Why | In this note |
|---|---|---|
| `/sbin/init` | The kernel runs it as PID 1. With an initramfs, `/init` instead | BusyBox (section 4) |
| The dynamic loader and the C library | Every dynamically linked program needs them before `main()` runs | `/lib/ld-linux-armhf.so.3`, `libc.so.6`, `libm.so.6` (section 5) |
| A shell and the basic commands | Startup scripts are shell scripts | BusyBox |
| An empty `/dev` | The kernel mounts devtmpfs there; without it, `devtmpfs: error mounting -2` | section 6 |
| Empty `/proc`, `/sys`, `/tmp`, `/run` | Mount points for virtual filesystems | section 3 |
| `/etc/inittab` and a startup script | What `init` does: mount filesystems, start services, open a login on the console | section 7 |
| `/etc/passwd`, `/etc/group` | User and group names; logins | section 8 |
| `/lib/modules/<release>/` | Modules for this exact kernel (kernel note, section 8) | `modules_install` (section 11) |

**The jobs of PID 1.** `init` is an ordinary program with three special duties:

- **Start everything else**: mount the remaining filesystems, start daemons, open logins.
- **Reap orphans**: a process whose parent exits becomes a child of PID 1, and stays a zombie until PID 1 calls `wait()` on it.
- **Never exit.** If PID 1 exits, the kernel panics with `Kernel panic - not syncing: Attempted to kill init! exitcode=0x...` (section 19). At shutdown, `init` stops the other processes, unmounts filesystems and asks the kernel to reboot or power off.

**The startup chain** of the system built in Part 2:

```
 kernel            mounts the root (or unpacks an initramfs), mounts devtmpfs on /dev,
   │               opens /dev/console, runs /sbin/init
   ▼
 /sbin/init        BusyBox init reads /etc/inittab
   │
   ├── sysinit ──> /etc/init.d/rcS: mount /proc, /sys, /tmp, /run, /dev/pts;
   │               device handling (mdev), hostname, network, services
   │
   └── respawn ──> getty on ttyS0 ──> login ──> /bin/sh
```

**Three ways to get a root filesystem:**

| | By hand (this note) | A distribution (Debian) | A build system (Buildroot, Yocto) |
|---|---|---|---|
| Effort to a first prompt | An afternoon | Minutes, with a ready image | An hour of build time |
| Size | 2 to 5 MiB without modules | Hundreds of MiB | A few MiB to as much as you add |
| Adding software | Cross-compile it yourself (section 14) | `apt install` | A package or recipe |
| Reproducible | Only if you script it | Hard to pin exactly | Yes, by design |
| Good for | Understanding, rescue images, very small systems | Prototypes, lab boards | Products |

The rest of this note builds the first column. Every piece built by hand here has a Buildroot option and a Yocto variable (section 13), so this is also how to read what a build system produces.

## 2. The example board, root filesystem and files

| Element | Value |
|---|---|
| Board | BeagleBone Black, booted from the SD card by mainline U-Boot ([bootloader note](../bootloader/bootloader.md)) |
| Kernel | Linux 7.1-rc1, `omap2plus_defconfig`, release `7.1.0-rc1-g66edb901bf87` ([kernel note](../kernel/kernel.md)). Kernel source in `~/bbb/linux`, build directory `~/bbb/build` |
| Cross-compiler | Arm GNU Toolchain 15.2.Rel1, `arm-none-linux-gnueabihf-`, glibc |
| C library on the board | The toolchain's own glibc, copied from its sysroot (section 5) |
| Basic commands | BusyBox 1.37.0, source in `~/bbb/busybox-1.37.0`, build directory `~/bbb/busybox-build` |
| Staging tree | `~/bbb/rootfs`: the root filesystem as a directory on the build machine |
| Images | `~/bbb/images/` |
| Root filesystem on the board | ext4 on SD card partition 2, `/dev/mmcblk0p2`. During development, NFS from `/srv/nfs/bbb` on the build machine |

The commands use these variables:

```sh
$ export ARCH=arm
$ export CROSS_COMPILE=arm-none-linux-gnueabihf-
$ export ROOTFS=~/bbb/rootfs
$ export SYSROOT=$(${CROSS_COMPILE}gcc -print-sysroot)
$ export KREL=7.1.0-rc1-g66edb901bf87
```

`SYSROOT` is the toolchain's copy of the target's C library and headers: in the Arm GNU Toolchain, the `arm-none-linux-gnueabihf/libc` directory of the installation. The board's libraries are copied from there (section 5). `KREL` must match `uname -r` on the board exactly, because it names the module directory.

**Where the space goes** in the finished system (**Typical**, orders of magnitude for this kind of build):

| Part | Size |
|---|---|
| BusyBox, dynamically linked, default configuration, stripped | About 1 MiB |
| glibc: loader, `libc.so.6`, `libm.so.6`, `libresolv.so.2`, stripped | About 2 MiB |
| `/etc`: configuration and scripts | Tens of KiB |
| `/lib/modules/<release>/`: all ~860 modules of `omap2plus_defconfig`, stripped | By far the largest part; section 18 keeps only the needed ones |

## 3. The directory layout: what goes where

The layout follows the [Filesystem Hierarchy Standard](https://refspecs.linuxfoundation.org/FHS_3.0/fhs/index.html) (FHS). Programs find each other and their files through these paths, so keeping to them saves configuring every program.

| Directory | Contents | On the minimal BeagleBone Black system |
|---|---|---|
| `/bin`, `/sbin` | Essential commands; `sbin` for administration | BusyBox links |
| `/usr/bin`, `/usr/sbin` | Other commands | BusyBox links, added software |
| `/lib` | Dynamic loader, C library, kernel modules, firmware | `ld-linux-armhf.so.3`, `libc.so.6`, `libm.so.6`, `modules/`, `firmware/` |
| `/usr/lib` | Other libraries | Added software |
| `/etc` | Configuration, startup scripts | `inittab`, `init.d/`, `fstab`, `passwd`, `group`, `shadow`, `hostname`... |
| `/dev` | Device nodes | Empty: devtmpfs is mounted on it |
| `/proc`, `/sys` | Kernel information and settings (virtual) | Empty mount points |
| `/tmp` | Temporary files, cleared at boot | Empty, mode `1777`, tmpfs |
| `/run` | Runtime state: PID files, sockets, lease files | Empty, tmpfs |
| `/var` | Variable data: logs, spool, state | `var/log`; `var/run` a link to `../run` |
| `/root` | Root's home directory | Empty, mode `0700` |
| `/home`, `/mnt`, `/media`, `/opt`, `/srv` | Users, temporary mounts, add-on software | Optional |
| `/boot` | Kernel and DTB, when the bootloader reads them from the root filesystem | Optional (bootloader note) |

**What the kernel itself depends on** is short. The rest is convention between userspace programs:

| Path | Used by | Configurable |
|---|---|---|
| `/sbin/init`, `/etc/init`, `/bin/init`, `/bin/sh` | The kernel, to start PID 1 (kernel note, section 13) | `init=` on the command line |
| `/init` | The kernel, in an initramfs | `rdinit=` |
| `/dev` | devtmpfs mount point | No |
| `/dev/console` | The first process's standard input and output | The kernel opens it in its built-in initramfs before mounting a disk root, so a disk root doesn't need it (section 6) |
| `/sbin/modprobe` | The kernel, to load a module it needs (`request_module()`) | `/proc/sys/kernel/modprobe` |
| `/lib/firmware/` | The kernel's firmware loader | `firmware_class.path=` |
| `/lib/ld-linux-armhf.so.3` | Every dynamically linked armhf program: the path is written in the program (section 5) | Only by relinking |
| `/lib/modules/<release>/` | `modprobe`, `depmod` | No, by convention |

**Merged `/usr`.** Debian (since 12) and other recent distributions make `/bin`, `/sbin` and `/lib` symbolic links to their `/usr` counterparts, and systemd expects that layout. For a hand-built BusyBox system either layout works. Keep the classic one here, and pick merged `/usr` when the system will run systemd (section 12).

**The skeleton:**

```sh
$ mkdir -p $ROOTFS && cd $ROOTFS
$ mkdir -p bin sbin usr/bin usr/sbin lib usr/lib \
           etc/init.d etc/network dev proc sys tmp run var/log root home mnt
$ ln -s ../run var/run
$ chmod 1777 tmp
$ chmod 0700 root
```

The tree belongs to your user on the build machine. Who owns the files on the board is a separate question (section 8).

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
