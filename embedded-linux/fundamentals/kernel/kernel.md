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

The bootloader ends with a jump. When the kernel's first instruction runs, there is RAM, a CPU, a compressed kernel image somewhere in that RAM, and a device tree blob whose address is in a register. Nothing else is set up: no drivers, no filesystems, no processes. When the kernel's own boot ends, one program, `init`, runs with process ID 1 from a root filesystem, with `/dev/console` as its standard input and output. Everything in this note happens between those two points, or explains how to produce the files involved.

A word used throughout: **userspace** means the programs that run outside the kernel, starting with `init`. They reach the hardware only through the kernel, with system calls, device files in `/dev` and the virtual filesystems `/proc` and `/sys`. The kernel runs in a privileged CPU mode; userspace doesn't. On the board, "the kernel booted" means userspace started.

**What the kernel receives, and from whom:**

| Input | Comes from | On the BeagleBone Black |
|---|---|---|
| The kernel image in RAM | The bootloader, loaded from storage or the network | `zImage` at `0x82000000` ([bootloader note](../bootloader/bootloader.md)) |
| A description of the hardware | The device tree blob (DTB), address in `r2` | `am335x-boneblack.dtb`, built with the kernel (section 12) |
| The command line | `/chosen/bootargs` in the DTB, written by U-Boot from its `bootargs` variable | `console=ttyS0,115200n8 root=/dev/mmcblk0p2 rootwait` (section 10) |
| Optionally, an initramfs | Built into the kernel, or loaded by the bootloader next to the DTB | none by default (section 13) |

