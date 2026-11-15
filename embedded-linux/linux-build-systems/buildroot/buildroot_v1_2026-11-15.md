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
