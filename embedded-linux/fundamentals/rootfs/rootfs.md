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
