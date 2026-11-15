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

Buildroot is a set of Makefiles and Kconfig menus. You choose what you want in a menu, the choices are saved in one `.config` file, and `make` builds everything they name, in dependency order:

```
 .config (Kconfig choices)                                 output/images/
 ┌───────────────────────────────┐                  ┌───────────────────────┐
 │ target CPU      BR2_cortex_a8 │                  │ MLO, u-boot.img       │
 │ toolchain       internal/ext. │                  │ zImage, *.dtb         │
 │ system          init, getty   │──> make ──────>  │ rootfs.ext4, .tar     │
 │ packages        BR2_PACKAGE_* │                  │ sdcard.img            │
 │ kernel, U-Boot  versions, cfg │                  └───────────────────────┘
 │ images          ext4, genimage│                  output/host/: toolchain, SDK
 └───────────────────────────────┘                  output/legal-info/: licenses
               ▲
               │ package/*/*.mk: how to download and build each piece
```

- **Kconfig**, the same menu system as the kernel's (kernel note, section 5), holds the choices. A **defconfig** is the short form of a `.config`: only the lines that differ from the defaults.
- **Packages** are directories under `package/`, each with a `Config.in` (the menu entry) and a `.mk` file (how to download, configure, build and install it). There are about 2900 of them.
- **`make`** walks the packages that are switched on, builds each one once, and leaves a stamp file so it isn't built again.

**What Buildroot is not:**
- **Not a distribution.** There's no package manager on the board and no package files: the output is a root filesystem image. To change the board's software, you rebuild the image.
- **Not incremental in general.** `make` knows when a package hasn't been built yet, but not when a change in the configuration should rebuild something already built. Section 14 explains when a full rebuild is needed.
- **Not built from the build machine's libraries.** Buildroot builds its own host tools when the build machine's are missing or too old, and its own cross-compiler unless you give it one.

**Buildroot or Yocto:** the [Yocto note](../yocto/yocto_v1_2026-11-14.md) (section 1) has the comparison table. In short, Buildroot fits one board and one firmware image, with a configuration you can read in one screen and a first build in tens of minutes; Yocto fits a family of products with packages, layers and shared state, at the price of weeks of learning.

## 2. The vocabulary

| Term | Meaning | File or option |
|---|---|---|
| Configuration | Every choice of this build, in Kconfig form | `.config`, written by `make menuconfig` |
| Defconfig | A minimal configuration: only what differs from the defaults | `configs/beaglebone_defconfig`, written by `make savedefconfig` |
| Option | One Kconfig symbol, all named `BR2_...` | `BR2_TARGET_GENERIC_HOSTNAME="bbb"` |
| Package | One piece of software: a `Config.in` and a `<name>.mk` | `package/busybox/` |
| Package infrastructure | Build logic shared by packages (`generic-package`, `autotools-package`, `cmake-package`, `meson-package`, `kernel-module`) | `package/pkg-*.mk`, used with `$(eval $(cmake-package))` |
| Host package | A tool built to run on the build machine | `host-genimage`, `host-pkgconf` |
| Target package | Software built for the board | `busybox`, `dropbear` |
| Toolchain | The cross-compiler and C library; built by Buildroot (internal) or prebuilt (external) | `Toolchain` menu, `BR2_TOOLCHAIN_*` |
| Staging | The sysroot: the target's headers and libraries, for compiling | `output/staging/` |
| Target | The root filesystem being assembled, without headers or static libraries | `output/target/` |
| Rootfs overlay | A directory of files copied over the target at the end | `BR2_ROOTFS_OVERLAY` |
| Post-build, post-image scripts | Your scripts, run before and after the images are made | `BR2_ROOTFS_POST_BUILD_SCRIPT`, `BR2_ROOTFS_POST_IMAGE_SCRIPT` |
| br2-external | A directory outside Buildroot with your defconfigs, board files and packages | `BR2_EXTERNAL=../bbb-external` |
| Hash file | Checksums for a package's downloads and license files | `package/busybox/busybox.hash` |

**The Buildroot tree:**

| Directory | Contents |
|---|---|
| `arch/` | CPU choices: architecture, variant, floating point, ABI |
| `board/` | Board files: scripts, `genimage.cfg`, patches, readme, per board |
| `boot/` | Bootloaders: U-Boot, Barebox, GRUB, TF-A, OP-TEE |
| `configs/` | The defconfigs: `make list-defconfigs` lists them |
| `fs/` | Filesystem image types: ext4, squashfs, UBIFS, cpio, EROFS |
| `linux/` | The kernel package and its extensions |
| `package/` | Every other package |
| `support/` | Internal scripts, including `support/scripts/genimage.sh` |
| `system/` | The skeleton of the root filesystem, the default device and permission tables, the init choice |
| `toolchain/` | Internal and external toolchain support |
| `utils/` | Developer tools: `brmake`, `check-package`, `test-pkg`, `size-stats-compare` |
| `docs/manual/` | The manual, the reference for everything here |
| `CHANGES` | What changed in each release; read it before upgrading |

**Three places for a setting.** Most questions about "where does this go?" have one answer:

| Question | Answered by | Example |
|---|---|---|
| Which hardware and which software? | The defconfig | `BR2_cortex_a8=y`, `BR2_PACKAGE_DROPBEAR=y` |
| Which files, users and layout on this board? | Your board directory: overlay, tables, scripts, `genimage.cfg` | `board/bbb/rootfs-overlay/etc/...` |
| How is one piece of software built? | Its package (`.mk`), or a fragment for Kconfig packages | `hello.mk`, `linux.fragment` |

Never edit Buildroot's own files for a product: your changes would be mixed with upstream's and lost at the next upgrade. Section 9 puts all three places in your own tree.

## 3. The build machine

**Distributions.** Buildroot runs on any recent Linux distribution; it doesn't keep a list of supported ones. It builds the host tools it can't find, so the list of required packages is short.

**Packages**, on Debian and Ubuntu (the mandatory list of the manual, plus the optional ones used in this note):

```sh
$ sudo apt-get install build-essential bc bzip2 cpio diffutils file findutils gzip \
      patch perl rsync sed tar unzip wget which
$ sudo apt-get install git libncurses-dev python3 graphviz python3-matplotlib
```

| Package | Needed for |
|---|---|
| `build-essential` (`gcc`, `g++`, `make`) | Building anything. Minimum versions: GNU make 3.81 and GCC 4.8 (any current distribution has more) |
| `wget`, `rsync`, `unzip` | Downloading and copying source |
| `libncurses-dev` | `make menuconfig` |
| `git` | Packages fetched from git, and keeping your own tree |
| `python3` | Some packages, `utils/` scripts, `make pkg-stats` |
| `graphviz`, `python3-matplotlib` | `make graph-depends`, `make graph-build`, `make graph-size` (section 17) |

**Resources:**

