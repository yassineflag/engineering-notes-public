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

The SD card layout is the one of the bootloader note: partition 1 is FAT (with `MLO`, `u-boot.img` and the boot files), partition 2 is the ext4 root filesystem. The kernel and DTB go on partition 1 next to `extlinux/extlinux.conf`, the modules go on partition 2.

```sh
$ B=~/bbb/build
$ sudo mount /dev/sdX1 /mnt/boot
$ sudo mount /dev/sdX2 /mnt/rootfs

# kernel and device tree, keeping the previous ones as a fallback
$ sudo cp /mnt/boot/zImage /mnt/boot/zImage.old
$ sudo cp /mnt/boot/am335x-boneblack.dtb /mnt/boot/am335x-boneblack.dtb.old
$ sudo cp $B/arch/arm/boot/zImage /mnt/boot/
$ sudo cp $B/arch/arm/boot/dts/ti/omap/am335x-boneblack.dtb /mnt/boot/

# modules: install into a staging directory as a normal user, then copy
$ make O=$B INSTALL_MOD_PATH=$HOME/bbb/staging INSTALL_MOD_STRIP=1 modules_install
$ sudo cp -a $HOME/bbb/staging/lib/modules /mnt/rootfs/lib/

$ sudo umount /mnt/boot /mnt/rootfs
```

- **`modules_install` copies every `.ko` into `<INSTALL_MOD_PATH>/lib/modules/<release>/kernel/`**, keeping the source tree's layout, copies `modules.order` and `modules.builtin`, then runs the build machine's `depmod` to generate `modules.dep`, `modules.alias` and the other index files (section 14). Without `INSTALL_MOD_PATH` it installs into the build machine's own `/lib/modules`, which is never what you want for a cross build.
- **Don't run `sudo make` without the variables.** `sudo` resets the environment, so the exported `ARCH` and `CROSS_COMPILE` disappear and `make` sees an x86 build. Either install as a normal user into a staging directory as above, or put the variables on the command line: `sudo make ARCH=arm CROSS_COMPILE=arm-none-linux-gnueabihf- O=$B ...`.
- **`INSTALL_MOD_STRIP=1`** strips debug symbols from the installed copies. With debug information enabled, it divides the size of `/lib/modules` several times over. The unstripped `.ko` files stay in the build directory for debugging.
- **Install the modules every time you install a kernel**, even when you only changed one driver: a new release string (section 8) means a new module directory, and the old one doesn't load.
- The `build` symbolic link in `/lib/modules/<release>/` points to the build directory on the build machine. On the board it dangles; it's harmless, and it's what lets a native build of out-of-tree modules find the kernel (section 15).

The `extlinux.conf` from the bootloader note boots these files, with the previous kernel as a second menu entry:

```
label current
    kernel /zImage
    fdt /am335x-boneblack.dtb
    append console=ttyS0,115200n8 root=/dev/mmcblk0p2 rw rootfstype=ext4 rootwait

label previous
    kernel /zImage.old
    fdt /am335x-boneblack.dtb.old
    append console=ttyS0,115200n8 root=/dev/mmcblk0p2 rw rootfstype=ext4 rootwait
```

**Other install targets:**

| Command | Installs | Use |
|---|---|---|
| `make dtbs_install INSTALL_DTBS_PATH=/mnt/rootfs/boot/dtbs/<release>` | Every DTB built, in their `ti/omap/` subdirectories | Distribution-style layouts. Name the DTB with its subdirectory in `extlinux.conf` (`fdt /boot/dtbs/<release>/ti/omap/am335x-boneblack.dtb`); `fdtdir` doesn't look into subdirectories (bootloader note) |
| `make headers_install INSTALL_HDR_PATH=<sysroot>/usr` | The UAPI headers (`linux/*.h`, `asm/*.h`) | Building a toolchain or a sysroot. The rule from the [toolchain note](../cross-compilation-toolchain/cross-compilation-toolchain.md): the headers must be no newer than the oldest kernel the programs run on |
| `make zinstall INSTALL_PATH=...` | `zImage`, `System.map` and `.config` with the release in their names | Runs `installkernel` if found; rarely useful for a board |

## 10. The kernel command line

The command line is a string of space-separated parameters that configures the kernel at boot, and part of it is passed on to `init`. On a device tree system it travels in the DTB.

**Where it comes from**, from source (`drivers/of/fdt.c`, `arch/arm/Kconfig`):