**What it must find on its own:** a driver for the serial port, to print anything; a driver for the storage that holds the root filesystem (the AM3358's SD/MMC controller); the code for that filesystem (ext4); and an executable `init` on it. If any of these is missing, the boot stops, and the last line on the console tells you which one (section 24).

**The stages**, from source (`init/main.c`, `init/do_mounts.c`, `arch/arm/kernel/`):

```
 zImage entry          decompressor: relocates if needed, unpacks the kernel
      │                (prints nothing on this kernel, section 11)
      ▼
 start_kernel()        reads the DTB: machine, memory, command line
      │                sets up memory management, interrupts, timers,
      │                the scheduler; the console starts here
      ▼
 kernel_init()         runs the initcalls: buses, then drivers; each driver
      │                matches device tree nodes and "probes" its device
      ▼
 root filesystem       unpacks the initramfs if any; otherwise waits for the
      │                root= device, mounts it, mounts devtmpfs on /dev
      ▼
 init (PID 1)          /sbin/init or init=...; the kernel's boot is over
```

**Three things that must agree.** Most boot problems are a disagreement between them:

| The kernel configuration (`.config`) | The device tree (`.dtb`) | The root filesystem |
|---|---|---|
| Which drivers exist, and whether they are built in or modules | Which devices exist on this board, at which addresses, which are enabled | Which modules are installed (`/lib/modules/<release>/`), which `init` runs |
| Changed with `menuconfig`, fragments (section 5) | Changed by editing a `.dts` or by overlays (section 12) | Changed by the rootfs build ([rootfs note](../rootfs/)) |

A driver that's enabled in the configuration does nothing if the device tree doesn't describe its device; a device in the tree stays silent if its driver isn't built; a driver built as a module needs a matching `/lib/modules/<release>/` on the root filesystem, where `<release>` is the exact version string of the running kernel (section 8).

## 2. The example board, kernel and files

The board is the one from the bootloader note: a **BeagleBone Black** (TI AM3358, one Cortex-A8 core, 512 MiB of DDR3L, 4 GB eMMC, microSD slot), with U-Boot loading the kernel from the SD card and the serial console on J1 at 115200 8N1.

| Element | Value |
|---|---|
| Kernel source | Mainline Linux at commit `66edb901bf87` (2 May 2026), just after 7.1-rc1. `make kernelversion` prints `7.1.0-rc1` |
| Configuration | `omap2plus_defconfig`, the mainline configuration for TI's 32-bit Arm SoCs (OMAP3/4/5, AM335x, AM437x, DRA7) |
| Cross-compiler | Arm GNU Toolchain 15.2.Rel1, prefix `arm-none-linux-gnueabihf-`. The distribution's `arm-linux-gnueabihf-gcc` works too: the kernel needs no C library ([toolchain note](../cross-compilation-toolchain/cross-compilation-toolchain.md)) |
| Source directory | `~/bbb/linux` |
| Build directory | `~/bbb/build`, separate from the source (`O=`, section 6) |
| Root filesystem | ext4 on SD card partition 2 (`/dev/mmcblk0p2`), or NFS during development |

The build commands in this note use two variables, which every cross build of the kernel needs:

```sh
$ export ARCH=arm
$ export CROSS_COMPILE=arm-none-linux-gnueabihf-
```

`ARCH` selects the `arch/` subdirectory (and so the Kconfig tree, the defconfigs and the image formats); `CROSS_COMPILE` is the prefix in front of `gcc`, `ld`, `objcopy` and the other tools. Both must be set for every `make` call in the same build directory, including `menuconfig` and `modules_install`: a missing `ARCH` makes `make` use the build machine's architecture, with a different set of options.

The files the board needs, with their sizes in the earlier build of this tree (section 7):

| File | Produced by | Size | Read by | Where it goes |
|---|---|---|---|---|
| `arch/arm/boot/zImage` | `make zImage` | 5,813,512 bytes | U-Boot (`bootz`) | boot partition, or `/boot` on the rootfs |
| `arch/arm/boot/dts/ti/omap/am335x-boneblack.dtb` | `make dtbs` | 98,077 bytes | U-Boot, then the kernel | next to the `zImage` |
| `lib/modules/<release>/`: 860 `.ko` files plus index files | `make modules`, `make modules_install` | tens of MiB | the kernel, through `modprobe` | the root filesystem |

The `zImage` and DTB are the same files the bootloader note decoded.

## 3. Which kernel: mainline, stable, LTS or vendor

Kernel versions come from several places, and the choice decides how much of this note works as written.

| Source | What it is | Release rhythm | Use it when |
|---|---|---|---|
| **Mainline** (Linus Torvalds' tree) | Where development lands. A two-week merge window ends with `-rc1`, then weekly release candidates until the release | A release about every 9 to 10 weeks | Your SoC is well supported upstream, you want the newest drivers, or you work on the kernel itself |
| **Stable** (`linux-7.0.y`) | Bug and security fixes backported to a release | Until shortly after the next release, then end of life | You follow mainline releases but want fixes |
| **Longterm (LTS)** | A stable branch kept for several years; usually the last mainline release of each year | Fixes for at least two years, longer for some | Products: one base for the product's life, with fixes |
| **Civil Infrastructure Platform (CIP)** | Some LTS kernels maintained for about 10 years by the CIP project | Long | Industrial products that ship for a decade |
| **Vendor kernel** (TI's `ti-linux-kernel`, BeagleBoard.org's kernels) | An LTS kernel plus the vendor's patches for features not upstream yet | Follows the vendor's SDK | The SoC needs drivers that mainline lacks (GPU, some remoteproc and PRU uses, new SoCs) |

Version numbers: `7.1` is a mainline release, `7.1.3` is the third stable update of it, `7.1-rc1` is the first release candidate. The `make kernelversion` form adds a `.0` (`7.1.0-rc1`), and the release string the running kernel reports may add a suffix (section 8). kernel.org lists the current stable and longterm branches and their end-of-life dates; check it rather than a date written in a note.

The AM335x has been supported in mainline for over a decade, so a mainline or LTS kernel with `omap2plus_defconfig` boots a BeagleBone Black with Ethernet, SD/eMMC, USB, I2C, SPI, GPIO and the PMIC working. This note uses a mainline snapshot because it's what the example build used; a product would start from an LTS release and follow its stable updates (section 22).

**Rule of thumb for a new board:** start with whatever kernel the board's vendor ships, and check how far mainline is from it. The further it is, the more of your time goes into carrying the vendor's patches rather than your own.

## 4. The source tree and Kbuild

A kernel tree is large (about 1.8 GB of files without its Git history), but a board's work touches a few places:

| Path | What's there | For the BeagleBone Black |
|---|---|---|
| `arch/arm/` | Everything specific to 32-bit Arm | |
| `arch/arm/configs/` | Default configurations | `omap2plus_defconfig` |
| `arch/arm/boot/dts/ti/omap/` | Device tree sources for TI's 32-bit SoCs (149 `.dts` files) | `am335x-boneblack.dts` and the `.dtsi` files it includes |
| `arch/arm/mach-omap2/` | SoC support code: power and clock domains, early setup | Generic for all OMAP2+ chips, matched by `ti,am33xx` (section 12) |
| `arch/arm/boot/compressed/` | The `zImage` decompressor | |
| `drivers/` | Device drivers, by subsystem | `drivers/net/ethernet/ti/` (CPSW Ethernet), `drivers/mmc/host/sdhci-omap.c`, `drivers/tty/serial/8250/8250_omap.c` |
| `init/` | `start_kernel()`, mounting the root, starting `init` | Section 13 |
| `include/uapi/` | The headers userspace sees (system calls, ioctls) | Installed into toolchains with `make headers_install` |
| `scripts/` | Build tools: `kconfig/`, `dtc/`, `config`, `diffconfig`, `decode_stacktrace.sh`, `faddr2line` | Sections 5 and 19 |
| `Documentation/` | The kernel's documentation (also at https://docs.kernel.org) | `admin-guide/kernel-parameters.txt`, `arch/arm/booting.rst` |

Two files in every directory drive the build. A `Kconfig` file declares options; a `Makefile` says which objects each option builds. The USB Ethernet gadget, used as the example of section 5, from source:

```
# drivers/usb/gadget/legacy/Kconfig
config USB_ETH
	tristate "Ethernet Gadget (with CDC Ethernet support)"
	depends on NET
	select USB_LIBCOMPOSITE
	select USB_U_ETHER
	...

# drivers/usb/gadget/legacy/Makefile
g_ether-y			:= ether.o
obj-$(CONFIG_USB_ETH)		+= g_ether.o
```

- `tristate` means the option can be `y` (built into the kernel image), `m` (built as a loadable module, here `g_ether.ko`) or `n`. A `bool` option has only `y` and `n`.
- `obj-$(CONFIG_USB_ETH)` becomes `obj-y`, `obj-m`, or `obj-` (ignored), depending on the value. This one line is the whole mechanism that decides between built-in and module.
- `depends on` hides the option until its dependencies are met; `select` forces other options on without checking their own dependencies. A `select` is why enabling one driver can turn on a library you never chose.
- Device trees use the same mechanism: `dtb-$(CONFIG_SOC_AM33XX) += am335x-boneblack.dtb` in `arch/arm/boot/dts/ti/omap/Makefile` builds the BeagleBone Black's DTB whenever AM335x support is on.

This build system is called **Kbuild**. Out-of-tree modules use it too (section 15).

## 5. Configuring: defconfig, menuconfig and fragments

The configuration is one file, `.config`, in the build directory: about 8,300 lines of `CONFIG_FOO=y`, `=m`, `="string"` or `# CONFIG_FOO is not set`. Nobody writes it by hand. It's generated from a **defconfig**, then changed with tools.

**Start from the defconfig.** **Real output**, shortened:

```
$ cd ~/bbb/linux
$ make O=~/bbb/build omap2plus_defconfig
  HOSTCC  scripts/basic/fixdep
  ...
  HOSTLD  scripts/kconfig/conf
#
# configuration written to .config
#
```

`make` first builds the configuration tool for the build machine (`HOSTCC`: compiled with the build machine's own compiler), then expands the 724-line `omap2plus_defconfig` into a full `.config`, in a few seconds. The result has 1,431 options set to `y` and 737 set to `m`.

**A defconfig is a minimal list.** It holds only the options that differ from their defaults; every other value comes from the `Kconfig` files. `make savedefconfig` produces one from a `.config` (it writes `defconfig` in the build directory). On this tree, saving the configuration right after loading `omap2plus_defconfig` gives a 697-line file: the upstream defconfig carries 29 lines that `savedefconfig` drops, because they're already the default, are forced by a `select`, or name options that no longer exist (`CONFIG_LEDS_LP5523` is one). A defconfig is minimal by convention, not by guarantee.

**Changing options.** Four tools, for different jobs:

| Tool | Use it for |
|---|---|
| `make menuconfig` (or `nconfig`, `xconfig`) | Exploring. `/` searches for a symbol and shows its location, its value, what it `depends on` and what `select`s it. Needs `libncurses-dev` on the build machine |
| `scripts/config --file .config -e FOO -m BAR -d BAZ` | Scripted changes, without a menu. `--set-str LOCALVERSION -bbb` for strings, `-s FOO` to print a value |
| A config fragment and `scripts/kconfig/merge_config.sh` | Keeping your changes as a short file in your own repository (below) |
| `make olddefconfig` | After any of the above: resolves dependencies and fills new options with their defaults, without asking |

`scripts/config` edits the text and checks nothing. Two real examples of what that means:

- **The parent limits the child.** In `omap2plus_defconfig`, `USB_GADGET` is `m`, and `USB_ETH` lives inside it, so it can't be built in. **Real output**:

  ```
  $ scripts/config --file .config -e USB_ETH
  $ grep '^CONFIG_USB_ETH=' .config
  CONFIG_USB_ETH=y
  $ make -s olddefconfig
  $ grep -E '^CONFIG_(USB_ETH|USB_GADGET)=' .config
  CONFIG_USB_GADGET=m
  CONFIG_USB_ETH=m
  ```

  `olddefconfig` turned `y` into `m` without a word.

- **Choices need the choice member.** `omap2plus_defconfig` enables debug information with `CONFIG_DEBUG_INFO_DWARF4=y`, one entry in a `choice` of DWARF versions. `scripts/config -d DEBUG_INFO` followed by `olddefconfig` changes nothing: `DEBUG_INFO` is selected by the chosen entry. Switching the choice does it: `scripts/config -d DEBUG_INFO_DWARF4 -e DEBUG_INFO_NONE`. Debug information makes the build directory several times larger; it's needed to decode crashes with line numbers (section 19).

**Always check the result** with `scripts/diffconfig`, which compares two `.config` files. Comparing the configuration of the earlier BeagleBone build against a fresh `omap2plus_defconfig` shows what its owner changed. **Real output**:

```
$ scripts/diffconfig defconfig-build/.config previous-build/.config
 USB_ETH n -> m
+USB_ETH_EEM y
+USB_ETH_RNDIS y
```

One change: the USB Ethernet gadget, as a module, with RNDIS (so that Windows sees the board as a network adapter over USB) and EEM. The two `+` lines are options that only exist once `USB_ETH` is enabled.

**Keep your changes as a fragment.** A fragment is a `.config`-style file with only the lines you care about. `merge_config.sh` applies it on top of a base defconfig. The fragment used for the tests below:

```
# bbb.config - changes on top of omap2plus_defconfig
CONFIG_LOCALVERSION="-bbb"
CONFIG_USB_ETH=m
CONFIG_USB_ETH_RNDIS=y
CONFIG_DEBUG_INFO_NONE=y
CONFIG_PSTORE=y
CONFIG_PSTORE_RAM=y
CONFIG_DYNAMIC_DEBUG=y
CONFIG_NO_SUCH_OPTION=y
```

The last line is a deliberate typo. **Real output**:

```
$ mkdir -p ~/bbb/build
$ scripts/kconfig/merge_config.sh -m -O ~/bbb/build arch/arm/configs/omap2plus_defconfig bbb.config
Using arch/arm/configs/omap2plus_defconfig as base
Merging bbb.config
#
# merged configuration written to ~/bbb/build/.config (needs make)
#
$ make O=~/bbb/build olddefconfig
$ scripts/diffconfig defconfig-build/.config ~/bbb/build/.config
 ...
 DEBUG_INFO_DWARF4 y -> n
 DEBUG_INFO_NONE n -> y
 DYNAMIC_DEBUG n -> y
 DYNAMIC_DEBUG_CORE n -> y
 LOCALVERSION "" -> "-bbb"
 PSTORE n -> y
 USB_ETH n -> m
+PSTORE_RAM y
 ...
+USB_ETH_RNDIS y
```

(The `...` lines are dependent options that appeared or disappeared, such as the `DEBUG_INFO_*` sub-options and `PSTORE_COMPRESS`.) Three things to know about the script:

- **The `-O` directory must exist.** When it doesn't, the script prints `output directory ... does not exist` and writes nothing, and a following `make olddefconfig` builds a default `.config` for the generic Arm platform, not for the BeagleBone. Nothing warns you; `diffconfig` shows hundreds of differences.
- **With `-m` ("merge only"), the typo disappears silently** in `olddefconfig`. Without `-m`, the script runs `make alldefconfig` itself and checks each requested value. **Real output**, shortened:

  ```
  $ scripts/kconfig/merge_config.sh -O ~/bbb/build arch/arm/configs/omap2plus_defconfig bbb.config
  ...
  WARNING: Value requested for CONFIG_LEDS_LP5523 not in final .config
  Requested value: CONFIG_LEDS_LP5523=m
  Actual value:
  WARNING: CONFIG_DEBUG_INFO_DWARF4 differs:
  Requested value: CONFIG_DEBUG_INFO_DWARF4=y
  Actual value:    # CONFIG_DEBUG_INFO_DWARF4 is not set
  WARNING: Value requested for CONFIG_DEBUG_INFO_SPLIT not in final .config
  ...
  WARNING: Value requested for CONFIG_NO_SUCH_OPTION not in final .config
  Requested value: CONFIG_NO_SUCH_OPTION=y
  Actual value:
  ```

  The typo is caught, together with warnings about stale lines of the upstream defconfig and about options the fragment overrides on purpose. Run without `-m` whenever you change the fragment, and read the warnings that name your own options.
- **Later files win.** Several fragments can follow the base (`base.config board.config debug.config`), which is how Buildroot (`BR2_LINUX_KERNEL_CONFIG_FRAGMENT_FILES`) and Yocto (`.cfg` files in `SRC_URI`) apply theirs.

**Read the configuration of a running kernel.** `omap2plus_defconfig` sets `CONFIG_IKCONFIG_PROC=y`, so the running kernel carries its own configuration:

```sh
# zcat /proc/config.gz | grep USB_ETH
```

It's the fastest answer to "is this driver built in?", and a way to recover the configuration of a kernel whose sources are lost. `scripts/extract-ikconfig` does the same from a `zImage` or `vmlinux` file on the build machine.

## 6. Building

**Host packages** (Debian and Ubuntu names). The kernel builds its own `dtc` and configuration tools, but needs these on the build machine:

```sh
$ sudo apt install build-essential bc bison flex libssl-dev libelf-dev \
      libncurses-dev xz-utils u-boot-tools
```

`libssl-dev` is for module signing and certificate tools, `libelf-dev` for `objtool` and BTF, `libncurses-dev` for `menuconfig`, `xz-utils` for the LZMA compressor used by `omap2plus_defconfig`'s `zImage`, `u-boot-tools` only for `make uImage`.

**The build.** With `ARCH` and `CROSS_COMPILE` exported (section 2):

```sh
$ cd ~/bbb/linux
$ make O=~/bbb/build omap2plus_defconfig          # or merge_config.sh, section 5
$ make O=~/bbb/build -j"$(nproc)" zImage modules dtbs
```

- **`O=` keeps every generated file in the build directory**, so one source tree serves several boards or configurations, and `rm -rf ~/bbb/build` is a full clean. The source tree must then stay clean: once a `.config` exists in the source directory itself, an `O=` build stops and asks for `make mrproper`. Pick one style per tree.
- **`-j"$(nproc)"`** runs one job per CPU. A full `omap2plus_defconfig` build compiles the kernel plus about 860 modules: expect tens of minutes on a laptop, less on a workstation, and a few GB in the build directory (several times more with debug information).
- **Only `zImage`, `modules` and `dtbs` are needed** for this board. Plain `make` builds the default targets of the architecture (on Arm: `zImage`, `modules` and `dtbs`, marked with `*` in `make help`).

| Target | Builds | Notes |
|---|---|---|
| `zImage` | `arch/arm/boot/zImage` | Self-decompressing kernel, booted with `bootz` |
| `Image` | `arch/arm/boot/Image` | Uncompressed, rarely used on 32-bit Arm |
| `uImage` | `arch/arm/boot/uImage` | `zImage` behind a legacy U-Boot header. Needs `LOADADDR=0x80008000` on a multi-platform kernel; `bootz` makes it unnecessary (bootloader note) |
| `modules` | Every `=m` option as a `.ko` file | |
| `dtbs` | Every DTB enabled by the configuration | `make ti/omap/am335x-boneblack.dtb` builds one |
| `modules_install`, `dtbs_install`, `headers_install` | Copies into a staging directory | Section 9 |
| `kernelrelease`, `kernelversion` | Prints the version strings | Section 8 |
| `clean`, `mrproper` | Removes outputs; `mrproper` also removes `.config` | |

The Arm-specific part of `make help` (**Real output**):

```
Architecture-specific targets (arm):
* zImage        - Compressed kernel image (arch/arm/boot/zImage)
  Image         - Uncompressed kernel image (arch/arm/boot/Image)
* xipImage      - XIP kernel image, if configured (arch/arm/boot/xipImage)
  uImage        - U-Boot wrapped zImage
  bootpImage    - Combined zImage and initial RAM disk
                  (supply initrd image via make variable INITRD=<path>)
  install       - Install uncompressed kernel
  zinstall      - Install compressed kernel
  uinstall      - Install U-Boot wrapped compressed kernel
```

**Reading the build output.** Each line is a step and a file: `CC` compiles a built-in object, `CC [M]` a module object, `LD [M]` links a module, `HOSTCC` builds a tool for the build machine, `DTC` compiles a device tree. `make V=1` prints the full commands; `make -s` prints only warnings and errors. A failure stops the build at the end of the running jobs, so the error is often a screen above the last line: search for `error:`.

**Rebuilding part of the tree.** `make` is incremental: after editing a file, the same command recompiles only what depends on it. To build one directory or one object while working on a driver:

```sh
$ make O=~/bbb/build drivers/usb/gadget/legacy/          # one directory
$ make O=~/bbb/build drivers/usb/gadget/legacy/ether.o   # one object
```

Neither relinks `vmlinux` or the `zImage`; run the full command before installing.

## 7. What the build produces

| File (in the build directory) | What it is | Goes to the board? |
|---|---|---|
| `vmlinux` | The kernel as an ELF file, with symbols (and debug information if enabled) | No. Keep it for debugging (section 19) |
| `arch/arm/boot/Image` | `vmlinux` stripped to raw binary with `objcopy` | Not on this board |
| `arch/arm/boot/zImage` | `Image` compressed, behind a small decompressor | Yes |
| `System.map` | Every kernel symbol and its address | No. Used to decode addresses |
| `arch/arm/boot/dts/ti/omap/*.dtb` | The device trees | One of them |
| `**/*.ko` | Modules | Through `modules_install` |
| `modules.order`, `modules.builtin`, `Module.symvers` | The module list in build order, the list of built-in modules, the exported symbols with their CRCs | `modules.order` and `modules.builtin` through `modules_install`; `Module.symvers` stays on the build machine for out-of-tree modules (section 15) |
| `.config`, `include/generated/` | The configuration and the headers generated from it | No |

**How a `zImage` is made.** `vmlinux` is linked for the kernel's virtual address (`0xC0008000` here, from `CONFIG_PAGE_OFFSET=0xC0000000`), `objcopy` turns it into `Image`, the compressor packs `Image`, and the decompressor in `arch/arm/boot/compressed/` is linked in front of it. At boot, the decompressor works out where it runs, unpacks the kernel just above the start of RAM, and jumps to it. The `zImage` is position-independent, which is why U-Boot can load it at `0x82000000` (bootloader note). Its header has the magic `0x016f2818` at offset `0x24`, which `bootz` checks.

**Compression.** `omap2plus_defconfig` chooses `CONFIG_KERNEL_LZMA=y`. The choice trades size for decompression time:

| Option | Size | Decompression | Notes |
|---|---|---|---|
| `KERNEL_GZIP` | Medium | Fast | The safe default on most architectures |
| `KERNEL_LZO`, `KERNEL_LZ4` | Larger | Fastest | When the storage is fast and boot time matters |
| `KERNEL_LZMA`, `KERNEL_XZ` | Smallest | Slowest | When the storage is small or slow |

On an SD card, the time to read a larger image and the time to decompress a smaller one are of the same order: measure on the board before changing it (section 21).

**Appended DTB.** `omap2plus_defconfig` enables `CONFIG_ARM_APPENDED_DTB`: the decompressor uses a DTB concatenated after the `zImage` (`cat zImage am335x-boneblack.dtb > zImage-dtb`) when the bootloader passes none. It's for old bootloaders without device tree support; with U-Boot, pass the DTB separately.

## 8. Version strings and module vermagic

The kernel has two version strings:

| Command | Prints | Built from |
|---|---|---|
| `make -s kernelversion` | `7.1.0-rc1` | `VERSION`, `PATCHLEVEL`, `SUBLEVEL`, `EXTRAVERSION` at the top of the `Makefile` |
| `make -s O=~/bbb/build kernelrelease` | for example `7.1.0-rc1-g66edb901bf87` | the version, plus `localversion*` files, `CONFIG_LOCALVERSION`, and a suffix from Git (`scripts/setlocalversion`) |

The **release** is the one that matters: `uname -r` prints it on the board, it names the module directory (`/lib/modules/7.1.0-rc1-g66edb901bf87/`), and it's part of every module's "vermagic".

**The Git suffix.** With `CONFIG_LOCALVERSION_AUTO=y` (the default in `omap2plus_defconfig`), `scripts/setlocalversion` adds the state of the Git tree, from source:

| The source tree | Suffix |
|---|---|
| Exactly at a release tag (`v7.1-rc1`) | none |
| Commits after a tag | `-<count>-g<hash>`, for example `-00123-g66edb901bf87` |
| A Git tree where the tag can't be found (shallow clone without tags) | `-g<hash>` only |
| Uncommitted changes to tracked files | `-dirty` appended |
| Not a Git tree (a tarball, an exported copy) | none |

The tree of this note was a shallow clone, hence `7.1.0-rc1-g66edb901bf87` (**Real output** of `KERNELVERSION=7.1.0-rc1 scripts/setlocalversion`: `-g66edb901bf87`). The consequence: **the same source built from a tarball and from a Git clone gives two different releases**, two module directories, and modules that don't load on the other kernel. For products, set the string yourself and make it independent of the tree's state: `CONFIG_LOCALVERSION_AUTO=n` and `CONFIG_LOCALVERSION="-bbb1"`.

**Vermagic.** Every module records the release and the configuration options that change the module ABI. Read it with `modinfo`:

```
$ modinfo -F vermagic g_ether.ko
7.1.0-rc1-g66edb901bf87 SMP mod_unload modversions ARMv7 p2v8
```

(**Expected**: the format is from `include/linux/vermagic.h` and `arch/arm/include/asm/module.h`; `SMP`, `mod_unload` and `modversions` follow `CONFIG_SMP`, `CONFIG_MODULE_UNLOAD` and `CONFIG_MODVERSIONS`, all set in `omap2plus_defconfig`; `ARMv7` and `p2v8` come from the CPU architecture and the physical-to-virtual patching.) When the module's vermagic doesn't match the running kernel's, loading fails. From source (`kernel/module/main.c`, `kernel/module/version.c`), the kernel log says one of:

```
g_ether: version magic '7.1.0-rc1 SMP mod_unload modversions ARMv7 p2v8' should be '7.1.0-rc1-g66edb901bf87 SMP mod_unload modversions ARMv7 p2v8'
g_ether: disagrees about version of symbol <symbol>
```

and `modprobe` or `insmod` reports `Invalid module format`. The first is a release or configuration mismatch; the second (with `CONFIG_MODVERSIONS`) means the module was built against a kernel whose exported function had a different signature. Both mean the same thing in practice: **build modules from the same tree, configuration and release as the kernel they run on**, and install them together (section 9).

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
