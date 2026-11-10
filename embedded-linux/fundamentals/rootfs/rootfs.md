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

BusyBox puts about 400 small commands ("applets") into one executable: a shell (`ash`), the file and text tools (`ls`, `cp`, `grep`, `sed`, `vi`), the system tools (`mount`, `init`, `getty`, `login`, `mdev`, `modprobe`), and network tools (`ip`, `udhcpc`, `ntpd`, `syslogd`). Each command is a link to `busybox`, which looks at the name it was started under (`argv[0]`) to know which applet to run. It is configured with the same Kconfig menus as the kernel. Its applets have fewer options than GNU coreutils and util-linux, which is the price of a binary of about 1 MiB. BusyBox is GPLv2: shipping it in a product means offering its source and your configuration.

**Getting the source:**

```sh
$ cd ~/bbb
$ wget https://busybox.net/downloads/busybox-1.37.0.tar.bz2
$ wget https://busybox.net/downloads/busybox-1.37.0.tar.bz2.sha256
$ sha256sum -c busybox-1.37.0.tar.bz2.sha256
$ tar xf busybox-1.37.0.tar.bz2
```

**Configuring**, in a separate build directory like the kernel's `O=`:

```sh
$ cd ~/bbb/busybox-1.37.0
$ BB=~/bbb/busybox-build
$ mkdir -p $BB
$ make O=$BB defconfig            # nearly every applet
$ make O=$BB menuconfig           # adjust
```

The options worth looking at:

| Option | Menu | Here |
|---|---|---|
| `CONFIG_CROSS_COMPILER_PREFIX` | Settings | Empty: the prefix is given with `CROSS_COMPILE=` on the command line. Setting it here instead saves typing |
| `CONFIG_STATIC` | Settings, "Build static binary" | Off: dynamic, with the libraries of section 5. See the table below |
| `CONFIG_INSTALL_APPLET_SYMLINKS` | Settings, "Applets links" | Default: symbolic links |
| `CONFIG_FEATURE_SUID` | Settings | Default: on. Matters only with users other than root (section 8) |
| `CONFIG_TC` | Networking Utilities | **Off**, see below |
| `CONFIG_INIT`, `CONFIG_GETTY`, `CONFIG_LOGIN`, `CONFIG_MDEV`, `CONFIG_SYSLOGD`, `CONFIG_KLOGD`, `CONFIG_UDHCPC`, `CONFIG_IFUP`, `CONFIG_NTPD` | Various | On in `defconfig`; the system of sections 6 to 9 uses them |
| `CONFIG_TELNETD`, `CONFIG_HTTPD`, `CONFIG_FTPD`, `CONFIG_TFTPD` | Networking Utilities | On in `defconfig`, never started unless a script does. Turn them off for a product (section 17) |

**A known build failure: `tc`.** BusyBox 1.37.0's `tc` applet uses the CBQ traffic-control definitions, which recent kernel UAPI headers (Linux 6.8 and later) no longer provide. A toolchain as recent as 15.2.Rel1 ships such headers, so the build stops in `networking/tc.c` with `TCA_CBQ_...` undeclared. Turn `tc` off:

```sh
$ sed -i 's/^CONFIG_TC=y/# CONFIG_TC is not set/' $BB/.config
$ make O=$BB oldconfig
```

For traffic control on the board, use the full `iproute2` instead (section 14).

**Building and installing into the staging tree:**

```sh
$ make O=$BB CROSS_COMPILE=$CROSS_COMPILE -j$(nproc)
$ make O=$BB CROSS_COMPILE=$CROSS_COMPILE CONFIG_PREFIX=$ROOTFS install
```

`install` copies `busybox` to `$ROOTFS/bin/` and creates one link per applet in `bin`, `sbin`, `usr/bin` and `usr/sbin`, plus `linuxrc` at the top (only used by the old initrd mechanism; it can be deleted). The list is in `$BB/busybox.links`. `CONFIG_PREFIX` is mandatory in practice: without it, BusyBox installs into `$BB/_install`.

**Checking the result** on the build machine (the binary can't run there):

```sh
$ file $ROOTFS/bin/busybox
$ ${CROSS_COMPILE}readelf -d $ROOTFS/bin/busybox | grep NEEDED
$ ${CROSS_COMPILE}readelf -l $ROOTFS/bin/busybox | grep interpreter
$ ls -l $ROOTFS/sbin/init
```

**Expected** (the library list depends on the configuration: BusyBox's link script drops libraries the binary doesn't use):

```
.../bin/busybox: ELF 32-bit LSB ..., ARM, EABI5 version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux-armhf.so.3, for GNU/Linux ..., stripped
 0x00000001 (NEEDED)                     Shared library: [libm.so.6]
 0x00000001 (NEEDED)                     Shared library: [libresolv.so.2]
 0x00000001 (NEEDED)                     Shared library: [libc.so.6]
      [Requesting program interpreter: /lib/ld-linux-armhf.so.3]
lrwxrwxrwx 1 me me ... .../sbin/init -> ../bin/busybox
```

`ARM, EABI5` and the `armhf` loader confirm a 32-bit hard-float binary for the BeagleBone Black. These three files are exactly what section 5 copies.

**Static or dynamic:**

| | Static (`CONFIG_STATIC=y`) | Dynamic (default) |
|---|---|---|
| Libraries on the board | None | Loader, `libc.so.6`, `libm.so.6`, `libresolv.so.2` |
| Size of `busybox` | Larger | Smaller |
| Size of the whole system | Smaller when BusyBox is the only program | Smaller as soon as other programs share the C library |
| With glibc | Links, with warnings such as `Using 'getaddrinfo' in statically linked applications requires at runtime the shared libraries from the glibc version used for linking`. Name and user lookups then depend on libraries that aren't there | Nothing special |
| Use | Initramfs, rescue systems; best with musl (section 5) | A normal root filesystem |

**Replacing an applet with the full tool.** A package installed later in the same path replaces the link with its own binary (section 14). Install BusyBox first, the full tools after it, and don't run BusyBox's `install` again afterwards, or it puts its links back.

## 5. The C library and other shared libraries

**What happens when a dynamic program starts.** The kernel reads the program's `PT_INTERP` header (the "program interpreter" of `readelf -l`), loads that file, here `/lib/ld-linux-armhf.so.3`, and hands control to it. The loader reads the program's `NEEDED` entries and looks for each library in: the program's `RUNPATH`, `LD_LIBRARY_PATH`, the cache `/etc/ld.so.cache`, then the default directories `/lib` and `/usr/lib`. It maps them, resolves symbols and only then calls `main()`. On a hand-built root filesystem there's no `ld.so.cache` and none is needed, as long as the libraries are in `/lib` or `/usr/lib`. `ldconfig` is only needed to build the cache for libraries placed in other directories listed in `/etc/ld.so.conf`.

Two consequences:

- **The loader's path is written into every program.** If it's missing, `exec` fails with `ENOENT` (error -2), and a shell says `not found` about a file that is visibly there. This is the most common first-boot failure of a hand-built root filesystem (section 19).
- **The libraries must come from the toolchain that built the programs**, the same glibc version or newer. A program linked against a newer glibc fails with ``version `GLIBC_2.xx' not found`` ([toolchain note](../cross-compilation-toolchain/cross-compilation-toolchain.md)).

**Finding what a program needs.** There's no `ldd` for a cross target: `ldd` runs the program. `readelf` gives the same information, one level at a time. This small script (written for this note, untested) follows the dependencies recursively in the sysroot:

```sh
#!/bin/sh
# needed-libs.sh FILE...: shared libraries the files need, recursively,
# looked up in the toolchain sysroot. Prints "name path" per library.
: "${CROSS_COMPILE:?}" "${SYSROOT:?}"
todo="$*"; seen=""
while [ -n "$todo" ]; do
    set -- $todo; f=$1; shift; todo="$*"
    for lib in $(${CROSS_COMPILE}readelf -d "$f" 2>/dev/null |
                 sed -n 's/.*Shared library: \[\(.*\)\]/\1/p'); do
        case " $seen " in *" $lib "*) continue ;; esac
        seen="$seen $lib"
        path=$(find "$SYSROOT/lib" "$SYSROOT/usr/lib" -maxdepth 1 -name "$lib" 2>/dev/null | head -n 1)
        echo "$lib ${path:-NOT-FOUND}"
        [ -n "$path" ] && todo="$todo $path"
    done