| Resource | Figure | Note |
|---|---|---|
| Disk | **Typical**: 5 to 15 GB for a small image with an internal toolchain | An SSD helps more than anything else |
| RAM | **Typical**: 4 GB works; 8 GB or more for C++ packages built in parallel | Running out of memory kills the compiler (section 20) |
| Time | **Typical**: 20 to 60 minutes for a first small image on 4 to 8 cores; less with an external toolchain | Downloads are a large share of the first build |
| Network | Several hundred MB to a few GB of source on the first build | Then kept in `dl/` (section 6) |

**Don't build as root.** Buildroot doesn't need root rights: the files of the image get their owners and permissions from tables (section 10) through `fakeroot`, the same tool the rootfs note uses (section 8). A build as root can overwrite files of the build machine if a package's install step is wrong.

**Check `PATH`.** Buildroot refuses to start when `PATH` contains the current directory (an empty entry, a `.`, or a trailing `:`) or spaces. Some shell profiles add them.

## 4. Getting Buildroot

Buildroot is used from its source tree: there is nothing to install. Two ways to get it:

```sh
$ git clone https://gitlab.com/buildroot.org/buildroot.git ~/buildroot
$ cd ~/buildroot
$ git tag -l '2025.02*'                  # the LTS and its point releases
$ git switch --detach 2025.02.<N>        # the latest point release
```

Or the release tarball from https://buildroot.org/downloads/ (`buildroot-2025.02.<N>.tar.xz`, with a `.sign` file to check its signature).

| Choice | Use |
|---|---|
| A tag of the LTS branch (`2025.02.<N>`) | Products. The point releases bring security fixes and package updates without changing the configuration |
| A tag of a regular release (`2026.05`) | Newer package versions, three months of maintenance |
| The `master` branch | Development of Buildroot itself; it changes every day |

**Keep Buildroot and your work apart.** Your configuration and files go in a separate tree (section 9). Buildroot itself stays unmodified, at a tag. Many projects keep both in one repository with Buildroot as a git submodule, or record the Buildroot version in a script that clones it; either way, the Buildroot version is part of what defines the product.

**Out-of-tree output.** By default everything is built in `output/` inside the Buildroot tree. `O=` puts it elsewhere, which lets one Buildroot tree build several configurations:

```sh
$ make O=$HOME/br-out/qemu qemu_arm_vexpress_defconfig
$ cd $HOME/br-out/qemu
$ make
```

`make` with `O=` writes a small `Makefile` in the output directory, so later commands run from there without `O=`. The rest of this note writes `output/` for the output directory.

## 5. Building and running an image in QEMU

**Choose a defconfig.** The list is long; the QEMU ones start with `qemu_`:

```sh
$ make list-defconfigs | grep qemu_arm
$ make qemu_arm_vexpress_defconfig
```

`qemu_arm_vexpress_defconfig` emulates an Arm Versatile Express board with a Cortex-A9: the closest emulated relative of the BeagleBone Black's Cortex-A8, with the same `arm-linux-gnueabihf` kind of binaries. `qemu_aarch64_virt_defconfig` is the 64-bit equivalent.

**Build:**

```sh
$ make
```

Don't add `-j`: Buildroot builds one package at a time and runs each package's own build in parallel (`BR2_JLEVEL`, which defaults to the number of CPUs plus one). Building several packages at once needs per-package directories (section 14).

**Expected**, the start and the end of the log:

```
>>> host-skeleton  Installing to host directory
>>> host-pkgconf <version> Downloading
>>> host-pkgconf <version> Extracting
>>> host-pkgconf <version> Patching
>>> host-pkgconf <version> Configuring
>>> host-pkgconf <version> Building
>>> host-pkgconf <version> Installing to host directory
...
>>> busybox <version> Building
...
>>>   Generating root filesystems common tables
>>>   Generating filesystem image rootfs.ext2
>>>   Executing post-image script board/qemu/post-image.sh
```

Every package goes through the same steps, and each `>>>` line is one of them: that's how to find where a build stopped (section 17).

**Run it:**

```sh
$ sudo apt-get install qemu-system-arm
$ output/images/start-qemu.sh serial-only
```

`start-qemu.sh` is written by the QEMU defconfigs' post-image script with the right `qemu-system-arm` command for the board. `serial-only` puts the console in the terminal (leave with `Ctrl-A x`). Without the script, the command is in `board/qemu/arm-vexpress/readme.txt` (**From memory**):

```sh
$ qemu-system-arm -M vexpress-a9 -smp 1 -m 256 \
      -kernel output/images/zImage -dtb output/images/vexpress-v2p-ca9.dtb \
      -drive file=output/images/rootfs.ext2,if=sd,format=raw \
      -append "console=ttyAMA0,115200 rootwait root=/dev/mmcblk0" \
      -net nic,model=lan9118 -net user -nographic
```

**Expected**, at the end of the boot:

```
Welcome to Buildroot
buildroot login: root
#
```

`root` has no password in the reference defconfigs. That's for development only (section 18).

**The `make` targets used most:**

| Command | What it does |
|---|---|
| `make <name>_defconfig` | Loads a defconfig into `.config` |
| `make menuconfig` | Changes the configuration (section 8) |
| `make` | Builds everything that isn't built yet |
| `make savedefconfig` | Writes the configuration back as a defconfig (section 8) |
| `make <pkg>` | Builds one package and what it depends on |
| `make <pkg>-rebuild`, `<pkg>-reconfigure`, `<pkg>-dirclean` | Rebuilds one package from the build step, the configure step, or from scratch (section 14) |
| `make clean` | Deletes the output directory, keeps `.config` and downloads |
| `make distclean` | Also deletes `.config`, and `dl/` when it's inside the tree |
| `make source` | Downloads everything without building |
| `make help` | Lists the targets |

## 6. The output directory: where everything goes

After a build:

```
output/
├── build/                        one directory per package: <name>-<version>/
├── host/                         everything that runs on the build machine
│   ├── bin/                      the cross-compiler, host tools
│   └── arm-buildroot-linux-gnueabihf/sysroot/    the sysroot
├── staging -> host/<tuple>/sysroot/     link: headers and libraries, for compiling
├── target/                       the root filesystem being assembled
├── images/                       what goes on the board
├── graphs/                       make graph-* output (section 17)
└── legal-info/                   make legal-info output (section 18)
```

| Directory | Contents | Use |
|---|---|---|
| `build/<pkg>-<version>/` | The unpacked, patched source, built in place, with the stamp files | Where to look when a package fails (section 17) |
| `host/` | The toolchain and host tools; also the base of the SDK (section 16) | `output/host/bin/arm-buildroot-linux-gnueabihf-gcc` |
| `staging/` | The sysroot: libraries and headers of every package installed with `INSTALL_STAGING` | The "staging tree" of the rootfs note (section 14), made automatically |
| `target/` | The root filesystem without headers, static libraries or documentation, and with stripped binaries | Look, don't edit: it's not the image (see below) |
| `images/` | Kernel, DTBs, bootloader, root filesystem images, the card image | The deliverables |

**`target/` is not the root filesystem image.** Its files belong to your user, and device nodes and setuid bits are missing. The real ownership and permissions are applied in a `fakeroot` session when the images are made, from the tables of section 10. Files you copy into `target/` by hand survive until the next `make clean` and make the build impossible to reproduce: use an overlay or a post-build script.