| Source | When it's used |
|---|---|
| U-Boot's `bootargs` variable | U-Boot writes it into `/chosen/bootargs` of the DTB it passes, replacing what the DTB had (bootloader note) |
| `/chosen/bootargs` in the `.dts` | When the bootloader doesn't set it |
| `CONFIG_CMDLINE` | With the default `CONFIG_CMDLINE_FROM_BOOTLOADER`: only when the two above give an empty string. `CONFIG_CMDLINE_EXTEND` appends it, `CONFIG_CMDLINE_FORCE` replaces whatever the bootloader passed |

**The trap in `omap2plus_defconfig`.** Its `CONFIG_CMDLINE` is `root=/dev/mmcblk0p2 rootwait console=ttyO2,115200`. It was written for another OMAP board: on a BeagleBone Black, UART2 isn't the console. If U-Boot boots the kernel without setting `bootargs` (a hand-typed `bootz` after `env default -a`, for example), the kernel uses this string, sends its console to UART2, and J1 stays silent after `Starting kernel ...`. The device tree's `stdout-path` doesn't help: a `console=` on the command line takes priority over it. Always set `bootargs` (or `append` in `extlinux.conf`), or set your own `CONFIG_CMDLINE` in a fragment.

**The parameters that matter on this board** (all in `Documentation/admin-guide/kernel-parameters.txt`):

| Parameter | Effect |
|---|---|
| `console=ttyS0,115200n8` | Kernel messages and `/dev/console` on UART0, the J1 header. Several `console=` are allowed; the last one becomes `/dev/console`. Use `ttyS0`, not the old `ttyO0` (bootloader note) |
| `earlycon=uart8250,mmio32,0x44e09000,115200n8` | Messages from the first milliseconds, before the real console driver (section 18) |
| `root=/dev/mmcblk0p2` or `root=PARTUUID=<uuid>` | The root filesystem. `mmcblk0` is the SD card, `mmcblk1` the eMMC (section 12); the partition UUID doesn't depend on the numbering |
| `rootfstype=ext4` | Skips trying every filesystem type |
| `rootwait`, `rootwait=<seconds>` | Waits for the root device to appear instead of failing at once. Needed for SD cards and USB, which are detected asynchronously |
| `ro` / `rw` | Mounts the root read-only (the default; `init` scripts remount it after `fsck`) or read-write |
| `init=/sbin/init` | The first program (section 13). `init=/bin/sh` gives a shell with no init scripts, for repairs |
| `rdinit=/init` | The first program in an initramfs |
| `loglevel=<n>`, `quiet`, `ignore_loglevel` | Which messages reach the console. `quiet` is `loglevel=4`; `ignore_loglevel` prints everything, for debugging |
| `panic=10` | Reboots 10 seconds after a panic instead of hanging. Pairs with the boot counter of the bootloader note |
| `ip=...`, `nfsroot=...`, `root=/dev/nfs` | NFS root (bootloader note, network boot) |
| `<module>.<parameter>=<value>` | A module parameter, for built-in drivers and modules loaded later (section 14) |
| `initcall_debug` | Logs every initcall and its duration (section 21) |

**What the kernel doesn't recognize goes to `init`.** From source (`init/main.c`): parameters without `=` that the kernel doesn't know become arguments of `init`, those with `=` become its environment variables, and the kernel logs `Unknown kernel command line parameters "...", will be passed to user space.` (`single` for a single-user boot works this way.) A misspelled kernel parameter ends up there too, so read that line when an option seems ignored.

On the board, the line the kernel actually used is in `/proc/cmdline`.

## 11. The boot, stage by stage, and its log

What follows is the console of a BeagleBone Black booting this kernel from the SD card. **Expected**: built from the format strings in the source, with `...` for values that depend on the build and the board, and only the lines worth knowing. Timestamps come from `CONFIG_PRINTK_TIME=y`, in seconds since the kernel started.

