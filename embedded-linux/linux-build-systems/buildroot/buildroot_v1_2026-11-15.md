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