**The stamp files** in each `build/<pkg>-<version>/` record which steps are done:

| Stamp | Step |
|---|---|
| `.stamp_downloaded` | Source in `dl/` with a valid hash |
| `.stamp_extracted`, `.stamp_patched` | Unpacked, patches applied |
| `.stamp_configured` | Configure step done |
| `.stamp_built` | Build step done |
| `.stamp_staging_installed`, `.stamp_target_installed`, `.stamp_images_installed`, `.stamp_host_installed` | Installed where the package goes |

`make` runs a step only when its stamp is missing. Deleting a stamp by hand reruns that step and the ones after it; the `<pkg>-rebuild` targets (section 14) do it for you.

**Downloads** go to `dl/<pkg>/` in the Buildroot tree, outside `output/`. To share them between Buildroot trees and keep them across `make distclean`, point every build to one directory, in the shell profile:

```sh
export BR2_DL_DIR=$HOME/br-dl
```

**What to keep and what to delete:**

| Directory | Deleting it |
|---|---|
| `output/` | Safe: `make` rebuilds everything. This is what `make clean` does |
| `dl/` or `BR2_DL_DIR` | Safe, but everything is downloaded again. Worth backing up for a product (section 18) |
| `.config` | Your configuration, unless it's saved as a defconfig. Save first (section 8) |

## 7. The BeagleBone Black: beaglebone_defconfig

Buildroot has a defconfig for the board, with its files in `board/beaglebone/`:

```sh
$ cat configs/beaglebone_defconfig
$ ls board/beaglebone/
$ cat board/beaglebone/readme.txt
$ make O=$HOME/br-out/bbb beaglebone_defconfig
$ make -C $HOME/br-out/bbb
```

**What the defconfig contains.** The shape of it (**From memory** of earlier releases; versions change, read your copy):

```
BR2_arm=y
BR2_cortex_a8=y
BR2_TARGET_GENERIC_GETTY_PORT="ttyS0"
BR2_ROOTFS_POST_IMAGE_SCRIPT="support/scripts/genimage.sh"
BR2_ROOTFS_POST_SCRIPT_ARGS="-c board/beaglebone/genimage.cfg"
BR2_LINUX_KERNEL=y
BR2_LINUX_KERNEL_CUSTOM_VERSION=y
BR2_LINUX_KERNEL_CUSTOM_VERSION_VALUE="<version>"
BR2_LINUX_KERNEL_DEFCONFIG="omap2plus"
BR2_LINUX_KERNEL_DTS_SUPPORT=y
BR2_LINUX_KERNEL_INTREE_DTS_NAME="<am335x-bone, am335x-boneblack and others>"
BR2_TARGET_ROOTFS_EXT2=y
BR2_TARGET_ROOTFS_EXT2_4=y
BR2_TARGET_UBOOT=y
BR2_TARGET_UBOOT_BUILD_SYSTEM_KCONFIG=y
BR2_TARGET_UBOOT_BOARD_DEFCONFIG="am335x_evm"
BR2_TARGET_UBOOT_FORMAT_IMG=y
BR2_TARGET_UBOOT_SPL=y
BR2_TARGET_UBOOT_SPL_NAME="MLO"
BR2_PACKAGE_HOST_GENIMAGE=y
BR2_PACKAGE_HOST_DOSFSTOOLS=y
BR2_PACKAGE_HOST_MTOOLS=y
```

Every line is something done by hand in the fundamentals notes:

| Option | By hand | Note |
|---|---|---|
| `BR2_arm`, `BR2_cortex_a8` (with the default NEON and hard-float choices) | `-mcpu=cortex-a8 -mfpu=neon -mfloat-abi=hard` | Toolchain note |
| `BR2_LINUX_KERNEL_DEFCONFIG`, `BR2_LINUX_KERNEL_INTREE_DTS_NAME` | `make omap2plus_defconfig`, `make zImage dtbs` | Kernel note |
| `BR2_TARGET_UBOOT_BOARD_DEFCONFIG`, `BR2_TARGET_UBOOT_SPL_NAME` | `make am335x_evm_defconfig`, `MLO` and `u-boot.img` | Bootloader note |
| `BR2_TARGET_GENERIC_GETTY_PORT` | The `getty` line of `inittab` | Rootfs note, section 7 |
| `BR2_TARGET_ROOTFS_EXT2_4` | `mkfs.ext4 -d` | Rootfs note, section 10 |
| `BR2_ROOTFS_POST_IMAGE_SCRIPT` with `genimage.cfg` | The card layout and `genimage` | Rootfs note, section 11 |

Lines not in the defconfig take their default: an internal toolchain (Buildroot builds the cross-compiler, with the default C library), BusyBox init, BusyBox, `devtmpfs` for `/dev`. `make menuconfig` shows them all.

**The output**, in `output/images/` (**Expected**):

| File | What it is |
|---|---|
| `MLO`, `u-boot.img` | The two stages of U-Boot |
| `zImage`, `am335x-boneblack.dtb` and the other DTBs | Kernel and device trees |
| `boot.vfat` | The FAT boot partition, made by `genimage` |
| `rootfs.ext4` (and `rootfs.ext2`, a link) | The root partition |
| `rootfs.tar` | The root filesystem as an archive, for NFS root (on by default) |
| `sdcard.img` | The whole SD card: partition table, boot partition, root partition |

The boot partition also holds the file U-Boot reads to start the kernel: `uEnv.txt` or `extlinux/extlinux.conf`, depending on the release. `board/beaglebone/genimage.cfg` lists what goes in it. The bootloader note (section 8) explains both; mainline U-Boot doesn't read `uenvcmd` from a `uEnv.txt`.

**Writing the card.** Check the device name first (`lsblk`): the command erases it.

```sh
$ sudo dd if=output/images/sdcard.img of=/dev/sdX bs=4M conv=fsync status=progress
```

Or `sudo bmaptool copy --nobmap output/images/sdcard.img /dev/sdX`. Hold the S2 button while powering the board to boot from the SD card instead of the eMMC (bootloader note).

**Expected**, on the serial console:

```
U-Boot SPL <version>
...
U-Boot <version>
...
Starting kernel ...
[    0.000000] Booting Linux on physical CPU 0x0
...
Welcome to Buildroot
buildroot login:
```

**For other boards** the same steps apply with another defconfig: `raspberrypi4_64_defconfig` for a Raspberry Pi 4 in 64-bit mode, and defconfigs for most boards of NXP, ST, Microchip, Rockchip and others. Read the board's `readme.txt` first. A vendor's own Buildroot tree (a "BSP") is usually an older Buildroot with extra packages; prefer the upstream defconfig when it exists.

## 8. Configuration: menuconfig, defconfigs and the toolchain

**Changing the configuration:**

```sh
$ make menuconfig            # or nconfig (ncurses), xconfig (Qt), gconfig (GTK)
```

The menus, in order:

| Menu | What it holds |
|---|---|
| Target options | Architecture, CPU, floating point, ABI |
| Toolchain | Internal or external toolchain, C library, GCC version, C++, kernel headers |
| Build options | Download directory, mirrors, parallelism, `ccache`, hardening flags, reproducibility |
| System configuration | Hostname, init system, `/dev` management, root password, getty, overlays, scripts, tables |
| Kernel | The Linux package: version, configuration, device trees |
| Target packages | Every package, in categories |
| Filesystem images | ext4, squashfs, UBIFS, cpio, tar, EROFS... |
| Bootloaders | U-Boot and others |
| Host utilities | Host tools to build, such as `genimage` |
| Legacy config options | Options removed in this release that your configuration still sets (section 19) |

In `menuconfig`, `/` searches for an option and shows its location, its dependencies and what selects it. That's the answer to "why can't I see package X?": usually it depends on a toolchain feature that is off (C++, threads, `wchar`, a C library).

**`select` and `depends on`.** A package that needs a library *selects* it, so switching on the package switches on the library. A package that needs a toolchain feature *depends on* it, because Buildroot can't change the toolchain behind your back: the package stays hidden until the feature is on.

**Saving the configuration.** `.config` is long and full of defaults. Keep the defconfig instead:

```sh
$ make savedefconfig
```

`savedefconfig` writes to the file named by `BR2_DEFCONFIG`, which is set to the defconfig you loaded: after `make beaglebone_defconfig`, it overwrites `configs/beaglebone_defconfig` in the Buildroot tree. To save elsewhere: `make savedefconfig BR2_DEFCONFIG=<path>`. Section 9 makes your own defconfig the one that gets written.

**Kconfig packages** (Linux, U-Boot, BusyBox, uClibc-ng, Barebox) have their own configuration, with their own targets:

| Command | What it does |
|---|---|
| `make linux-menuconfig` | Opens the kernel's `menuconfig` on the kernel's `.config` in `output/build/linux-<version>/` |
| `make linux-update-defconfig` | Writes the kernel configuration to `BR2_LINUX_KERNEL_CUSTOM_CONFIG_FILE` (needs a custom configuration file, section 12) |
| `make linux-diff-config` | Prints the changes from the base configuration: the start of a fragment |
| `make uboot-menuconfig`, `uboot-update-defconfig` | The same for U-Boot |
| `make busybox-menuconfig`, `busybox-update-config` | The same for BusyBox |

Changes made with `linux-menuconfig` alone are lost at the next `make clean`. Save them as a fragment or a configuration file (section 12).

**The toolchain** is the first big decision, because changing it later rebuilds everything:

| Choice | Option | Pros | Cons |
|---|---|---|---|
| Internal | `BR2_TOOLCHAIN_BUILDROOT` | Any C library (glibc, musl, uClibc-ng), any GCC version Buildroot offers, built to match | Adds **typically** 15 to 30 minutes to every full build |
| External, known | `BR2_TOOLCHAIN_EXTERNAL_BOOTLIN`, `BR2_TOOLCHAIN_EXTERNAL_ARM_ARM` | Downloaded prebuilt, no build time | Its versions are fixed by the toolchain release |
| External, custom | `BR2_TOOLCHAIN_EXTERNAL_CUSTOM` | Any toolchain: a vendor's, your company's, the Arm GNU Toolchain of the toolchain note | You must declare its GCC version, kernel headers version, C library and features; Buildroot checks them |