done
```

Run it on everything in the tree that is an ELF file (`find $ROOTFS -type f -exec file {} + | grep ELF`). Add the loader named by `readelf -l ... | grep interpreter`, which the script doesn't print.

**Copying them:**

```sh
$ cd $SYSROOT/lib
$ cp -L ld-linux-armhf.so.3 libc.so.6 libm.so.6 libresolv.so.2 $ROOTFS/lib/
$ ${CROSS_COMPILE}strip --strip-unneeded $ROOTFS/lib/*.so*
```

- **`cp -L`** follows symbolic links, so a link in the sysroot becomes a real file under the name the programs ask for. The program asks for the **soname** (`libc.so.6`); the development link without a number (`libc.so`) is only used when linking and has no place on the board.
- **Strip with the cross `strip`**: the build machine's `strip` may not understand Arm objects. Keep the unstripped sysroot for debugging with GDB (section 19).
- **For a development system, copying the whole `$SYSROOT/lib`** (all `*.so*` files) avoids chasing libraries one at a time, for a few more MiB. For a product, copy only what the script lists.

**glibc details that matter on a small system:**

- Since glibc 2.34, `libpthread`, `libdl`, `libutil` and `libanl` are part of `libc.so.6`, along with the `files` and `dns` name-service modules. No `libnss_*` files are needed for `/etc/passwd`, `/etc/group` and `/etc/hosts` lookups.
- A three-line `/etc/nsswitch.conf` states the lookup order instead of relying on glibc's built-in defaults:

  ```
  passwd: files
  group:  files
  shadow: files
  hosts:  files dns
  ```

- glibc 2.39 dropped `libcrypt`, and Arm's toolchain releases since 14.2.Rel1 no longer ship it. BusyBox doesn't need it (it has its own `crypt()`), but other programs that check passwords do: build `libxcrypt` for them (section 14).
- Locales (`/usr/lib/locale`) and character-set converters (`gconv`) aren't needed for BusyBox. Without them, programs run in the `C` locale.
- Time zones: copy one file from a zoneinfo database to `/etc/localtime`, or set `TZ` (section 9).

**Which C library.** The C library is chosen with the toolchain; a program built for one doesn't run on another ([toolchain note](../cross-compilation-toolchain/cross-compilation-toolchain.md)).

| | glibc | musl | uClibc-ng |
|---|---|---|---|
| Size on the board | About 2 MiB | Under 1 MiB, loader and libc in one file | About 1 MiB, configurable |
| Static linking | Discouraged (name lookups, locales) | Clean | Works |
| Compatibility with desktop software | Complete | Very good; some glibc extensions missing | Lower |
| Needed by | systemd, most vendor binaries | Alpine, small static tools | Systems without an MMU |
| Toolchains | Arm GNU Toolchain, distributions, Bootlin, Buildroot, Yocto | Bootlin, Buildroot, Yocto, musl-cross-make | Bootlin, Buildroot |

## 6. /dev: devtmpfs, mdev and udev

A **device node** is a special file, character or block, whose major and minor numbers point to a driver: `/dev/ttyS0`, `/dev/mmcblk0p2`, `/dev/null`. Programs open these files to reach the hardware. Something must create them, with the right owners and permissions, including for devices that appear later (a USB stick, a module loaded at run time).

**Four ways, from simplest to fullest:**

| Method | Who creates the nodes | Owners, permissions, symbolic links | Loads modules for new devices | Use |
|---|---|---|---|---|
| Static nodes | You, with `mknod` when building the image | Fixed at build time | No | Old kernels without devtmpfs; an initramfs's `/dev/console` |
| devtmpfs alone | The kernel, as drivers register devices | Kernel defaults: mostly `root`, `0600` | No | The simplest working system |
| devtmpfs + `mdev` | The kernel; BusyBox `mdev` applies rules | `/etc/mdev.conf` | Yes, with a rule | BusyBox systems |
| devtmpfs + `udev` | The kernel; `systemd-udevd` (or the standalone `eudev`) applies rules | `/etc/udev/rules.d/`, `/lib/udev/rules.d/` | Yes | Distributions, systemd systems |

**devtmpfs** is a filesystem in which the kernel itself creates a node for every device with a major and minor number, named as the driver says. `CONFIG_DEVTMPFS=y` creates it and `CONFIG_DEVTMPFS_MOUNT=y` mounts it on `/dev` after mounting a disk or NFS root. `omap2plus_defconfig` has both (kernel note, section 13). Two consequences:

- **The root filesystem needs an empty `/dev` directory and nothing in it.** Nodes created with `mknod` in the image are hidden as soon as devtmpfs is mounted over them. `/dev/console` isn't needed either: the kernel opened the console in its built-in initramfs, before mounting the root.
- **`CONFIG_DEVTMPFS_MOUNT` doesn't apply to an initramfs.** When the kernel runs `/init` from an initramfs, it never reaches the code that mounts devtmpfs; the initramfs must run `mount -t devtmpfs devtmpfs /dev` itself (section 10).

**mdev.** BusyBox's `mdev` has two modes:

- **`mdev -s`** ("scan") walks `/sys` once and applies the rules to the devices that already exist (coldplug).
- **`mdev -d`** runs as a daemon and receives the kernel's device events (uevents) over a netlink socket, for devices that appear later (hotplug). The older way to receive them is `echo /sbin/mdev > /proc/sys/kernel/hotplug`, which makes the kernel start one `mdev` process per event. It needs `CONFIG_UEVENT_HELPER`, and the netlink daemon is the better choice.

Start the daemon first, then scan, so that no event falls between the two (section 7 shows the script).

**`/etc/mdev.conf`** is a list of rules. The first rule whose regular expression matches the device name applies, and the rest are ignored:

```
# <device regex>  <uid>:<gid> <mode>  [>link or =newname]  [@cmd on creation, $cmd on removal, *cmd both]
null              0:0   666
zero              0:0   666
full              0:0   666
random            0:0   666
urandom           0:0   666
tty               0:5   666
ttyS[0-9]*        0:20  660
mmcblk[0-9].*     0:6   660
rtc0              0:0   644   >rtc
$MODALIAS=.*      0:0   660   @modprobe "$MODALIAS"
```

The group numbers match `/etc/group` of section 8 (`tty` 5, `disk` 6, `dialout` 20). `>rtc` creates a `/dev/rtc` link to `rtc0`. The last line handles events that carry a `MODALIAS` variable, which are devices without a driver yet, and runs `modprobe` on it. Without `/etc/mdev.conf` at all, `mdev` creates every node as `root:root 0660`.

**Loading modules for devices present at boot.** An event happens only once. Devices that the kernel found before `mdev -d` started already had their event, and `mdev -s` only scans devices that have nodes. The usual workaround, in the startup script, is to read every `modalias` file in `/sys` and load what matches:

```sh
find /sys/devices -name modalias | while read f; do modprobe -q "$(cat "$f")"; done 2>/dev/null
```

`modprobe` looks each alias up in `/lib/modules/$(uname -r)/modules.alias` (kernel note, section 14). Aliases with no matching module fail quietly. On a system with every needed driver built in, this line does nothing and can go.

**Firmware.** Some drivers load a firmware file when they probe: Wi-Fi chips, some USB devices, the AM335x's power-management coprocessor (`am335x-pm-firmware.elf`, needed for suspend). The kernel reads these files from `/lib/firmware/` itself; no userspace helper is involved. A driver that probes **before** the root filesystem is mounted can't find them: build the firmware into the kernel (`CONFIG_EXTRA_FIRMWARE`), or build the driver as a module so it probes later.

**`/dev/pts` and `/dev/shm`** aren't created by devtmpfs:

- **`/dev/pts`** (filesystem `devpts`) holds the pseudo-terminals that SSH sessions, `script` and terminal multiplexers use. Without it, SSH logins fail to get a terminal.
- **`/dev/shm`** (a tmpfs) holds POSIX shared memory (`shm_open()`).

Create both directories in the startup script after devtmpfs is mounted, then mount them from `/etc/fstab` (section 9).

## 7. init: inittab and the startup scripts

**BusyBox `init`** reads `/etc/inittab` and runs the entries in order of action. **Without `/etc/inittab`** it uses built-in defaults:

```
::sysinit:/etc/init.d/rcS
::askfirst:-/bin/sh                 (on the console, then on tty2, tty3 and tty4)
::ctrlaltdel:/sbin/reboot
::shutdown:/sbin/swapoff -a
::shutdown:/bin/umount -a -r
::restart:/sbin/init
```

That is enough to boot to a shell, which is why a BusyBox system with no `/etc` at all still reaches a prompt. It says `Please press Enter to activate this console.`, then opens a shell with no job control (`can't access tty; job control turned off`), because the console isn't a real terminal device.

**The format** is `<id>:<runlevels>:<action>:<process>`:

- **`id`** is the terminal the process runs on, without `/dev/` (`ttyS0`). Empty means `/dev/console`.
- **`runlevels`** is ignored by BusyBox `init`. SysV `init` uses it (section 12).
- **`process`** is a command line. A leading `-` starts the shell as a login shell, which reads `/etc/profile`.

| Action | When | Waits for it | Typical use |
|---|---|---|---|
| `sysinit` | First, at boot | Yes | `/etc/init.d/rcS` |
| `wait` | After `sysinit` | Yes | One-time setup that must finish first |
| `once` | After `sysinit` | No | Starting something once |
| `respawn` | After the above; restarted whenever it exits | No | `getty`; a daemon running in the foreground |
| `askfirst` | Like `respawn`, but prints `Please press Enter to activate this console.` first | No | A development shell |
| `ctrlaltdel` | When `init` receives `SIGINT` | | `reboot` |
| `shutdown` | At `halt`, `reboot`, `poweroff` | Yes | Stop services, unmount |
| `restart` | When `init` receives `SIGQUIT` | | Re-execute `init` |

**`/etc/inittab`** for the BeagleBone Black:

```
# /etc/inittab - BusyBox init
::sysinit:/etc/init.d/rcS

# login prompt on the serial console (J1, 115200 8N1)
ttyS0::respawn:/sbin/getty -L ttyS0 115200 vt100

::ctrlaltdel:/sbin/reboot
::shutdown:/etc/init.d/rcK
::shutdown:/bin/umount -a -r
::restart:/sbin/init
```

- **The terminal is `ttyS0`**: the kernel's name for UART0 with the 8250 driver. `ttyO0` is the name used by the old `omap-serial` driver, which mainline configurations no longer use. A `getty` on a terminal that doesn't exist respawns and fails in a loop (section 21).
- `getty -L` treats the line as local, without modem control signals. `getty` prints `/etc/issue` and `<hostname> login:`, then runs `/bin/login`.
- **For development, skip the login:** `ttyS0::respawn:-/bin/sh` gives a root shell on the serial console, with job control because `ttyS0` is a real terminal. Keep it out of products (section 17).

**`/etc/init.d/rcS`**, the startup script, run once by the `sysinit` line:

```sh
#!/bin/sh
# /etc/init.d/rcS - run once at boot by init

# filesystems from /etc/fstab: /proc, /sys, /dev/pts, /dev/shm, /tmp, /run (section 9)
mkdir -p /dev/pts /dev/shm
mount -a
mkdir -p /run/lock

hostname -F /etc/hostname

# devices: hotplug daemon first, then existing devices, then their modules (section 6)
mdev -d
mdev -s
find /sys/devices -name modalias | while read f; do modprobe -q "$(cat "$f")"; done 2>/dev/null

# keep kernel messages off the console once userspace runs (still in dmesg)
dmesg -n 1

# services, in name order: S01syslog, S10network, ...
for s in /etc/init.d/S[0-9][0-9]*; do
    [ -x "$s" ] && "$s" start
done
```

**`/etc/init.d/rcK`**, run at shutdown, stops the same services in reverse order:

```sh
#!/bin/sh
# /etc/init.d/rcK - run at shutdown by init
for s in $(ls -r /etc/init.d/S[0-9][0-9]* 2>/dev/null); do
    [ -x "$s" ] && "$s" stop
done
```

**A service script** follows the `S<two digits><name>` convention, which is also Buildroot's:

```sh
#!/bin/sh
# /etc/init.d/S01syslog - system and kernel logs (section 9)
case "$1" in
    start) syslogd -C64; klogd ;;
    stop)  killall klogd syslogd ;;
    *)     echo "usage: $0 start|stop"; exit 1 ;;
esac
```

Make them executable: `chmod 755 $ROOTFS/etc/init.d/rc* $ROOTFS/etc/init.d/S*`. Two rules prevent most startup-script failures: **Unix line endings** (a script saved with CRLF asks for the interpreter `/bin/sh\r`, which doesn't exist), and **`#!/bin/sh`**, not `#!/bin/bash`, unless bash is installed.

**Supervision.** BusyBox `init` restarts only what `/etc/inittab` lists. A daemon started from a script that later crashes stays dead. For the application of a product, the simplest supervision is a `respawn` line that runs it **in the foreground**: `::respawn:/usr/bin/myapp`. Section 12 compares init systems that do more.

**What the console shows** when this system boots. **Expected**, after the kernel lines of the kernel note's section 11:

```
[    ...] Run /sbin/init as init process

bbb login: root
Password:
~ #
```

And at `reboot`, from BusyBox `init`. **Expected**:

```
The system is going down NOW!
Sent SIGTERM to all processes
Sent SIGKILL to all processes
Requesting system reboot
[    ...] reboot: Restarting system
```

## 8. Users, logins and file ownership

**The three files.** User and group names exist only in these files; the kernel knows only numbers.

```
# /etc/passwd   name:password:UID:GID:comment:home:shell
root:x:0:0:root:/root:/bin/sh
daemon:x:1:1:daemon:/usr/sbin:/bin/false
nobody:x:65534:65534:nobody:/nonexistent:/bin/false
```

```
# /etc/group    name:password:GID:members
root:x:0:
daemon:x:1:
tty:x:5:
disk:x:6:
dialout:x:20:
nogroup:x:65534:
```

```
# /etc/shadow   name:hash:last change:min:max:warn:inactive:expire:
root::20400:0:99999:7:::
```

- `x` in `/etc/passwd` means "the password is in `/etc/shadow`". `/etc/shadow` is readable only by root: mode `0600` (or `0640` with group `shadow`).
- **An empty hash field means no password**: `login` accepts `root` without asking. That's convenient on a development board, and it must never ship (section 17).
- The third field of `/etc/shadow` is the day of the last change, counted from 1 January 1970. Any value works here.
- Add the shells to **`/etc/shells`** (`/bin/sh` and `/bin/ash`, one per line). Some programs, Dropbear among them, refuse logins for users whose shell isn't listed.

**Setting a password.** Generate a SHA-512 hash on the build machine and paste it into the second field of `/etc/shadow`:

```sh
$ openssl passwd -6
Password:
Verifying - Password:
$6$...
```

BusyBox `login` verifies `$6$` hashes when `CONFIG_USE_BB_CRYPT_SHA` is on (it is in `defconfig`). `passwd` on the board does the same, but on a read-only root filesystem it can't write `/etc/shadow` (section 15).

**How a login works:** `init` starts `getty` on `ttyS0` (section 7). `getty` opens the line, sets its speed, prints the prompt and reads a user name, then runs `login`. `login` checks the hash in `/etc/shadow`, changes to the user's UID and GID, changes to the home directory and runs the shell from `/etc/passwd`. A login shell reads `/etc/profile`.

**setuid BusyBox.** Commands like `su`, `passwd` and `ping` need root privileges when a non-root user runs them. With a single `busybox` binary, that means making it setuid root (`chmod 4755 bin/busybox`), and BusyBox drops the privileges for every applet that doesn't need them (`CONFIG_FEATURE_SUID`). A system where everything runs as root doesn't need it. In a product with non-root users, set it deliberately: it's the one setuid file in the system.

**File ownership: the build machine and the board disagree.** Everything in `$ROOTFS` belongs to your user on the build machine, typically UID 1000. Copied as is, the board sees `/bin/busybox`, `/etc/shadow` and the rest owned by UID 1000, a user that doesn't exist there. Things then break in ways that don't point at the cause: a setuid bit that does nothing, an SSH server that refuses a key file not owned by root, `/etc/shadow` readable by "someone". The files on the board must belong to root (UID 0), except the few owned by service users.

| Method | How | Notes |
|---|---|---|
| `sudo chown` on the copy | Copy the tree to the card or NFS export, then `sudo chown -R 0:0` it (section 11) | Simplest. Set setuid bits **after** `chown`, which clears them |
| `fakeroot` | Run the ownership changes and the image creation inside one `fakeroot` session | No root needed on the build machine. The tree on disk keeps your UID; only the image has the right owners |
| Options of the image tools | `mksquashfs -all-root`, `cpio -R 0:0`, the kernel's `CONFIG_INITRAMFS_ROOT_UID` (section 10) | Every file belongs to root, with no exceptions |
| A device or permission table | Buildroot's `makedevs` format: one line per path with owner, mode and type | What build systems use |

With `fakeroot`, everything that must see root ownership happens in the same session:

```sh
$ mkdir -p ~/bbb/images
$ fakeroot sh -c "
    chown -R 0:0 $ROOTFS
    chmod 4755 $ROOTFS/bin/busybox
    chmod 0600 $ROOTFS/etc/shadow
    mkfs.ext4 -q -L rootfs -d $ROOTFS ~/bbb/images/rootfs.ext4 256M
"
```

`fakeroot` intercepts `chown` and `stat` for the programs it runs and remembers the fake owners in memory, so `mkfs.ext4` writes root-owned files into the image. When the session ends, the fake owners are lost. To build several images from the same fake state, save and reload it with `fakeroot -s <file>` and `fakeroot -i <file>`.

**Permissions to get right:**

| Path | Mode |
|---|---|
| `/tmp` | `1777` (everyone writes, only the owner deletes) |
| `/root` | `0700` |
| `/etc/shadow` | `0600` |
| `/etc/passwd`, `/etc/group` | `0644` |
| Scripts in `/etc/init.d/` | `0755` |
| `/bin/busybox` | `0755`, or `4755` for setuid applets |

## 9. Configuration: mounts, hostname, network, time and logs

**`/etc/fstab`**, read by `mount -a` in `rcS`:

```
# <file system>  <mount point>  <type>   <options>                     <dump> <pass>
proc             /proc          proc     defaults                       0      0
sysfs            /sys           sysfs    defaults                       0      0
devpts           /dev/pts       devpts   gid=5,mode=620,ptmxmode=666    0      0
tmpfs            /dev/shm       tmpfs    mode=1777,nosuid,nodev         0      0
tmpfs            /tmp           tmpfs    mode=1777,nosuid,nodev         0      0
tmpfs            /run           tmpfs    mode=0755,nosuid,nodev         0      0
```

- The root filesystem isn't listed: the kernel already mounted it, read-write because of `rw` on the command line (kernel note, section 10). For a read-only root, see section 15.
- `gid=5` is the `tty` group of section 8, so that terminals belong to it.
- tmpfs lives in RAM and starts empty at every boot. Without a `size=` option, each one may grow to half the RAM.
- The kernel options these lines rely on (`CONFIG_PROC_FS`, `CONFIG_SYSFS`, `CONFIG_TMPFS`, `CONFIG_UNIX98_PTYS`) are in `omap2plus_defconfig`.

**`/etc/mtab`** should be a link to the kernel's list of mounts, so that `mount` and `df` show the truth:

```sh
$ ln -s /proc/self/mounts $ROOTFS/etc/mtab
```

**Name and hosts:**

```sh
$ echo bbb > $ROOTFS/etc/hostname
$ printf '127.0.0.1\tlocalhost\n127.0.1.1\tbbb\n' > $ROOTFS/etc/hosts
```

**`/etc/profile`**, read by login shells:

```sh
# /etc/profile
export PATH=/usr/sbin:/usr/bin:/sbin:/bin
export PS1='\u@\h:\w\$ '
export TZ=UTC0
umask 022
```

**Network, with BusyBox `ifup`.** `/etc/network/interfaces` uses the Debian format:

```
# /etc/network/interfaces
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
    address 192.168.0.50
    netmask 255.255.255.0
    gateway 192.168.0.1

# or, with a DHCP server:
# iface eth0 inet dhcp
```

```sh
#!/bin/sh
# /etc/init.d/S10network
case "$1" in
    start) ifup -a ;;
    stop)  ifdown -a ;;
esac
```

- `ifup` keeps its state in `/var/run/ifstate`, so `/var/run` must exist and be writable. Here it's the link to `/run` of section 3. Without it, `ifup` fails with `can't open '/var/run/ifstate'`.
- **`inet dhcp` runs `udhcpc`, which only negotiates the lease.** Configuring the interface, the route and `/etc/resolv.conf` is the job of a script, `/usr/share/udhcpc/default.script`, which BusyBox doesn't install. Copy `examples/udhcp/simple.script` from the BusyBox source to that path and make it executable. Without it, `udhcpc` reports a lease and the interface stays without an address.
- **On an NFS root, don't touch `eth0`** (section 11): the kernel configured it to mount the root, and `ifdown` or a new DHCP lease cuts the root filesystem off.
- DNS: `nameserver 192.168.0.1` in `/etc/resolv.conf` for a static setup; the `udhcpc` script writes it with DHCP.

**Time.** The AM335x has an RTC, but the BeagleBone Black has no battery for it: after a power cut, the clock starts again from an arbitrary date, usually in 2000. A wrong date breaks TLS certificate checks, makes `make` complain about clock skew on NFS, and confuses log timestamps. Set the time from the network once it's up:

```sh
ntpd -q -n -p pool.ntp.org      # set once and exit; without -q, keeps running as a daemon
hwclock -w                      # store it in the RTC, which keeps it across reboots, not power cuts
```

Time zones: `TZ=UTC0` in `/etc/profile` is a POSIX time-zone string, which needs no files. For named zones (`Europe/Paris`), copy that file from a zoneinfo database, built for the target or architecture-independent, to `/etc/localtime`.

**Logs.** BusyBox `syslogd` collects messages from programs over the `/dev/log` socket; `klogd` copies kernel messages into it.

| Command | Logs go to | Read with |
|---|---|---|
| `syslogd` | `/var/log/messages`, a file on the root filesystem | `cat`, `tail -f` |
| `syslogd -C64` | A 64 KiB ring buffer in shared memory, nothing written to flash | `logread`, `logread -f` |
| `syslogd -R 192.168.0.10` | A remote syslog server | The server's logs |

The ring buffer suits an SD card or eMMC root, and needs `CONFIG_SYSVIPC` in the kernel. Kernel messages stay in `dmesg` either way.

**Modules to load at boot**, when no device event loads them (section 6). BusyBox doesn't read `/etc/modules` by itself, so a script does:

```sh
#!/bin/sh
# /etc/init.d/S05modules - load the modules listed in /etc/modules
[ "$1" = start ] || exit 0
sed 's/#.*//' /etc/modules 2>/dev/null | while read mod args; do
    [ -n "$mod" ] && modprobe "$mod" $args