```
Starting kernel ...

[    0.000000] Booting Linux on physical CPU 0x0
[    0.000000] Linux version 7.1.0-rc1-g66edb901bf87 (me@build) (arm-none-linux-gnueabihf-gcc (Arm GNU Toolchain 15.2.Rel1 ...) 15.2.1 ..., ...) #1 SMP ...
[    0.000000] CPU: ARMv7 Processor [...] revision ... (ARMv7), cr=...
[    0.000000] OF: fdt: Machine model: TI AM335x BeagleBone Black
...
[    0.000000] Kernel command line: console=ttyS0,115200n8 root=/dev/mmcblk0p2 rw rootfstype=ext4 rootwait
...
[    0.000000] Memory: ...K/524288K available (...K kernel code, ...K rwdata, ...K rodata, ...K init, ...K bss, ...K reserved, ...K cma-reserved, ...K highmem)
...
[    ...] 44e09000.serial: ttyS0 at MMIO 0x44e09000 (irq = ..., base_baud = 3000000) is a 8250
[    ...] printk: legacy console [ttyS0] enabled
...
[    ...] mmc0: new high speed SDHC card at address ...
[    ...] mmcblk0: mmc0:... ... GiB
[    ...]  mmcblk0: p1 p2
...
[    ...] Waiting for root device /dev/mmcblk0p2...
[    ...] EXT4-fs (mmcblk0p2): mounted filesystem ... r/w with ordered data mode. ...
[    ...] VFS: Mounted root (ext4 filesystem) on device 179:2.
[    ...] devtmpfs: mounted
[    ...] VFS: Pivoted into new rootfs
[    ...] Freeing unused kernel image (initmem) memory: ...K
[    ...] Run /sbin/init as init process
```

Reading it stage by stage:

1. **Between `Starting kernel ...` and the first line, nothing.** The decompressor can print `Uncompressing Linux... done, booting the kernel.` (`arch/arm/boot/compressed/misc.c`), but only with `CONFIG_DEBUG_UNCOMPRESS`, which depends on `!ARCH_MULTIPLATFORM` (`arch/arm/Kconfig.debug`). `omap2plus_defconfig` builds a multi-platform kernel, so this kernel's decompressor is silent. The first lines are printed into a memory buffer and only appear once a console exists; the gap before them is normal.
2. **`setup_arch()` reads the DTB.** `Machine model:` is the `model` property of the root node (section 12). `Kernel command line:` is the final command line of section 10: check it first when something behaves differently from what `bootargs` says.
3. **`Memory:`** reports the RAM the kernel manages: `524288K` is the 512 MiB from the DTB's memory node (U-Boot fixes it to the real size). With `CONFIG_HIGHMEM` and the 3 GB/1 GB split of `CONFIG_VMSPLIT_3G`, the part above the kernel's direct mapping is reported as highmem.
4. **The console comes up** when the 8250 driver probes UART0: `ttyS0 at MMIO 0x44e09000` is UART0's address, and `base_baud = 3000000` is its 48 MHz clock divided by 16. `printk: legacy console [ttyS0] enabled` (`kernel/printk/internal.h`; `legacy` marks a console driver not yet converted to the newer `nbcon` interface) is the moment everything buffered so far is printed. Before this line, only `earlycon` can show anything (section 18).
5. **Drivers probe** as their initcalls run, in an order that isn't guaranteed. The SD card appears as `mmcblk0`, with its partitions listed.
6. **The root filesystem.** `Waiting for root device` appears when `rootwait` is set and the card isn't detected yet (`init/do_mounts.c`). `179:2` is the device number: major 179 is MMC, minor 2 the second partition of the first card. Then devtmpfs is mounted on `/dev`, and the kernel moves the mounted root to `/` (`VFS: Pivoted into new rootfs`, in this tree's `prepare_namespace()`).
7. **`Freeing unused kernel image (initmem) memory`**: code and data used only during boot are released. Then `Run /sbin/init as init process` (`init/main.c`): the kernel's boot is over, and the next lines come from `init`.

Everything printed since power-on is still in the kernel's log buffer once the system is up: `dmesg` on the board shows it, including the lines printed before the console existed.

## 12. The device tree from the kernel's side

The bootloader note looked at the DTB as a file that U-Boot loads and edits. The kernel reads it as the list of hardware that exists on the board.

**What the DTB in use says**, inspected on the build machine (**Real output**, on the `am335x-boneblack.dtb` of the earlier build):

```
$ fdtget -t s am335x-boneblack.dtb / model
TI AM335x BeagleBone Black
$ fdtget -t s am335x-boneblack.dtb / compatible
ti,am335x-bone-black ti,am335x-bone ti,am33xx
$ fdtget -t s am335x-boneblack.dtb /chosen stdout-path
serial0:115200n8
$ fdtget -t s am335x-boneblack.dtb /aliases serial0
/ocp/interconnect@44c00000/segment@200000/target-module@9000/serial@0
$ fdtget -t x am335x-boneblack.dtb /memory@80000000 reg
80000000 20000000
```

**How the kernel uses it**, from source:

- **The machine.** The root `compatible` list goes from the most specific to the most generic. The kernel looks for a machine description that matches any entry: `arch/arm/mach-omap2/board-generic.c` declares `DT_MACHINE_START(AM33XX_DT, "Generic AM33XX (Flattened Device Tree)")` for `ti,am33xx`. There is no BeagleBone-specific C code: the board is entirely described by the tree.
- **The drivers.** Each node with a `compatible` property becomes a device, and a driver binds to it when its `of_match_table` lists the same string. Three examples on this board:

  | Node | `compatible` | Driver |
  |---|---|---|
  | UART0 | `ti,am3352-uart` | `drivers/tty/serial/8250/8250_omap.c` |
  | SD card controller (MMC0) | `ti,am335-sdhci` | `drivers/mmc/host/sdhci-omap.c` |
  | Ethernet switch | `ti,am335x-cpsw-switch`, `ti,cpsw-switch` | `drivers/net/ethernet/ti/cpsw_new.c` |

  When a device doesn't work, `grep -r '"<compatible>"' drivers/` finds the driver, and its Kconfig symbol tells you what to enable (section 5).
- **`status`.** A node with `status = "disabled"` gets no device. SoC `.dtsi` files disable most peripherals, and the board `.dts` enables the ones wired on the board with `status = "okay"`.
- **`aliases` fix the numbering.** `serial0 = &uart0` makes UART0 `ttyS0`. In `am33xx.dtsi`, `mmc0 = &mmc1` and `mmc1 = &mmc2` make the SD card slot (the SoC's first controller) `mmcblk0` and the eMMC `mmcblk1`, whatever order they probe in. `ethernet0` names the first Ethernet port.
- **`/chosen`.** `bootargs` is the command line (section 10); `stdout-path` names the console when the command line has no `console=`.

On the board, the tree the kernel actually received, after U-Boot's changes, is visible as files:

```sh
# cat /proc/device-tree/model; echo
# ls /proc/device-tree/chosen/
# dtc -I fs -O dts /sys/firmware/devicetree/base > running.dts    # needs dtc on the board
```

**A device tree for your own board.** A product based on the BeagleBone Black, or a cape that is always fitted, gets its own `.dts` that includes the board's and changes it. For example, an SSD1306 OLED display on I2C2 (the bus on header pins P9_19 and P9_20):

```dts
// arch/arm/boot/dts/ti/omap/am335x-boneblack-myproduct.dts
/dts-v1/;
#include "am335x-boneblack.dts"

/ {
	model = "My product (BeagleBone Black)";
};

&i2c2 {
	oled@3c {
		compatible = "solomon,ssd1306fb-i2c";
		reg = <0x3c>;
	};
};
```

Then add it to `arch/arm/boot/dts/ti/omap/Makefile`, next to the BeagleBone Black's lines:

```make
dtb-$(CONFIG_SOC_AM33XX) += am335x-boneblack-myproduct.dtb
DTC_FLAGS_am335x-boneblack-myproduct += -@
```

and build it alone with `make O=$B ti/omap/am335x-boneblack-myproduct.dtb`. `-@` keeps the label symbols, so overlays can still be applied on top (bootloader note). I2C2 is already enabled in `am335x-bone-common.dtsi` (it carries the cape EEPROMs), so only the child node is new; for a peripheral on other pins, the pin multiplexing (`pinctrl-0`, `AM33XX_PADCONF`) has to be added too. The display also needs its driver enabled in the configuration, which is the "three things must agree" of section 1. `make dtbs_check` validates trees against the binding schemas in `Documentation/devicetree/bindings/` (it needs the `dtschema` Python package).

## 13. Root filesystem, initramfs and init

**The order**, from source (`init/main.c`, `init/initramfs.c`, `init/do_mounts.c`):

1. The kernel always starts with a RAM filesystem as `/` (rootfs) and unpacks an initramfs into it: the one built into the kernel, then the one the bootloader passed, if any. When nothing was configured, the built-in one is the default list in `usr/default_cpio_list`:

   ```
   dir /dev 0755 0 0
   nod /dev/console 0600 0 0 c 5 1
   dir /root 0700 0 0
   ```

2. It opens `/dev/console` there as standard input, output and error for the first process. Without it: `Warning: unable to open an initial console.`
3. **If `/init` exists in that RAM filesystem** (or the program named by `rdinit=`), the kernel runs it and is done. Mounting the real root filesystem is then the initramfs's job (`switch_root`).
4. **Otherwise it mounts the root itself** (`prepare_namespace()`): waits `rootdelay=` seconds if given, waits for drivers to finish probing, parses `root=`, waits for the device with `rootwait`, mounts it, mounts devtmpfs on its `/dev`, and makes it `/`.
5. **It runs the first program**: `init=` if given (a failure panics with `Requested init /sbin/xyz failed (error -2).`), else `CONFIG_DEFAULT_INIT` if set, else the first of `/sbin/init`, `/etc/init`, `/bin/init`, `/bin/sh` that starts. If none does: `Kernel panic - not syncing: No working init found.  Try passing init= option to kernel. See Linux Documentation/admin-guide/init.rst for guidance.`

**What the kernel needs from the root filesystem**, which is the boundary with the [rootfs note](../rootfs/):

- The **filesystem driver built in** (`CONFIG_EXT4_FS=y` here), and the **storage driver built in** (`CONFIG_MMC_SDHCI_OMAP=y`). A module can't be loaded from a filesystem that isn't mounted yet; only an initramfs can load storage drivers as modules.
- An **empty `/dev` directory**, where the kernel mounts devtmpfs (`CONFIG_DEVTMPFS_MOUNT=y`). Without it, `devtmpfs: error mounting -2` (`drivers/base/devtmpfs.c`; `-2` is `ENOENT`), and `init` starts with an empty `/dev`.
- An **`init` that can run on this CPU**: an ARM binary, and, if dynamically linked, the right dynamic loader and C library. When the loader is missing, `exec` fails with `ENOENT` (error -2) even though `/sbin/init` exists; the [toolchain note](../cross-compilation-toolchain/cross-compilation-toolchain.md) explains why.
- **`/proc` and `/sys` directories**, which `init` scripts mount.

**When the root can't be mounted**, from source (`init/do_mounts.c`), the kernel prints `VFS: Cannot open root device "..." or unknown-block(...): error ...`, then `Please append a correct "root=" boot option; here are the available partitions:` followed by the block devices it knows, then panics with `VFS: Unable to mount root fs on ...`. That list is the most useful line of the log: if `mmcblk0p2` isn't in it, the storage driver or the device tree is the problem; if it is, check `root=` and `rootfstype=`. An unparsable `root=` with `rootwait` prints `Disabling rootwait; root= is invalid.` instead of waiting forever.

**An initramfs** is a cpio archive of a small root filesystem. Two ways to use one:

| | Built into the kernel | Loaded by the bootloader |
|---|---|---|
| How | `CONFIG_INITRAMFS_SOURCE` names a directory or a file list; the build packs it into `vmlinux` | A separate file; U-Boot loads it and passes its address in `/chosen` (`bootz <kernel> <initrd> <fdt>`) |
| Updating it | Rebuild the kernel | Replace the file |
| Typical use | Small systems that run entirely from RAM; recovery images | Distribution-style early boot (decrypt, assemble RAID, then `switch_root`) |

An example from the earlier BeagleBone work: an initramfs wrapped as a legacy U-Boot image (**Real output**):

```
$ mkimage -l initramfs
Image Name:   Root Filesystem
Created:      Sun Jun 14 19:50:37 2026
Image Type:   ARM Linux RAMDisk Image (uncompressed)
Data Size:    3098001 Bytes = 3025.39 KiB = 2.95 MiB
Load Address: 80800000
Entry Point:  80800000
$ dd if=initramfs bs=64 skip=1 2>/dev/null | file -
/dev/stdin: gzip compressed data, from Unix
$ dd if=initramfs bs=64 skip=1 2>/dev/null | zcat | cpio -tv 2>/dev/null | grep -E ' (init|sbin/init)( |$)'
lrwxrwxrwx   1 ...       10 May 12  2026 init -> /sbin/init
lrwxrwxrwx   1 ...       19 May 12  2026 sbin/init -> /sbin/init.sysvinit
```

The payload after the 64-byte header is a gzip-compressed cpio archive of 849 entries ("RAMDisk Image (uncompressed)" describes the U-Boot wrapper, not the content; the kernel decompresses gzip itself with `CONFIG_RD_GZIP=y`). It contains `/init`, so the kernel runs it (step 3) and ignores `root=`: that init, SysV init here, runs from RAM. With a legacy image U-Boot knows the size from the header (`bootz ${kernel_addr_r} ${ramdisk_addr_r} ${fdt_addr_r}`); a raw cpio file needs `${ramdisk_addr_r}:${filesize}`.

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