The Bootlin toolchains (https://toolchains.bootlin.com) are themselves built with Buildroot, in glibc, musl and uClibc-ng versions, "stable" or "bleeding edge", for most architectures; Buildroot downloads the one that matches the target options. For a product, an external toolchain shortens every full build, and the toolchain stays the same when Buildroot's internal one moves to a new GCC.

**Kernel headers** in the toolchain must be the same as or older than the kernel that runs on the board (toolchain note, section 11). With an internal toolchain, set them to the kernel's version (`BR2_KERNEL_HEADERS_AS_KERNEL`, or the matching `BR2_PACKAGE_HOST_LINUX_HEADERS_CUSTOM_<x>_<y>`).

## 9. Your own tree: br2-external

A **br2-external tree** is a directory outside Buildroot that Buildroot reads as if it were part of it: defconfigs, board files and packages. Everything of yours goes there, in your own git repository. This note calls it `bbb-external`:

```
bbb-external/
├── external.desc                        name and description
├── external.mk                          includes the packages' .mk files
├── Config.in                            includes the packages' Config.in files
├── configs/
│   └── bbb_defconfig                    this section
├── board/bbb/
│   ├── rootfs-overlay/                  section 10
│   ├── users.txt                        section 10
│   ├── post-build.sh                    section 10
│   ├── linux.fragment                   section 12
│   ├── busybox.fragment                 section 12
│   ├── extlinux.conf                    section 13
│   ├── genimage.cfg                     section 13
│   └── post-image.sh                    section 13
├── patches/
│   └── linux/0001-<change>.patch        section 12
└── package/hello/
    ├── Config.in                        section 11
    ├── hello.mk
    ├── S99hello
    └── src/
```

**The three files that make it a tree:**

`external.desc`:

```
name: BBB
desc: BeagleBone Black product
```

`external.mk`:

```
include $(sort $(wildcard $(BR2_EXTERNAL_BBB_PATH)/package/*/*.mk))
```

`Config.in`:

```
source "$BR2_EXTERNAL_BBB_PATH/package/hello/Config.in"
```

| Element | Meaning |
|---|---|
| `name: BBB` | The tree's name. Buildroot defines `BR2_EXTERNAL_BBB_PATH`, the tree's absolute path, for use in options, `.mk` files and `Config.in` |
| `external.mk` | Read by `make`. The `wildcard` line picks up every package without listing them |
| `Config.in` | Read by Kconfig. Its entries appear under "External options" in `menuconfig`. Each package must be listed by hand |

**Using it.** The tree is named once; Buildroot remembers it in the output directory:

```sh
$ cd ~/buildroot
$ make BR2_EXTERNAL=$HOME/bbb-external O=$HOME/br-out/bbb bbb_defconfig
$ cd $HOME/br-out/bbb
$ make menuconfig
$ make savedefconfig          # writes back to bbb-external/configs/bbb_defconfig
$ make
```

`make list-defconfigs` now lists the tree's defconfigs too. Several trees can be combined, separated with `:`, each with its own name.

**Starting the defconfig.** Copy the board's defconfig, then move the board files to your tree:

```sh
$ cp configs/beaglebone_defconfig ~/bbb-external/configs/bbb_defconfig
$ cp board/beaglebone/genimage.cfg ~/bbb-external/board/bbb/
```

In the defconfig, refer to your files through the tree's path variable, never with an absolute path, so the tree works wherever it is checked out:

```
BR2_ROOTFS_OVERLAY="$(BR2_EXTERNAL_BBB_PATH)/board/bbb/rootfs-overlay"
```

**The defconfig of this note** (**Illustrative**; the version numbers are placeholders and each line is explained in its section):

```
BR2_arm=y
BR2_cortex_a8=y
BR2_TOOLCHAIN_EXTERNAL=y
BR2_TOOLCHAIN_EXTERNAL_BOOTLIN=y
BR2_GLOBAL_PATCH_DIR="$(BR2_EXTERNAL_BBB_PATH)/patches"
BR2_TARGET_GENERIC_HOSTNAME="bbb"
BR2_TARGET_GENERIC_ISSUE="BBB Linux"
BR2_TARGET_GENERIC_GETTY_PORT="ttyS0"
BR2_SYSTEM_DHCP="eth0"
BR2_ROOTFS_OVERLAY="$(BR2_EXTERNAL_BBB_PATH)/board/bbb/rootfs-overlay"
BR2_ROOTFS_USERS_TABLES="$(BR2_EXTERNAL_BBB_PATH)/board/bbb/users.txt"
BR2_ROOTFS_POST_BUILD_SCRIPT="$(BR2_EXTERNAL_BBB_PATH)/board/bbb/post-build.sh"
BR2_ROOTFS_POST_IMAGE_SCRIPT="$(BR2_EXTERNAL_BBB_PATH)/board/bbb/post-image.sh"
BR2_LINUX_KERNEL=y
BR2_LINUX_KERNEL_CUSTOM_VERSION=y
BR2_LINUX_KERNEL_CUSTOM_VERSION_VALUE="6.12.<N>"
BR2_LINUX_KERNEL_DEFCONFIG="omap2plus"
BR2_LINUX_KERNEL_CONFIG_FRAGMENT_FILES="$(BR2_EXTERNAL_BBB_PATH)/board/bbb/linux.fragment"
BR2_LINUX_KERNEL_DTS_SUPPORT=y
BR2_LINUX_KERNEL_INTREE_DTS_NAME="ti/omap/am335x-boneblack"
BR2_LINUX_KERNEL_NEEDS_HOST_OPENSSL=y
BR2_PACKAGE_BUSYBOX_CONFIG_FRAGMENT_FILES="$(BR2_EXTERNAL_BBB_PATH)/board/bbb/busybox.fragment"
BR2_PACKAGE_DROPBEAR=y
BR2_PACKAGE_HELLO=y
BR2_TARGET_ROOTFS_EXT2=y
BR2_TARGET_ROOTFS_EXT2_4=y
BR2_TARGET_ROOTFS_EXT2_SIZE="128M"
# BR2_TARGET_ROOTFS_TAR is not set
BR2_TARGET_UBOOT=y
BR2_TARGET_UBOOT_BUILD_SYSTEM_KCONFIG=y
BR2_TARGET_UBOOT_CUSTOM_VERSION=y
BR2_TARGET_UBOOT_CUSTOM_VERSION_VALUE="<version>"
BR2_TARGET_UBOOT_BOARD_DEFCONFIG="am335x_evm"
BR2_TARGET_UBOOT_FORMAT_IMG=y
BR2_TARGET_UBOOT_SPL=y
BR2_TARGET_UBOOT_SPL_NAME="MLO"
BR2_PACKAGE_HOST_GENIMAGE=y
BR2_PACKAGE_HOST_DOSFSTOOLS=y
BR2_PACKAGE_HOST_MTOOLS=y
```

With `BR2_TOOLCHAIN_EXTERNAL_BOOTLIN`, `menuconfig` picks the Bootlin toolchain that matches the target options; choose the C library and "stable" there and save. Don't write the toolchain variant by hand in the defconfig: its option name comes from the menu.

**What else a tree can hold:** `board/<name>/` for several boards, `configs/` with one defconfig per product variant, and package patches. Buildroot's own tree is the example to copy: a board directory in `board/` and a package in `package/` look the same in your tree.

## 10. Customizing the root filesystem

**How the root filesystem is assembled.** At the end of `make`, after every package is installed in `output/target/`:

```
 skeleton (system/skeleton)  ─┐
 packages' install steps      ├─> output/target/ ─> overlays ─> post-build scripts
 init, getty, hostname, DHCP ─┘                                       │
                                                                      ▼
 users tables, device and permission tables ─> fakeroot ─> rootfs images ─> post-image scripts
```

Everything the rootfs note does by hand in sections 3 to 9 comes from this: the directory layout from the skeleton, `inittab` and the startup scripts from the init choice, `passwd` and `group` from the skeleton and the users tables, `/dev` from the device creation choice.

**System configuration options** (`menuconfig`, System configuration):

| Option | Effect | Rootfs note |
|---|---|---|
| `BR2_TARGET_GENERIC_HOSTNAME`, `BR2_TARGET_GENERIC_ISSUE` | `/etc/hostname`, the banner before the login | Section 9 |
| `BR2_INIT_BUSYBOX`, `BR2_INIT_SYSV`, `BR2_INIT_OPENRC`, `BR2_INIT_SYSTEMD`, `BR2_INIT_NONE` | The init system. BusyBox init is the default; systemd needs glibc | Sections 7, 12 |
| `BR2_ROOTFS_DEVICE_CREATION_DYNAMIC_DEVTMPFS`, `_MDEV`, `_EUDEV`; `_STATIC` | `/dev` management. With systemd, udev comes with it | Section 6 |
| `BR2_TARGET_GENERIC_GETTY_PORT`, `_BAUDRATE` | The console login | Section 7 |
| `BR2_TARGET_ENABLE_ROOT_LOGIN`, `BR2_TARGET_GENERIC_ROOT_PASSWD` | Root login and its password (section 18) | Section 8 |
| `BR2_SYSTEM_DHCP` | An interface configured by DHCP at boot | Section 9 |
| `BR2_TARGET_GENERIC_REMOUNT_ROOTFS_RW` | Remount the root read-write at boot. Off for a read-only root (section 18) | Section 15 |
| `BR2_ROOTFS_MERGED_USR` | `/bin`, `/sbin`, `/lib` as links into `/usr`. Required by systemd | Section 3 |
| `BR2_SYSTEM_BIN_SH_*` | Which shell is `/bin/sh` | Section 4 |

**The rootfs overlay** is a directory copied over `output/target/` at the end of every `make`. Files keep their relative paths:

```
board/bbb/rootfs-overlay/
├── etc/
│   ├── fstab                     replaces the skeleton's
│   ├── network/interfaces
│   └── profile.d/bbb.sh
└── root/.ssh/authorized_keys     needs mode 0700 on .ssh, see below
```

The overlay replaces whole files. To change one line in a file a package installs, use a post-build script instead. The overlay is applied again by every `make`, so an edit to it reaches the image without any rebuild target.

**A post-build script** runs after the overlays, before the images. Its first argument is the target directory, and Buildroot exports `TARGET_DIR`, `HOST_DIR`, `STAGING_DIR`, `BINARIES_DIR`, `BUILD_DIR`, `BASE_DIR` and `BR2_CONFIG` (the path of `.config`). `board/bbb/post-build.sh` (**Illustrative**):

```sh
#!/bin/sh
set -e
TARGET_DIR="$1"
BOARD_DIR="$(dirname "$0")"

# Record which version of the external tree built this image
git -C "$BOARD_DIR" describe --always --dirty > "$TARGET_DIR/etc/bbb-release"

# Don't start telnetd from the default inittab, if a package added it
sed -i '/telnetd/d' "$TARGET_DIR/etc/inittab"
```

Make it executable (`chmod +x`). It runs from the Buildroot tree, as your user, not as root. Scripts listed in `BR2_ROOTFS_POST_BUILD_SCRIPT` (separated by spaces) run in order; `BR2_ROOTFS_POST_SCRIPT_ARGS` adds arguments to every post-build and post-image script.

**Users** come from users tables, one line per user, in the `makeusers` format. `board/bbb/users.txt`:

```
app -1 app -1 =changeme /home/app /bin/sh - Application user
```

| Field | Value here | Meaning |
|---|---|---|
| Username | `app` | |
| UID | `-1` | Chosen by Buildroot from the free range |
| Group | `app` | Created if missing |
| GID | `-1` | Chosen by Buildroot |
| Password | `=changeme` | `=` prefix: clear text, hashed at build time. `!` prefix: login disabled. An existing hash: used as is |
| Home | `/home/app` | Created and owned by the user; `-` for none |
| Shell | `/bin/sh` | |
| Groups | `-` | Extra groups, separated by commas |
| Comment | `Application user` | The rest of the line |

A clear-text password in a table ends up in version control: for a product, put a hash (`openssl passwd -6`, rootfs note section 8), or disable password logins.

**Ownership and permissions** of files in the image come from permission tables in the `makedevs` format (`BR2_ROOTFS_DEVICE_TABLE`, default `system/device_table.txt`, which sets `/etc/shadow` and others). Add your own file to the list:

```
#<name>              <type> <mode> <uid> <gid> <major> <minor> <start> <inc> <count>
/root/.ssh           d      700    0     0     -       -       -       -     -
/usr/bin/hello       f      755    0     0     -       -       -       -     -
```

Types are `f` (file), `d` (directory), `c` and `b` (device nodes, needed only with a static `/dev`), `r` (recursive on a directory). Packages declare their own users and permissions in their `.mk` file (section 11).

**Adding software that Buildroot already packages** is the usual case: switch it on in Target packages. Search with `/`, or read `package/` (`ls package | grep -i <name>`). The package options in `menuconfig` set its build options, the way `PACKAGECONFIG` does in Yocto.

## 11. Your own package

A package is a directory with two files: the menu entry and the build rules. For the `hello` program of the toolchain note, kept in the external tree:

`package/hello/Config.in`:

```
config BR2_PACKAGE_HELLO
	bool "hello"
	help
	  Hello program for the BeagleBone Black.
```

`package/hello/hello.mk`:

```
################################################################################
#
# hello
#
################################################################################

HELLO_VERSION = 1.0
HELLO_SITE = $(BR2_EXTERNAL_BBB_PATH)/package/hello/src
HELLO_SITE_METHOD = local
HELLO_LICENSE = MIT
HELLO_LICENSE_FILES = LICENSE

define HELLO_BUILD_CMDS
	$(TARGET_MAKE_ENV) $(MAKE) $(TARGET_CONFIGURE_OPTS) -C $(@D)
endef

define HELLO_INSTALL_TARGET_CMDS
	$(INSTALL) -D -m 0755 $(@D)/hello $(TARGET_DIR)/usr/bin/hello
endef

define HELLO_INSTALL_INIT_SYSV
	$(INSTALL) -D -m 0755 $(HELLO_PKGDIR)/S99hello $(TARGET_DIR)/etc/init.d/S99hello
endef

$(eval $(generic-package))
```

And `package/hello/src/` holds `hello.c`, `LICENSE` and a `Makefile` that uses `CC`, `CFLAGS` and `LDFLAGS`:

```
hello: hello.c
	$(CC) $(CFLAGS) $(LDFLAGS) -o $@ $<
```

| Line | Meaning |
|---|---|
| `HELLO_` prefix | The package name in upper case, with `-` replaced by `_`. It must match the directory and file name (`hello/hello.mk`) |
| `HELLO_SITE`, `HELLO_SITE_METHOD = local` | The source is a local directory, copied with `rsync` into `output/build/hello-1.0/` |
| `HELLO_LICENSE`, `HELLO_LICENSE_FILES` | The license (SPDX identifier) and the files that hold it, for `make legal-info` (section 18) |
| `HELLO_BUILD_CMDS` | The build step. `$(@D)` is the build directory. `TARGET_CONFIGURE_OPTS` sets `CC`, `CFLAGS`, `LDFLAGS` and the other tools to the cross-toolchain |
| `HELLO_INSTALL_TARGET_CMDS` | Installs into `TARGET_DIR`, the `DESTDIR` of the rootfs note |
| `HELLO_INSTALL_INIT_SYSV` | Installs the start script when the init system is BusyBox or SysV init; `HELLO_INSTALL_INIT_SYSTEMD` does the same for systemd |
| `$(HELLO_PKGDIR)` | The package's directory, for files kept next to the `.mk` |
| `$(eval $(generic-package))` | Turns the variables into the download, extract, patch, configure, build and install steps |

The lines inside `define` blocks start with a tab, as in any Makefile. `utils/check-package package/hello/*` checks the formatting against Buildroot's coding style.

`package/hello/S99hello`, a start script in the format of the rootfs note (section 7):

```sh
#!/bin/sh
case "$1" in
start)   printf "Starting hello: "; start-stop-daemon -S -b -q -x /usr/bin/hello && echo OK ;;
stop)    printf "Stopping hello: "; start-stop-daemon -K -q -x /usr/bin/hello && echo OK ;;
restart) "$0" stop; "$0" start ;;
*)       echo "Usage: $0 {start|stop|restart}"; exit 1 ;;
esac
```

Then list the package in the tree's `Config.in` (section 9), switch it on, and build it:

```sh
$ make menuconfig            # External options -> hello
$ make hello
$ make                       # to get it into the images
```

**A package for a real project**, downloaded and built with its own build system, uses an infrastructure. `package/libfoo/libfoo.mk` for a CMake project (**Illustrative**):

```
LIBFOO_VERSION = 1.2.3
LIBFOO_SITE = https://example.org/releases
LIBFOO_SOURCE = libfoo-$(LIBFOO_VERSION).tar.gz
LIBFOO_LICENSE = BSD-3-Clause
LIBFOO_LICENSE_FILES = LICENSE
LIBFOO_CPE_ID_VENDOR = example
LIBFOO_INSTALL_STAGING = YES
LIBFOO_DEPENDENCIES = host-pkgconf zlib openssl
LIBFOO_CONF_OPTS = -DBUILD_TESTS=OFF

$(eval $(cmake-package))
```

`package/libfoo/libfoo.hash`:

```
# Locally computed
sha256  <sha256 of the tarball>  libfoo-1.2.3.tar.gz
sha256  <sha256 of the file>     LICENSE
```

`package/libfoo/Config.in`:

```
config BR2_PACKAGE_LIBFOO
	bool "libfoo"
	depends on BR2_TOOLCHAIN_HAS_THREADS
	select BR2_PACKAGE_ZLIB
	select BR2_PACKAGE_OPENSSL
	help
	  Example library.

	  https://example.org/libfoo

comment "libfoo needs a toolchain w/ threads"
	depends on !BR2_TOOLCHAIN_HAS_THREADS
```

| Element | Meaning |
|---|---|
| `LIBFOO_SITE`, `LIBFOO_SOURCE` | Where to download, and the file name. For git: `LIBFOO_SITE = https://example.org/libfoo.git`, `LIBFOO_SITE_METHOD = git`, `LIBFOO_VERSION = <commit or tag>`. For GitHub: `$(call github,<owner>,<repo>,$(LIBFOO_VERSION))` |
| `.hash` file | Checked after every download. A package with a hash file and no entry for the downloaded file fails (section 20) |
| `LIBFOO_INSTALL_STAGING = YES` | Also installs into staging, so other packages can link to the library. Off by default |
| `LIBFOO_DEPENDENCIES` | Packages to build first. `host-pkgconf` when the build calls `pkg-config` |
| `LIBFOO_CONF_OPTS` | Options for the infrastructure: CMake `-D` options, `./configure` options, Meson options |
| `LIBFOO_CPE_ID_*` | The package's name in the vulnerability database, for `make pkg-stats` (section 18) |
| `select` | Switches on the libraries the package needs |
| `depends on`, `comment` | Hides the package when a toolchain feature is missing, and says why in the menu |

**A rule that catches everyone:** `select` doesn't follow the selected package's own `depends on`. If `openssl` depends on a toolchain feature, `libfoo` must depend on it too, or the build fails with a broken configuration.

**The infrastructures:**

| Infrastructure | For | Typical variables |
|---|---|---|
| `generic-package` | Anything, with your own commands | `_BUILD_CMDS`, `_INSTALL_TARGET_CMDS` |
| `autotools-package` | `./configure && make` | `_CONF_OPTS`, `_AUTORECONF = YES` |
| `cmake-package` | CMake | `_CONF_OPTS` |
| `meson-package` | Meson | `_CONF_OPTS` |
| `python-package` | Python modules | `_SETUP_TYPE` (`setuptools`, `pep517`, `flit`...) |
| `cargo-package`, `golang-package` | Rust, Go | Vendored dependencies are downloaded with the source |
| `kernel-module` | Out-of-tree modules, combined with `generic-package` | `_MODULE_SUBDIRS` (section 12) |
| `kconfig-package` | Packages configured with Kconfig | `_KCONFIG_FILE`, `_KCONFIG_FRAGMENT_FILES` |

Prefixing an infrastructure with `host-` (`$(eval $(host-cmake-package))`) builds the same package for the build machine.

**What a package can also declare:**

| Variable | Effect |
|---|---|
| `HELLO_USERS` | Users and groups, one per line, in the users table format of section 10 |
| `HELLO_PERMISSIONS` | Permission table lines for the package's files |
| `HELLO_POST_INSTALL_TARGET_HOOKS` | Extra steps after an install step: `HELLO_POST_INSTALL_TARGET_HOOKS += HELLO_EXTRA_FILES` with a `define HELLO_EXTRA_FILES` block |
| `HELLO_PRE_CONFIGURE_HOOKS`, `HELLO_POST_PATCH_HOOKS`... | The same for every step |
| `HELLO_IGNORE_CVES` | Known vulnerabilities that a patch in the package fixes (section 18) |

**Patches** for a package go in its directory as `<number>-<description>.patch`, applied in order. For packages you don't own, use the global patch directory instead (section 12), so Buildroot's own `package/` stays untouched.

## 12. The kernel and the bootloader

**The kernel** is the `linux` package, configured in the Kernel menu. It runs the commands of the kernel note: configure, `zImage`, `dtbs`, `modules`, `modules_install` into the target.

| Option | Meaning | Kernel note |
|---|---|---|
| `BR2_LINUX_KERNEL_CUSTOM_VERSION_VALUE` | A kernel.org release, downloaded as a tarball | Section 3 |
| `BR2_LINUX_KERNEL_CUSTOM_GIT`, `_CUSTOM_REPO_URL`, `_CUSTOM_REPO_VERSION` | A git tree at a commit or tag: a vendor's kernel, or your own branch | Section 16 |
| `BR2_LINUX_KERNEL_CUSTOM_TARBALL_LOCATION` | Any tarball URL | |
| `BR2_LINUX_KERNEL_DEFCONFIG` | An in-tree defconfig, without `_defconfig`: `omap2plus` | Section 5 |
| `BR2_LINUX_KERNEL_USE_CUSTOM_CONFIG`, `_CUSTOM_CONFIG_FILE` | A full configuration file of yours, instead of an in-tree defconfig | Section 5 |
| `BR2_LINUX_KERNEL_CONFIG_FRAGMENT_FILES` | Fragments merged after the base, in order | Section 5 |
| `BR2_LINUX_KERNEL_ZIMAGE`, `_UIMAGE`, `_IMAGE`... | The image format | Section 7 |
| `BR2_LINUX_KERNEL_INTREE_DTS_NAME` | Device trees from the kernel tree, without `.dts`: `ti/omap/am335x-boneblack` since Linux 6.5 moved the Arm ones into vendor directories | Section 12 |
| `BR2_LINUX_KERNEL_CUSTOM_DTS_PATH` | Your own `.dts` files, built with the kernel's include paths | Section 16 |
| `BR2_LINUX_KERNEL_INSTALL_TARGET` | Also install the kernel and DTBs in `/boot` of the root filesystem | Section 9 |
| `BR2_LINUX_KERNEL_NEEDS_HOST_OPENSSL`, `_LIBELF`, `_PAHOLE` | Host libraries the kernel build needs for some options (module signing, BTF) | Section 6 |

**A kernel fragment**, `board/bbb/linux.fragment`, in the same form as the kernel note's `bbb.config` (section 5):

```
CONFIG_IKCONFIG=y
CONFIG_IKCONFIG_PROC=y
# CONFIG_DRM is not set
```

To write one: `make linux-menuconfig`, change the options, then `make linux-diff-config` and copy what it prints. Check the result as in the kernel note: an option whose dependencies aren't met disappears silently from the `.config` in `output/build/linux-<version>/`.

**BusyBox** works the same: `BR2_PACKAGE_BUSYBOX_CONFIG` for a full configuration, `BR2_PACKAGE_BUSYBOX_CONFIG_FRAGMENT_FILES` for fragments. `board/bbb/busybox.fragment`:

```
# CONFIG_TELNETD is not set
CONFIG_FEATURE_VI_COLON=y
```

**Kernel patches.** The global patch directory holds patches for any package, by package name and, optionally, version:

```
patches/
├── linux/
│   ├── 0001-ARM-dts-am335x-boneblack-add-my-cape.patch
│   └── linux.hash                  hash of the custom kernel tarball
└── uboot/
    └── 0001-<change>.patch
```

`BR2_GLOBAL_PATCH_DIR="$(BR2_EXTERNAL_BBB_PATH)/patches"` names it. Patches in `patches/linux/<version>/` apply only to that version. Write them with `git format-patch` from the kernel tree (kernel note, section 16). A `.hash` file next to them gives the custom version a checksum; with `BR2_DOWNLOAD_FORCE_CHECK_HASHES`, Buildroot refuses any download without one.

**Out-of-tree modules** combine `kernel-module` with `generic-package`:

```
MYDRV_VERSION = 1.0
MYDRV_SITE = $(BR2_EXTERNAL_BBB_PATH)/package/mydrv/src
MYDRV_SITE_METHOD = local
MYDRV_LICENSE = GPL-2.0

$(eval $(kernel-module))
$(eval $(generic-package))
```

The source holds a `Kbuild` or `Makefile` with `obj-m += mydrv.o`, as in the kernel note (section 15). Buildroot builds it against its own kernel and installs it in `/lib/modules/<release>/updates/`, so the vermagic matches (kernel note, section 8). `Config.in` needs `depends on BR2_LINUX_KERNEL`.

**The bootloader** is the U-Boot package, in the Bootloaders menu:

| Option | Meaning | Bootloader note |
|---|---|---|
| `BR2_TARGET_UBOOT_CUSTOM_VERSION_VALUE` | A release: the same choices as for the kernel (version, git, tarball) | Section 4 |
| `BR2_TARGET_UBOOT_BOARD_DEFCONFIG` | `am335x_evm`, without `_defconfig` | Section 4 |
| `BR2_TARGET_UBOOT_CONFIG_FRAGMENT_FILES` | Fragments, for example a different boot delay | Section 4 |
| `BR2_TARGET_UBOOT_FORMAT_IMG`, `BR2_TARGET_UBOOT_SPL`, `_SPL_NAME="MLO"` | Which files to copy to `output/images/` | Section 9 |
| `BR2_TARGET_UBOOT_NEEDS_DTC`, `_PYLIBFDT`, `_OPENSSL`, `_GNUTLS`, `_UTIL_LINUX` | Host tools recent U-Boot versions need. The build error names the missing one | |
| `BR2_PACKAGE_UBOOT_TOOLS` | `mkimage` for the build machine, `fw_printenv` and `fw_setenv` for the board | Sections 12, 15 |

A recent U-Boot often needs more host tools than the defconfig of an older Buildroot release switches on: when the U-Boot build stops with a missing Python module or header, switch on the matching `NEEDS` option.

**Using the trees of the fundamentals notes.** To build the kernel or U-Boot from a tree you already work in, point the package to it with `OVERRIDE_SRCDIR` (section 15). Buildroot then builds your checkout, with its configuration options.

## 13. The card image: genimage

`genimage` builds partition images and a partitioned disk image from a description, without root rights and without loop devices. The rootfs note introduces it (section 11). Buildroot runs it from a post-image script, after the root filesystem images are made.

`board/bbb/genimage.cfg` (**Illustrative**):

```
image boot.vfat {
	vfat {
		files = {
			"MLO",
			"u-boot.img",
			"zImage",
			"am335x-boneblack.dtb"
		}
		file extlinux/extlinux.conf {
			image = "extlinux.conf"
		}
	}
	size = 32M
}

image sdcard.img {
	hdimage {
	}

	partition boot {
		partition-type = 0xC
		bootable = "true"
		image = "boot.vfat"
	}

	partition rootfs {
		partition-type = 0x83
		image = "rootfs.ext4"
	}
}
```

| Element | Meaning |
|---|---|
| `image boot.vfat` | A FAT filesystem image with the boot files, taken from `output/images/` |
| `file extlinux/extlinux.conf` | A file placed in a subdirectory of the FAT image |
| `hdimage` | A disk image with an MBR partition table. `partition-table-type = "gpt"` for GPT |
| `partition-type = 0xC`, `bootable = "true"` | FAT32 with the boot flag: the AM335x ROM code looks for `MLO` on the active FAT partition (bootloader note) |
| `image = "rootfs.ext4"` | The root filesystem image Buildroot made (`BR2_TARGET_ROOTFS_EXT2_4`) |

`board/bbb/extlinux.conf`, read by U-Boot's standard boot (bootloader note, section 8):

```
label buildroot
    kernel /zImage
    fdt /am335x-boneblack.dtb
    append console=ttyS0,115200n8 root=/dev/mmcblk0p2 rootwait ro
```

`ro` mounts the root read-only, then the init scripts remount it read-write unless `BR2_TARGET_GENERIC_REMOUNT_ROOTFS_RW` is off (section 18). `mmcblk0` is the SD card with the mainline device tree; the eMMC is `mmcblk1`.

`board/bbb/post-image.sh` copies the file into place and runs `genimage` through Buildroot's wrapper:

```sh
#!/bin/sh
set -e
BOARD_DIR="$(dirname "$0")"

cp "$BOARD_DIR/extlinux.conf" "$BINARIES_DIR/extlinux.conf"
support/scripts/genimage.sh -c "$BOARD_DIR/genimage.cfg"
```

`support/scripts/genimage.sh` sets `genimage`'s input to `output/images/`, its temporary directory and its root path. Many board directories in Buildroot skip the script and set `BR2_ROOTFS_POST_IMAGE_SCRIPT="support/scripts/genimage.sh"` with `BR2_ROOTFS_POST_SCRIPT_ARGS="-c <path>/genimage.cfg"`, as `beaglebone_defconfig` does (section 7). Remember that the arguments go to post-build scripts too.

**A product layout** adds what the rootfs note describes in sections 15 and 16: two root partitions for A/B updates and a data partition (**Illustrative**):

```
image sdcard.img {
	hdimage {
	}
	partition boot {
		partition-type = 0xC
		bootable = "true"
		image = "boot.vfat"
	}
	partition rootfsA {
		partition-type = 0x83
		image = "rootfs.ext4"
		size = 256M
	}
	partition rootfsB {
		partition-type = 0x83
		image = "rootfs.ext4"
		size = 256M
	}
	partition data {
		partition-type = 0x83
		image = "data.ext4"
	}
}
```

An MBR holds four primary partitions, which this layout uses. For more, use GPT (`partition-table-type = "gpt"` in `hdimage`) only if the boot ROM supports it: the AM335x ROM reads an MBR to find `MLO` on an SD card. `data.ext4` is another `image` block with an `ext4` section and a size, made by `genimage` itself.

**Filesystem image types**, in the Filesystem images menu:

| Option | Image | Rootfs note |
|---|---|---|
| `BR2_TARGET_ROOTFS_EXT2` with `_EXT2_4`, `_EXT2_SIZE` | ext4, of a fixed size | Section 10 |
| `BR2_TARGET_ROOTFS_SQUASHFS` | Compressed, read-only | Section 10 |
| `BR2_TARGET_ROOTFS_EROFS` | Compressed, read-only, faster random reads | Section 10 |
| `BR2_TARGET_ROOTFS_UBI`, `_UBIFS` | For raw NAND flash | Section 10 |
| `BR2_TARGET_ROOTFS_CPIO` | An initramfs archive | Section 10 |
| `BR2_TARGET_ROOTFS_INITRAMFS` | The cpio built into the kernel image | Kernel note, section 13 |
| `BR2_TARGET_ROOTFS_TAR` | An archive, for NFS root or for unpacking elsewhere | Section 11 |

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