done
```

Options go in `/etc/modprobe.d/*.conf` (`options g_ether host_addr=...`), which BusyBox `modprobe` reads like kmod's.

**The `/etc` of the minimal system**, all together:

| File | Section |
|---|---|
| `inittab`, `init.d/rcS`, `init.d/rcK`, `init.d/S01syslog`, `init.d/S05modules`, `init.d/S10network` | 7, 9 |
| `fstab`, `mtab` (link) | 9 |
| `mdev.conf` | 6 |
| `passwd`, `group`, `shadow`, `shells` | 8 |
| `hostname`, `hosts`, `profile`, `network/interfaces`, `resolv.conf` | 9 |
| `nsswitch.conf` | 5 |
| `modules`, `modprobe.d/` | 9 |

## 10. Filesystem images: ext4, squashfs, UBIFS and initramfs

The staging tree becomes a **filesystem image**: a file holding a complete filesystem, written as is to a partition. The format depends on the storage and on whether the root must be writable.

**Block device or raw flash first.** An SD card or eMMC has a controller that hides the flash behind a block device, with wear leveling and bad-block handling: it takes the same filesystems as a hard disk. Raw NAND or NOR flash, soldered without a controller, shows up as an MTD device and needs filesystems built for flash. The BeagleBone Black has eMMC and SD, so ext4 or squashfs; UBIFS is here for boards with raw NAND.

| Format | Writable | Storage | Kernel option | Build tool | Use |
|---|---|---|---|---|---|
| ext4 | Yes (or mounted `ro`) | Block device | `CONFIG_EXT4_FS` | `mkfs.ext4 -d` | The default |
| squashfs | No, compressed | Block device | `CONFIG_SQUASHFS` plus `_XZ` or `_ZSTD` | `mksquashfs` | Read-only root, dm-verity (sections 15, 17) |
| EROFS | No, compressed | Block device | `CONFIG_EROFS_FS` | `mkfs.erofs` | A newer read-only alternative, used by Android |
| F2FS | Yes | Block device | `CONFIG_F2FS_FS` | `mkfs.f2fs` | A writable alternative designed for flash |
| UBIFS on UBI | Yes | Raw NAND | `CONFIG_MTD_UBI`, `CONFIG_UBIFS_FS` | `mkfs.ubifs`, `ubinize` | Boards with raw NAND |
| JFFS2 | Yes | Small NOR | `CONFIG_JFFS2_FS` | `mkfs.jffs2` | Older and small-flash systems |
| cpio (initramfs) | In RAM | Loaded with the kernel | `CONFIG_BLK_DEV_INITRD` | `cpio`, `gen_init_cpio` | Rescue, small systems that run from RAM |
| tar | | | | `tar` | Transport: NFS, update tools, `mmdebstrap` |

Whatever the format, the kernel needs its driver **built in** to mount it as root (kernel note, section 13). `omap2plus_defconfig` has ext4 built in. Check the others before relying on them: `grep -E 'SQUASHFS|EROFS|F2FS|UBIFS' ~/bbb/build/.config`.

**ext4.** `mkfs.ext4 -d` (e2fsprogs 1.43 and later) fills the new filesystem from a directory, with no mount and no root. Run it inside the `fakeroot` session of section 8, so the files belong to root:

```sh
$ fakeroot sh -c "chown -R 0:0 $ROOTFS && mkfs.ext4 -q -L rootfs -d $ROOTFS ~/bbb/images/rootfs.ext4 256M"
```

Given a size, `mkfs.ext4` creates the image file. To write it to the card's second partition and grow it to the partition's size:

```sh
$ sudo dd if=~/bbb/images/rootfs.ext4 of=/dev/sdX2 bs=4M conv=fsync status=progress
$ sudo e2fsck -f /dev/sdX2
$ sudo resize2fs /dev/sdX2
```

A recent `mkfs.ext4` enables features (`metadata_csum`, `orphan_file`) that older tools may not know. A board-side `e2fsck` from an old e2fsprogs refuses such a filesystem. U-Boot reads it, but its ext4 write support refuses `metadata_csum` (`ext4write`, an environment stored on ext4). When that matters, disable the feature: `mkfs.ext4 -O ^metadata_csum ...`.

**squashfs**, compressed and read-only:

```sh
$ mksquashfs $ROOTFS ~/bbb/images/rootfs.squashfs -comp xz -all-root -noappend
```

`-all-root` makes every file belong to root, so no `fakeroot` is needed. To keep a few files owned by other users, use `fakeroot` and leave the option out. Written to partition 2 with `dd`, it's booted with `root=/dev/mmcblk0p2 rootfstype=squashfs ro rootwait`. Since nothing can write to it, `/tmp`, `/run` and the other writable paths must be tmpfs or another partition (section 15).

**UBIFS**, for raw NAND (not on this board). UBI manages the raw flash (wear leveling, bad blocks) and offers volumes; UBIFS is the filesystem in a volume. The parameters come from the NAND chip's geometry: page size, erase block size. For 2 KiB pages and 128 KiB erase blocks without subpages:

```sh
$ mkfs.ubifs -r $ROOTFS -m 2048 -e 126976 -c 2048 -o rootfs.ubifs
$ cat ubinize.cfg
[rootfs]
mode=ubi
image=rootfs.ubifs
vol_id=0
vol_type=dynamic
vol_name=rootfs
vol_flags=autoresize
$ ubinize -o rootfs.ubi -m 2048 -p 128KiB -s 2048 ubinize.cfg
```

`-e` is the logical erase block: 128 KiB minus two pages for UBI's headers. `-c` is the maximum number of logical erase blocks the filesystem may grow to. The kernel mounts it with `ubi.mtd=<partition> root=ubi0:rootfs rootfstype=ubifs`. Wrong geometry is the usual failure, and the kernel's UBI messages name the mismatch.

**initramfs.** An initramfs is a cpio archive that the kernel unpacks into RAM and runs `/init` from (kernel note, section 13). The same tree works with three changes:

1. **`/init` must exist**: `ln -s sbin/init $ROOTFS/init`. With it, the kernel never looks at `root=`.
2. **Mount devtmpfs yourself**: add `mountpoint -q /dev || mount -t devtmpfs devtmpfs /dev` at the top of `rcS`. `CONFIG_DEVTMPFS_MOUNT` doesn't apply to an initramfs (section 6), and the test keeps the same script working on a disk root, where the kernel already mounted it.
3. **Mind the size**: the whole archive is unpacked into RAM and stays there. Leave the modules out, or keep only those needed.

Packed by hand, loaded by U-Boot:

```sh
$ cd $ROOTFS
$ find . | cpio -o -H newc -R 0:0 | gzip -9 > ~/bbb/images/initramfs.cpio.gz
```

`-H newc` is the only cpio format the kernel reads, and `-R 0:0` makes every file belong to root. The kernel unpacks gzip by itself (`CONFIG_RD_GZIP`). U-Boot passes it with the `initrd` line of `extlinux.conf`, or `bootz ${kernel_addr_r} ${ramdisk_addr_r}:${filesize} ${fdt_addr_r}` when loaded by hand (bootloader note). The archive needs no `/dev/console`: the kernel unpacks its built-in default list first, and that list has it.

Built into the kernel instead, one file to boot:

```
CONFIG_INITRAMFS_SOURCE="/home/me/bbb/rootfs"
CONFIG_INITRAMFS_ROOT_UID=1000
CONFIG_INITRAMFS_ROOT_GID=1000
```

The build packs the directory into `zImage`. Files owned by UID and GID 1000 (your user) are stored as root. A directory source replaces the default list, so `/dev/console` must then be added: `CONFIG_INITRAMFS_SOURCE` also takes a file list in the format of `usr/gen_init_cpio`, next to the directory, separated by a space:

```
# ~/bbb/initramfs-extra.txt
dir /dev 0755 0 0
nod /dev/console 0600 0 0 c 5 1
```

Every change to the root filesystem then means rebuilding the kernel. That suits a recovery image that rarely changes, not daily development.

## 11. Installing: SD card, eMMC and NFS root

**On the SD card**, partition 2 of the layout from the bootloader note (MBR; FAT partition 1 with `MLO`, `u-boot.img`, `zImage`, the DTB and `extlinux/extlinux.conf`; ext4 partition 2). Copying the tree is simpler than an image while developing:

```sh
$ sudo mount /dev/sdX2 /mnt/rootfs
$ sudo cp -a $ROOTFS/. /mnt/rootfs/
$ sudo chown -R 0:0 /mnt/rootfs
$ sudo chmod 0600 /mnt/rootfs/etc/shadow
$ sudo chmod 4755 /mnt/rootfs/bin/busybox        # only for setuid applets (section 8)
$ sudo umount /mnt/rootfs
```

**The kernel modules** go in with the kernel, from the kernel build directory, as in section 9 of the kernel note. Installing them into the staging tree keeps the tree complete:

```sh
$ make -C ~/bbb/linux O=~/bbb/build INSTALL_MOD_PATH=$ROOTFS INSTALL_MOD_STRIP=1 modules_install
$ ls $ROOTFS/lib/modules/
7.1.0-rc1-g66edb901bf87
$ rm -f $ROOTFS/lib/modules/$KREL/build       # points into the build machine; useless on the board
```

The directory name must equal `uname -r` on the board. A kernel rebuilt with a new release string needs its modules installed again (kernel note, section 8).

**The kernel command line** from the kernel note's `extlinux.conf` boots this partition:

```
append console=ttyS0,115200n8 root=/dev/mmcblk0p2 rw rootfstype=ext4 rootwait
```

`mmcblk0` is the SD card and `mmcblk1` the eMMC, because of the `mmc0` alias in the device tree. To stop depending on the numbering, use the partition's identifier instead. On an MBR disk it's the disk identifier plus the partition number:

```sh
$ sudo blkid -s PARTUUID -o value /dev/sdX2
1a2b3c4d-02
```

Then `root=PARTUUID=1a2b3c4d-02`.

**A whole-card image.** For several boards, or for the eMMC, build one file with both partitions and their content. `genimage` (Pengutronix, used by Buildroot) does it from a description (**untested**):

```
# genimage.cfg
image boot.vfat {
    vfat {
        files = { "MLO", "u-boot.img", "zImage", "am335x-boneblack.dtb" }
        file extlinux/extlinux.conf { image = "extlinux.conf" }
    }
    size = 64M
}

image sdcard.img {
    hdimage { }

    partition boot {
        partition-type = 0xC
        bootable = "true"
        offset = 4M
        image = "boot.vfat"
    }

    partition rootfs {
        partition-type = 0x83
        image = "rootfs.ext4"
    }
}
```

```sh
$ genimage --config genimage.cfg --inputpath ~/bbb/images --outputpath ~/bbb/images \
           --rootpath $ROOTFS --tmppath /tmp/genimage
$ sudo dd if=~/bbb/images/sdcard.img of=/dev/sdX bs=4M conv=fsync status=progress
```

The input files (`MLO`, `u-boot.img`, `zImage`, the DTB, `extlinux.conf`, `rootfs.ext4`) are expected in `--inputpath`. The partition starts at 4 MiB, like the bootloader note's layout.

**On the eMMC.** Boot from the SD card, copy `sdcard.img` onto it (or fetch it over the network), and on the board:

```
# cat /sys/block/mmcblk1/device/type
MMC
# dd if=/root/sdcard.img of=/dev/mmcblk1 bs=4M conv=fsync
```

`MMC` confirms `mmcblk1` is the eMMC; an SD card says `SD`. Without the S2 button held at power-on, the ROM then boots from the eMMC (bootloader note). The image's `root=` must name `mmcblk1p2`, or better a `PARTUUID`, which stays right on either device.

**Over NFS**, for development: the root filesystem is a directory on the build machine, and a change there is visible on the board without copying anything. On the build machine:

```sh
$ sudo apt install nfs-kernel-server
$ sudo mkdir -p /srv/nfs/bbb
$ sudo rsync -a --chown=0:0 $ROOTFS/ /srv/nfs/bbb/
$ echo '/srv/nfs/bbb 192.168.0.50(rw,sync,no_subtree_check,no_root_squash)' | sudo tee -a /etc/exports
$ sudo exportfs -ra
```

- `no_root_squash` lets the board's root user act as root on the export. Without it, root becomes `nobody` and the system can't write its own files.
- `rsync --chown=0:0` (rsync 3.1 and later) copies with root ownership. Run the same line again after each change in `$ROOTFS`. Adding `--delete` makes the export an exact copy, but also deletes what the board wrote there (logs, keys).

The kernel side, from the bootloader note: `root=/dev/nfs nfsroot=192.168.0.10:/srv/nfs/bbb,vers=3,tcp ip=192.168.0.50:192.168.0.10::255.255.255.0::eth0:off rw`, with `CONFIG_ROOT_NFS`, `CONFIG_IP_PNP` and the Ethernet driver built in. On the root filesystem side:

- **Leave `eth0` alone.** The kernel configured it to mount the root. An `ifdown eth0`, a DHCP client restarting it, or a network manager taking it over cuts the root filesystem off and freezes the system. In `/etc/network/interfaces`, declare it `iface eth0 inet manual`, or leave it out.
- **No `fsck`**, and no line for `/` in `/etc/fstab`.
- **Clocks.** Files written by the board carry its clock, which without NTP is years behind (section 9). `make` and other tools on the build machine then warn about clock skew.

## 12. Init systems: BusyBox init, SysV init, systemd and others

| | BusyBox init | SysV init | systemd | OpenRC | runit, s6 |
|---|---|---|---|---|---|
| Configuration | `inittab` plus shell scripts | `inittab` with runlevels, `/etc/rcN.d/` links to scripts | Unit files | Shell scripts with dependency headers | One directory per service with a `run` script |
| Order | Name order of the scripts | Name order (`S10`, `S20`...) per runlevel | Dependencies, in parallel | Dependencies | Parallel; services wait on each other explicitly |
| Restarts crashed services | Only `inittab` lines | Only `inittab` lines | Yes (`Restart=`) | With its supervisor | Yes, the core of the design |
| Logs | `syslogd` | `syslogd` | journald | `syslogd` | Per-service loggers |
| Hardware watchdog | A separate `watchdog` daemon | A separate daemon | Built in (`RuntimeWatchdogSec=`) | A separate daemon | A separate daemon |
| Also brings | Nothing | Nothing | udev, network (networkd), time sync, login sessions, timers, cgroups for every service | Nothing | Nothing |
| Size and needs | Inside BusyBox | Small | Several MiB, glibc, many kernel options | Small | Very small |
| Found in | Buildroot's default, rescue images | Older distributions, Yocto's former default | Debian, Yocto option, Buildroot option | Alpine, Gentoo | Void Linux (runit), some embedded products (s6) |

**How to choose:**

- **A few services, small storage, fast boot**: BusyBox init, with the product's application on a `respawn` line so it's restarted when it crashes (section 7).
- **Many services with dependencies, network management, hardware watchdog, per-service resource limits, a team used to desktop Linux**: systemd. It costs storage and some boot time, and saves writing everything that systemd already does.
- **Supervision without systemd**: runit or s6, or BusyBox's own `runsv` and `runsvdir` applets, which implement runit.

**A systemd unit** for the application of a product, for comparison with an `inittab` line. In `/etc/systemd/system/myapp.service`:

```ini
[Unit]
Description=My application
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/bin/myapp
Restart=on-failure
RestartSec=2
User=myapp
WatchdogSec=10

[Install]
WantedBy=multi-user.target
```

`WatchdogSec=` restarts the service if it stops calling `sd_notify("WATCHDOG=1")` for 10 seconds: it only works when the application does that. On a serial console, systemd starts a login on the `console=` device by itself (`serial-getty@ttyS0.service`), so there's no `inittab` to write. The kernel options systemd requires (cgroups, `CONFIG_FHANDLE`, `CONFIG_DEVTMPFS` and others) are listed in its `README`.

## 13. Distributions and build systems

**A Debian root filesystem** gives `apt` and every Debian package, for a few hundred MiB. Debian's `armhf` port runs on the Cortex-A8. BeagleBoard.org publishes ready Debian images for the board; to build one yourself with your own kernel, use `mmdebstrap`, which runs the target's package scripts through QEMU user-mode emulation:

```sh
$ sudo apt install mmdebstrap qemu-user-static binfmt-support arch-test
$ sudo mmdebstrap --architectures=armhf --variant=minbase \
      --include=systemd-sysv,udev,kmod,iproute2,openssh-server \
      trixie ~/bbb/debian-rootfs http://deb.debian.org/debian
$ sudo chroot ~/bbb/debian-rootfs passwd root
$ echo bbb | sudo tee ~/bbb/debian-rootfs/etc/hostname
```

Then install the kernel modules into it (section 11) and write it to the card like the BusyBox tree. `chroot` into an `armhf` tree works on an x86 machine because `binfmt_misc` hands Arm binaries to QEMU. It's also a quick way to try a program before the board is at hand. The kernel stays the one from the kernel note; Debian's own kernel packages aren't installed.

**Buildroot** builds a toolchain, BusyBox, packages, the root filesystem and the images in one `make`, with `make beaglebone_defconfig` as a starting point for this board. **Yocto** builds a distribution with recipes; the reference BSP layer has a `beaglebone-yocto` machine. Both are covered in their own notes ([Buildroot](../../linux-build-systems/buildroot/), [Yocto](../../linux-build-systems/yocto/)). What they generate is what this note builds by hand:

| Piece | By hand | Buildroot | Yocto |
|---|---|---|---|
| BusyBox configuration | Section 4 | `BR2_PACKAGE_BUSYBOX_CONFIG` | `busybox` recipe: `defconfig` and `.cfg` fragments in a `.bbappend` |
| C library | Section 5 | Toolchain menu: glibc, musl or uClibc-ng | `TCLIBC` |
| `/dev` management | Section 6 | `BR2_ROOTFS_DEVICE_CREATION_*` (devtmpfs, mdev, eudev) | `VIRTUAL-RUNTIME_dev_manager` |
| Init system | Section 7 | `BR2_INIT_BUSYBOX`, `_SYSV`, `_OPENRC`, `_SYSTEMD` | `INIT_MANAGER` |
| Console login | Section 7 | `BR2_TARGET_GENERIC_GETTY_PORT="ttyS0"` | `SERIAL_CONSOLES = "115200;ttyS0"` |
| Root password | Section 8 | `BR2_TARGET_GENERIC_ROOT_PASSWD` | `extrausers` class, `EXTRA_USERS_PARAMS` |
| Hostname | Section 9 | `BR2_TARGET_GENERIC_HOSTNAME` | `hostname:pn-base-files` |
| Your own files | Section 9 | `BR2_ROOTFS_OVERLAY`, `BR2_ROOTFS_POST_BUILD_SCRIPT` | A recipe, or `ROOTFS_POSTPROCESS_COMMAND` |
| Image format | Section 10 | `BR2_TARGET_ROOTFS_EXT2_4`, `BR2_TARGET_ROOTFS_SQUASHFS` | `IMAGE_FSTYPES` |
| Whole-card image | Section 11 | `genimage`, from `BR2_ROOTFS_POST_IMAGE_SCRIPT` | `wic`, with a `.wks` file |

**When to build by hand:** to learn, for a rescue or factory-test initramfs with a handful of tools, and to debug what a build system produced. For a product, a build system's reproducibility, license reports and CVE tracking (section 17) are worth its learning curve.

## 14. Adding software

**Your own program** is one compiler call and one copy:

```sh
$ ${CROSS_COMPILE}gcc -O2 -Wall -o myapp myapp.c
$ install -D -m 0755 myapp $ROOTFS/usr/bin/myapp
$ ${CROSS_COMPILE}strip $ROOTFS/usr/bin/myapp
```

**A package with a build system** follows one rule: **`--prefix` is where the files will be on the board, `DESTDIR` is where they go now.**

```sh
$ ./configure --host=arm-none-linux-gnueabihf --prefix=/usr --sysconfdir=/etc
$ make -j$(nproc)
$ make DESTDIR=$ROOTFS install
```

`--prefix=$ROOTFS/usr` seems to work, but writes the build machine's path into the program wherever it records its own location (configuration file paths, plugin directories), and those paths don't exist on the board. CMake's equivalent is `-DCMAKE_INSTALL_PREFIX=/usr` at configure time and `DESTDIR=$ROOTFS` at install time; Meson's is `--prefix=/usr` and `meson install --destdir $ROOTFS`. The [toolchain note](../cross-compilation-toolchain/cross-compilation-toolchain.md) covers each build system.

**Libraries need two trees.** A program that uses a library needs its headers, `.so` link and pkg-config file at build time, and only the runtime `.so.N` on the board:

| | Staging tree (`~/bbb/staging`) | Root filesystem (`$ROOTFS`) |
|---|---|---|
| Contains | Headers, `.so` links, `.a` files, `.pc` files, unstripped libraries | Runtime libraries (`libfoo.so.1`), stripped |
| Used by | The compiler and linker on the build machine; GDB | The board |
| Filled by | `make DESTDIR=~/bbb/staging install` for each library | Copying the runtime files from the staging tree |

Building a program against the staging tree:

```sh
$ export PKG_CONFIG_SYSROOT_DIR=~/bbb/staging
$ export PKG_CONFIG_LIBDIR=~/bbb/staging/usr/lib/pkgconfig:~/bbb/staging/usr/share/pkgconfig
$ ./configure --host=arm-none-linux-gnueabihf --prefix=/usr \
      CPPFLAGS=-I$HOME/bbb/staging/usr/include LDFLAGS=-L$HOME/bbb/staging/usr/lib
$ make -j$(nproc) && make DESTDIR=$ROOTFS install
$ cp -a ~/bbb/staging/usr/lib/libfoo.so.* $ROOTFS/usr/lib/
$ ${CROSS_COMPILE}strip --strip-unneeded $ROOTFS/usr/lib/libfoo.so.*.*
```

- `PKG_CONFIG_LIBDIR` replaces pkg-config's default search path, so the build machine's own `.pc` files are never found. `PKG_CONFIG_SYSROOT_DIR` prefixes the paths in the `.pc` files with the staging tree.
- Here `cp -a` is right: it keeps the `libfoo.so.1 -> libfoo.so.1.2.3` link the loader looks up.
- **Delete libtool's `.la` files** in the staging tree. They hold absolute paths (`/usr/lib`) that send a later link to the build machine's own libraries.
- Run `needed-libs.sh` (section 5) on the new programs to check that nothing is missing from `$ROOTFS`.
- Drop what the board doesn't need: `usr/include`, `usr/share/man`, `usr/share/doc`, `usr/share/info`, `*.a`, `usr/lib/pkgconfig`.

**Example: an SSH server, Dropbear.** OpenSSH is the full server; Dropbear is a small one, often used on embedded systems. Built as a single binary with links (**untested**):

```sh
$ cd ~/bbb/src/dropbear-<version>
$ ./configure --host=arm-none-linux-gnueabihf --prefix=/usr --disable-zlib
$ make PROGRAMS="dropbear dbclient dropbearkey scp" MULTI=1 -j$(nproc)
$ make PROGRAMS="dropbear dbclient dropbearkey scp" MULTI=1 DESTDIR=$ROOTFS install
$ mkdir -p $ROOTFS/etc/dropbear
```

And on a `respawn` line in `/etc/inittab`, in the foreground (`-F`), creating host keys on first use (`-R`):

```
::respawn:/usr/sbin/dropbear -F -R
```

What it needs from the rest of the system: `/dev/pts` mounted (section 6), the user's shell in `/etc/shells` (section 8), and for password logins, a `crypt()`. The toolchain's glibc no longer has `libcrypt` (section 5). Depending on the version, `configure` then turns password logins off or the build fails. Either build `libxcrypt` into the staging tree first, or use key logins only, which is better anyway: put the public key in `/root/.ssh/authorized_keys`, mode `0600`, in a directory of mode `0700` owned by root. Dropbear refuses password logins for accounts with an empty password, so the development `root` of section 8 can only log in with a key.

**Where packages come from otherwise:** Buildroot and Yocto have recipes for thousands of packages with these steps already solved (section 13). A recipe from either one shows the configure options that work for cross builds of a given package, even when you build it by hand.

## 15. Read-only root and persistent data

A root filesystem mounted read-write gets written to all the time, by logs, temporary files and state. A product usually mounts it **read-only**:

- **Power cuts.** ext4's journal protects the filesystem's structure, not the content of files being written. A board unplugged at the wrong moment can come back with a truncated configuration file. A read-only root can't be damaged that way.
- **Flash wear.** SD cards and eMMC wear out with writes. A read-only root writes nothing.
- **A known state.** Every board runs exactly the image that was tested. That's the base for image updates (section 16) and integrity checking (section 17).

**What writes to the root filesystem**, and where it goes instead:

| Path | Written by | Read-only solution |
|---|---|---|
| `/tmp`, `/run`, `/var/run` | Everything | tmpfs (already, section 9) |
| `/var/log` | `syslogd` | `syslogd -C` ring buffer, a tmpfs, or `/data/log` to keep logs |
| `/var/lib`, `/var/cache`, `/var/spool` | Package-specific state | A link to `/data`, or a tmpfs when it may be lost |
| `/etc/resolv.conf` | The `udhcpc` script | A link to `/run/resolv.conf`, and the script writes there |
| `/etc/dropbear/` host keys | `dropbear -R` | Keys in `/data`; Dropbear's `-r` option names the key file |
| `/etc/shadow` | `passwd` | Fixed in the image, or an overlay on `/etc` |
| `/etc/machine-id` | systemd at first boot | systemd handles a read-only root itself; see its documentation |
| Application settings | The application | `/data` |

**A data partition.** Add a third partition to the card (ext4), mounted on `/data`, and change the command line from `rw` to `ro`:

```
# /etc/fstab, added line
/dev/mmcblk0p3   /data   ext4   defaults,noatime   0   2
```

For maintenance, `mount -o remount,rw /` makes the root writable until the next boot. BusyBox has no `e2fsck`; for the `pass` field `2` to mean anything, add e2fsprogs and check `/data` in `rcS` before `mount -a`. Applications should write a file safely: write a temporary file, `fsync` it, then `rename` it over the old one. The rename is atomic, so after a power cut the file is either the old version or the new one.

**An overlay on `/etc`.** overlayfs (`CONFIG_OVERLAY_FS`) shows a writable upper directory on top of a read-only lower one. Changes go to the upper directory, everything else is read from below. Early in `rcS`, after `/data` is mounted:

```sh
mkdir -p /data/etc.upper /data/etc.work
mount -t overlay overlay -o lowerdir=/etc,upperdir=/data/etc.upper,workdir=/data/etc.work /etc
```

The upper and work directories must be on the same filesystem. A change made on one board survives image updates, which is wanted for a network setting and unwanted for a file the new image changed. Decide which files may be changed, and reset the upper directory when the image changes.

**An overlay on the whole root**, from an initramfs that mounts it before `init` runs (**untested** sketch):

```sh
#!/bin/sh
# /init of a small initramfs: read-only root + writable overlay, then the real init
mount -t proc proc /proc
mount -t sysfs sysfs /sys
mount -t devtmpfs devtmpfs /dev
while [ ! -b /dev/mmcblk0p2 ]; do sleep 1; done          # what rootwait does
mkdir -p /ro /rw /newroot
mount -o ro /dev/mmcblk0p2 /ro
mount /dev/mmcblk0p3 /rw              # or: mount -t tmpfs tmpfs /rw, to discard changes at reboot
mkdir -p /rw/upper /rw/work
mount -t overlay overlay -o lowerdir=/ro,upperdir=/rw/upper,workdir=/rw/work /newroot
mkdir -p /newroot/ro /newroot/rw
mount --move /ro /newroot/ro
mount --move /rw /newroot/rw
mount --move /dev /newroot/dev
umount /proc /sys
exec switch_root /newroot /sbin/init
```

`switch_root` deletes the initramfs's files to free the RAM, makes `/newroot` the root and executes `init` there as PID 1. The tmpfs variant is popular for kiosks and appliances: every boot starts from the image, whatever happened before.

## 16. Updates

| Strategy | How | For | Against |
|---|---|---|---|
| Packages (`apt`, `opkg`) | Replace packages one by one | Small downloads | Every board ends up in a slightly different state; a power cut mid-update can leave a broken system |
| Image, A/B ("symmetric") | Two root filesystem slots: write the inactive one, switch, fall back if the new one doesn't boot | Atomic, with automatic rollback | Twice the space |
| Image, recovery ("asymmetric") | A small recovery system writes the main one | Less space | No service and no fallback during the update |
| File-based (OSTree) | A versioned tree of files, deployed with hard links | Small downloads, several versions kept | More complex; constraints on the layout |
| Containers for the application | A fixed base system, applications updated as container images | Applications updated independently | A container runtime, storage, and still a base system to update |

**A/B on this board.** The card gets partition 1 (FAT: `MLO`, `u-boot.img`, the U-Boot environment), partitions 2 and 3 (root filesystems A and B), and partition 4 (`/data`, section 15). Each root filesystem holds its own kernel and DTB in `/boot`, so a slot always has a kernel and modules from the same build (kernel note, section 8). The bootloader chooses the slot and counts failed boots (bootloader note, boot counter and A/B):

1. The update tool writes the new image to the inactive slot and checks it.
2. It tells the bootloader to try that slot (`fw_setenv`), with a limited number of attempts.
3. The board reboots into the new slot. If the system comes up healthy, a service confirms the slot and resets the counter.
4. If it doesn't (panic, `init` failing, the watchdog firing), the counter runs out and the bootloader goes back to the old slot.

"Healthy" is defined by the product: the application started, the network came up, the server answered. Confirming too early defeats the rollback.

**Tools.** Writing the slot safely, verifying signatures, talking to the bootloader and handling power cuts at every step is what these tools do. Don't rewrite them in scripts:

| Tool | Model | Notes |
|---|---|---|
| SWUpdate | Images or files, described by `sw-description` | Very flexible; signed and encrypted updates; Buildroot and Yocto support it |
| RAUC | Signed bundles (X.509), A/B slots described in `system.conf` | Integrates with U-Boot and systemd; delta downloads with adaptive updates |
| Mender | Client plus a server, A/B | Open-source client and server, with a hosted service |
| OSTree | File-based, content-addressed | Used by Fedora's immutable editions and some embedded distributions |

**Update images are signed**, and the board verifies the signature before writing anything. An unsigned update is a way for anyone on the network to replace the system.

## 17. Security

| Area | Development system | Product |
|---|---|---|
| Root password | Empty | Locked (`*` or `!` in `/etc/shadow`), or a strong hash; never the same hash on every board if it's ever used |
| Serial console | `respawn` shell, no login | A login, or no `getty` at all; a console that can be enabled only by a signed debug image |
| SSH | Anything | Keys only, no passwords, no root login with a password |
| SSH host keys | Anything | **Generated on each board at first boot**, never stored in the image: otherwise every board has the same key, and one leaked image lets anyone impersonate them all |
| Per-device secrets | | Never in the image: provisioned per board, ideally in a secure element or the SoC's key storage |
| Services | Whatever BusyBox `defconfig` has | Only what the product uses: no `telnetd`, `httpd`, `ftpd`, `tftpd` in the BusyBox configuration |
| Users | Everything as root | The application as its own user. Grant only the capabilities it needs (`CAP_NET_BIND_SERVICE`...), or start it from systemd with `User=` and `AmbientCapabilities=` |
| Mount options | Defaults | `nosuid,nodev` on `/tmp`, `/run`, `/data`; `noexec` where no programs should run |
| Root filesystem | Read-write | Read-only (section 15), checked by dm-verity |
| Debug tools | `gdbserver`, `strace` | Removed |

**dm-verity** checks every block of a read-only root filesystem as it's read, against a hash tree whose top hash ("root hash") is trusted. A modified block produces an I/O error instead of modified data. With the tools of cryptsetup (**untested**):

```sh
$ veritysetup format rootfs.squashfs rootfs.hashtree
...
Root hash:      <64 hexadecimal digits>
```

The hash tree goes on its own partition (here partition 3) next to the image. Without an initramfs, the kernel sets up the mapping from its command line (`CONFIG_DM_INIT`, `CONFIG_DM_VERITY` built in; `Documentation/admin-guide/device-mapper/dm-init.rst`):

```
dm-mod.create="vroot,,,ro,0 <sectors> verity 1 /dev/mmcblk0p2 /dev/mmcblk0p3 4096 4096 <blocks> 1 sha256 <root hash> <salt>" root=/dev/dm-0 rootfstype=squashfs ro
```

`<sectors>` is the image size in 512-byte sectors, `<blocks>` in 4096-byte blocks, and the `1` before `sha256` skips the superblock `veritysetup` wrote at the start of the hash device. The root hash and salt come from the `format` output. dm-verity is only as trustworthy as the command line that carries the root hash, so the command line must itself be verified: in a signed FIT image with the kernel, checked by a verified bootloader (bootloader note, verified boot and its limits on this board).

**Licenses.** A root filesystem is a collection of other people's software under their licenses. BusyBox, the kernel and many tools are GPLv2: shipping them means providing their source, your configurations and your modifications. glibc is LGPL: programs linking to it dynamically are fine, static linking brings obligations. Keep the list of what's in the image and under which license. Buildroot (`make legal-info`) and Yocto (license manifests, SPDX) produce it.

**Vulnerabilities.** Every component gets CVEs over time; BusyBox has regular ones in `ash`, `httpd` and other applets. Keep the exact versions of what's in the image, watch their advisories, and plan the update path (section 16) before the first vulnerability, not after. Buildroot (`make pkg-stats`) and Yocto (`cve-check`) list the known CVEs for the packages they build.

## 18. Size

**Where the space goes** (**Typical**, for this kind of system):

| Item | Size | Reduce by |
|---|---|---|
| Kernel modules: all of `omap2plus_defconfig` | By far the largest part | Install only the modules this board loads (below), or a board-specific kernel configuration (kernel note, section 21) |
| glibc | About 2 MiB | Copy only the libraries used (section 5); musl for under 1 MiB |
| BusyBox, `defconfig` | About 1 MiB | Turn off unused applets; a few hundred KiB for a small set |
| Your programs and libraries | Varies | `-Os`, `strip`, `-ffunction-sections -fdata-sections -Wl,--gc-sections`, LTO |
| Headers, static libraries, man pages, documentation | Often several MiB after adding packages | Don't copy them (section 14) |
| Debug symbols | Several times the code | `strip --strip-unneeded`; keep unstripped copies on the build machine |

**Only the modules the board uses.** On a booted development board, `lsmod` lists the loaded modules. On the build machine, kmod's `modprobe` resolves their dependencies against the staging tree, so only those files are copied, then `depmod` rebuilds the indexes:

```sh
$ for m in $(cat lsmod-from-board.txt); do
      modprobe -d ~/bbb/staging-modules -S $KREL --show-depends $m
  done | awk '/^insmod/ {print $2}' | sort -u > modules.list
$ # copy the files in modules.list, plus modules.order and modules.builtin*, into $ROOTFS/lib/modules/$KREL/
$ depmod -b $ROOTFS $KREL
```

`~/bbb/staging-modules` is a full `modules_install` kept on the build machine (section 11, with `INSTALL_MOD_PATH=~/bbb/staging-modules`); `modprobe -d` expects `lib/modules/<release>/` under it. Devices plugged in later (USB) need their modules too: add them to the list.

**Measuring:**

```sh
$ du -sh $ROOTFS/*                                  # per top-level directory
$ du -a $ROOTFS | sort -n | tail -20                # the 20 largest files and directories
$ ls -l ~/bbb/images/                               # the images themselves
```

**Compression** is a property of the filesystem: squashfs and EROFS are compressed, ext4 isn't. A read-only squashfs root is often a third to half the size of the same tree in ext4, at some CPU cost when reading. On the board, `free` and `df` give the real numbers, including tmpfs contents, which live in RAM.

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
