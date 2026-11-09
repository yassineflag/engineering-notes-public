# Bootloaders for Embedded Linux: U-Boot

A beginner's path from power-on to a running Linux kernel: what a bootloader does, how to build U-Boot, how to use its command line in QEMU, and how to put it on a BeagleBone Black or a Raspberry Pi 4.

**You need:** the PC, toolchains and settings files from the [cross-compilation note](../cross-compilation-toolchain/cross-compilation-toolchain.md) (sections 3 and 4). No board is needed until section 9. For a board, you also need a 3.3 V USB-to-serial cable.

**How to read it:**
- **Parts 1 and 2 are essential.** Read them in order and type the commands.
- **Part 3 is optional.** Each section says when you need it. If something fails, start at [section 10](#10-troubleshooting).

**How commands are labelled:**

| Label | Meaning |
|---|---|
| **Expected (QEMU)** | **Not run for this note.** Written for Ubuntu 22.04 (GCC 11.4, QEMU 6.2) and U-Boot v2026.10. The output is what U-Boot's source code and documentation say you should see, shortened. Dates, addresses, sizes and timings will differ |
| **Expected (board)** | **Not run on real hardware.** The commands are what you should type, and the output is what you should see, based on the board's documentation and U-Boot's source |
| **Illustrative** | Shows the form of a command, file or message. Not meant to be typed as is |

## Table of contents

**Part 1: Concepts (essential)**

- [1. What a bootloader does](#1-what-a-bootloader-does)
- [2. The words you need](#2-the-words-you-need)
- [3. How each board boots](#3-how-each-board-boots)

**Part 2: A working example (essential)**

- [4. Get and build U-Boot](#4-get-and-build-u-boot)
- [5. First boot in QEMU](#5-first-boot-in-qemu)
- [6. The environment](#6-the-environment)
- [7. Boot Linux from U-Boot](#7-boot-linux-from-u-boot)
- [8. Make it boot by itself](#8-make-it-boot-by-itself)
- [9. Run it on a board](#9-run-it-on-a-board)

**Part 3: Going further (optional)**

- [10. Troubleshooting](#10-troubleshooting)
- [11. Memory and load addresses](#11-memory-and-load-addresses)
- [12. Kernel image formats and FIT](#12-kernel-image-formats-and-fit)
- [13. The device tree in the bootloader](#13-the-device-tree-in-the-bootloader)
- [14. Booting over the network](#14-booting-over-the-network)
- [15. The environment from Linux](#15-the-environment-from-linux)
- [16. Safe updates and verified boot](#16-safe-updates-and-verified-boot)
- [17. Debugging U-Boot](#17-debugging-u-boot)
- [18. Where the rest is covered](#18-where-the-rest-is-covered)
- [References](#references)

---

## 1. What a bootloader does

When a board is powered on, its main memory (DRAM) doesn't work yet, and nothing knows where Linux is stored. The Linux kernel can't start itself. Something has to prepare the hardware, copy the kernel into RAM, tell it about the board, and jump to it. That is the **bootloader**.

Its job, in four steps:

| Step | In plain words | In U-Boot |
|---|---|---|
| **1. Bring up the hardware** | Set the clocks, start the DRAM controller so RAM can be used, start a serial port to print messages | Done by the SPL on the BeagleBone, by the firmware on the Raspberry Pi |
| **2. Find the files** | Read the kernel, the device tree and maybe an initramfs from an SD card, eMMC, USB or the network | `load`, `tftp` |
| **3. Prepare the hand-off** | Write the kernel command line and the RAM size into the device tree | Automatic, from the `bootargs` variable |
| **4. Jump to the kernel** | Start the kernel with the CPU in the state Linux expects | `bootz`, `booti`, `bootm` |

After step 4 the bootloader is gone: Linux takes over the whole RAM, including where U-Boot was.

**Why you should care.** The bootloader is the first thing you can change on a board. When the serial port shows nothing, or Linux stops right after `Starting kernel ...`, the cause is very often here. It is also the tool you use to test a new kernel without rewriting the SD card, to boot over the network, and to recover a board.

This note uses **U-Boot**, the bootloader of most embedded Linux boards, including both boards here.

---

## 2. The words you need

| Word | Meaning |
|---|---|
| **Boot ROM** | Code written into the chip at the factory. It runs first, and can only load a small program from a few fixed places. You can't change it |
| **SPL** (secondary program loader) | A small first stage of U-Boot that fits in the chip's internal memory (SRAM). It starts the DRAM, then loads full U-Boot into it |
| **MLO** | The file name the BeagleBone's boot ROM looks for on an SD card. It holds the SPL |
| **U-Boot proper** | The full U-Boot, with drivers and a command line. Usually just called U-Boot |
| **Firmware** | On the Raspberry Pi, closed-source programs from Raspberry Pi Ltd that run before U-Boot or Linux |
| **Serial console** | The board's serial port (UART), connected to the PC with a USB-to-serial cable. On most boards it is the only way to talk to the bootloader |
| **Environment** | U-Boot's variables, as `name=value` text. They can be saved to storage |
| **`bootcmd`** | The variable U-Boot runs when the countdown ends |
| **`bootargs`** | The kernel command line. Linux shows it in `/proc/cmdline` |
| **Device tree** | A description of the board's hardware for the kernel: which devices exist and at which addresses. The source is a `.dts` file, the compiled form a `.dtb` |
| **Load address** | The address in RAM where a file is copied before it is used |
| **Kernel image** | The kernel file. On 32-bit Arm it is a **zImage**, which decompresses itself. On 64-bit Arm it is an **Image** |
| **initramfs** (or initrd) | A small filesystem loaded into RAM with the kernel. Linux uses it before, or instead of, a root filesystem on disk |
| **TFTP** | A very simple file transfer protocol. U-Boot uses it to load files from the PC over the network |
| **defconfig** | A file that selects U-Boot's options for one board, such as `am335x_evm_defconfig` |

---

## 3. How each board boots

Each board has a different chain before Linux starts. QEMU is the third "board", and the simplest.

```
BeagleBone Black              Raspberry Pi 4                  QEMU virt
────────────────              ──────────────                  ─────────
Boot ROM in the AM3358        Boot ROM in the BCM2711         QEMU puts u-boot.bin in
  │ finds MLO on SD or eMMC     │ runs the bootloader stored    emulated flash, a device
  ▼                             │ in an EEPROM on the board     tree in RAM, and starts
SPL (MLO), in internal SRAM     ▼                               the CPU at address 0
  │ starts DRAM               EEPROM bootloader                    │
  │ loads u-boot.img            │ reads the SD card's              │
  ▼                             │ FAT partition                    │
U-Boot                          ▼                                  │
  │ loads kernel and DTB      GPU firmware (start4.elf)            │
  ▼                             │ reads config.txt, cmdline.txt    │
Linux                           │ loads the DTB and the kernel     ▼
                                ▼                               U-Boot
                              Linux, or U-Boot then Linux          │
                                                                   ▼
                                                                Linux
```

**The Raspberry Pi 4 doesn't need U-Boot.** Its firmware can start Linux directly, and Raspberry Pi OS does this by default. You add U-Boot for a command line, network boot, boot scripts or safe updates, and to boot the same way as other boards.

| | BeagleBone Black | Raspberry Pi 4 | QEMU |
|---|---|---|---|
| First code you can replace | MLO (the SPL) | `config.txt` and the kernel file | `u-boot.bin` |
| Who starts the DRAM | The SPL | The firmware | Nobody needs to, RAM just works |
| Where U-Boot is stored | `MLO` and `u-boot.img` on the SD card's FAT partition, or in eMMC | `u-boot.bin` on the FAT boot partition, loaded by the firmware as if it were the kernel | Emulated flash |
| Where the device tree comes from | A `.dtb` file that U-Boot loads | The firmware, from `bcm2711-rpi-4-b.dtb` plus overlays | QEMU generates it |
| U-Boot defconfig | `am335x_evm_defconfig` | `rpi_4_defconfig` | `qemu_arm_defconfig` (32-bit) or `qemu_arm64_defconfig` (64-bit) |
| Command that starts the kernel | `bootz` | `booti` | `bootz` (32-bit) or `booti` (64-bit) |
| Serial console | J1 header, 115200 baud | GPIO 14 and 15 (pins 8 and 10), 115200 baud | Your terminal |

**What QEMU proves, and what it doesn't.** QEMU's `virt` machine is a generic Arm computer, not an emulated AM3358 or BCM2711. It shows how U-Boot works: its commands, its environment, and how it starts Linux. It can't test the board's SPL, DRAM, SD card or serial wiring. For those you need the board.

---

## 4. Get and build U-Boot

Install the tools U-Boot needs to build, plus QEMU and `mkimage`:

```sh
sudo apt install bc bison flex device-tree-compiler libssl-dev libgnutls28-dev \
    python3-dev python3-setuptools python3-pyelftools swig uuid-dev \
    u-boot-tools qemu-system-arm wget
git clone --depth 1 -b v2026.10 https://git.u-boot-project.org/u-boot/u-boot.git ~/boot/u-boot
```

`v2026.10` is the U-Boot release from October 2026. U-Boot publishes one every three months. Older guides point to `source.denx.de`, which now redirects to `git.u-boot-project.org`. The `qemu-system-arm` package installs both `qemu-system-arm` and `qemu-system-aarch64`.

Add these lines to the settings files from the cross-compilation note.

At the end of `~/cross/target-bbb.sh`:

```sh
# Bootloader note
export BOOTDIR=$HOME/boot/bbb
export UBOOT_QEMU=qemu_arm_defconfig
export UBOOT_BOARD=am335x_evm_defconfig
export QEMU_SYS="qemu-system-arm -M virt -m 1G -nographic"
export NETBOOT=https://deb.debian.org/debian/dists/trixie/main/installer-armhf/current/images/netboot
export NETBOOT_KERNEL=vmlinuz
```

At the end of `~/cross/target-pi4.sh`:

```sh
# Bootloader note
export BOOTDIR=$HOME/boot/pi4
export UBOOT_QEMU=qemu_arm64_defconfig
export UBOOT_BOARD=rpi_4_defconfig
export QEMU_SYS="qemu-system-aarch64 -M virt -cpu cortex-a72 -m 1G -nographic"
export NETBOOT=https://deb.debian.org/debian/dists/trixie/main/installer-arm64/current/images/netboot/debian-installer/arm64
export NETBOOT_KERNEL=linux
```

`BOOTDIR` holds everything built for one target, so the two never mix. `NETBOOT` is used in section 7. Load your settings file, as in the first note:

```sh
. ~/cross/target-bbb.sh
mkdir -p $BOOTDIR
```

Build U-Boot for QEMU. **Expected (QEMU):**

```console
$ cd ~/boot/u-boot
$ make O=$BOOTDIR/build-qemu $UBOOT_QEMU
#
# configuration written to .config
#
$ make O=$BOOTDIR/build-qemu -j$(nproc)
```

The build takes a minute or two. What the commands do:

| Part | Why |
|---|---|
| `CROSS_COMPILE` | Set by your settings file. U-Boot's Makefile reads it, as the kernel's does. The defconfig selects the architecture |
| `$UBOOT_QEMU` | The defconfig: writes `.config` with this board's options |
| `O=$BOOTDIR/build-qemu` | Puts the build output in its own folder, so the source tree stays clean and each configuration has its own folder |

The two files that matter in `$BOOTDIR/build-qemu`:

| File | What it is |
|---|---|
| `u-boot.bin` | The raw binary that the machine runs. **This is what you install** |
| `u-boot` | The same program as an ELF file with symbols, for the debugger (section 17) |

To change options, run `make O=$BOOTDIR/build-qemu menuconfig`, the same menu as the kernel's.

**U-Boot doesn't use the C library.** It runs before any operating system exists, so the glibc version rule of the cross-compilation note doesn't apply. Any recent cross compiler for the right architecture can build it.

---

## 5. First boot in QEMU

Save this script as `~/boot/run-qemu.sh`, and make it executable with `chmod +x ~/boot/run-qemu.sh`:

```sh
#!/bin/sh
# Start U-Boot in QEMU, for the target chosen by the settings file
cd "$BOOTDIR" || exit 1
[ -f envstore.img ] || truncate -s 64M envstore.img
mkdir -p tftp
exec $QEMU_SYS -bios build-qemu/u-boot.bin \
    -drive if=pflash,format=raw,index=1,file=envstore.img \
    -nic user,model=e1000,tftp=tftp "$@"
```

| Option | Why |
|---|---|
| `-M virt` | QEMU's generic Arm machine, well supported by U-Boot and Linux |
| `-cpu cortex-a72` | Pi settings only. Without it, `qemu-system-aarch64` starts a 32-bit CPU. The 32-bit `virt` machine can't emulate the BeagleBone's Cortex-A8, so it keeps its default, a Cortex-A15 |
| `-m 1G` | 1 GiB of RAM |
| `-nographic` | The board's serial port is your terminal. **Press Ctrl-A, then X, to quit** |
| `-bios u-boot.bin` | QEMU puts U-Boot in the emulated flash at address 0 and starts the CPU there, the way a boot ROM starts its first program |
| `-drive if=pflash,index=1,...` | A second flash chip, backed by the file `envstore.img`, where U-Boot saves its variables (section 6) |
| `-nic user,model=e1000,tftp=tftp` | A network card. QEMU provides a DHCP server and a TFTP server that serves the `tftp` folder (section 7) |

Start it, and **press a key during the countdown**. **Expected (QEMU)**, BeagleBone Black settings, shortened:

```console
$ ~/boot/run-qemu.sh

U-Boot 2026.10 (Nov 09 2026 - 21:14:03 +0100)

DRAM:  1 GiB
...
Loading Environment from Flash... *** Warning - bad CRC, using default environment

...
Hit any key to stop autoboot:  0
=>
```

`=>` is U-Boot's prompt. The warning is normal the first time: nothing has been saved yet, so U-Boot uses the variables it was built with. If you miss the countdown, U-Boot tries to boot, finds nothing, and shows the prompt anyway.

Try a few commands:

```console
=> version
U-Boot 2026.10 (Nov 09 2026 - 21:14:03 +0100)

arm-linux-gnueabihf-gcc (Ubuntu 11.4.0-1ubuntu1~22.04.3) 11.4.0
GNU ld (GNU Binutils for Ubuntu) 2.38
=> bdinfo
...
DRAM bank   = 0x00000000
-> start    = 0x40000000
-> size     = 0x40000000
...
relocaddr   = 0x7f...
...
=> md.b 40000000 4
40000000: d0 0d fe ed                                      ....
```

| Command | What it shows |
|---|---|
| `version` | U-Boot's version, and the toolchain that built it |
| `bdinfo` | Board information. RAM starts at `0x40000000` on QEMU's `virt` machine. `relocaddr` is where U-Boot moved itself, near the top of RAM, to leave the rest free for Linux |
| `md.b 40000000 4` | **Memory display**: four bytes at that address. `d0 0d fe ed` is the start of every device tree, the one QEMU placed at the start of RAM |
| `help`, `help bootz` | The list of commands, and help for one |
| `reset` | Restart the machine |

U-Boot reads all numbers as **hexadecimal**, with or without `0x`. `md 1000` means address `0x1000`.

---

## 6. The environment

U-Boot's variables control everything it does. They live in RAM while it runs, and can be saved to storage.

**Expected (QEMU)**, BeagleBone Black settings:

```console
=> printenv kernel_addr_r fdt_addr ramdisk_addr_r
kernel_addr_r=0x40400000
fdt_addr=0x40000000
ramdisk_addr_r=0x44000000
=> printenv bootcmd
bootcmd=bootflow scan -lb
=> setenv hello 'echo kernel goes to ${kernel_addr_r}'
=> run hello
kernel goes to 0x40400000
=> setenv hello
```

| Command | Effect |
|---|---|
| `printenv [name ...]` | Show all variables, or the ones named |
| `setenv name value` | Set a variable. **With no value, it deletes it** |
| `run name` | Run the commands stored in a variable |
| `saveenv` | Save all variables to storage |
| `env default -a` | Go back to the built-in variables, in RAM only until you `saveenv` |

**Quotes matter.** In single quotes, `${kernel_addr_r}` is stored as text and replaced when the variable runs. Without quotes, it's replaced at once, by the value at that moment.

U-Boot's shell is a small version of `sh`: `;` separates commands, `&&` and `||` work, and so do `if test ...; then ...; fi` and `for`.

**Save a variable.** `bootdelay` is the countdown in seconds:

```console
=> setenv bootdelay 5
=> saveenv
Saving Environment to Flash... ... OK
```

Quit QEMU (Ctrl-A, then X) and start it again. This time:

```
Loading Environment from Flash... OK
...
Hit any key to stop autoboot:  5
```

On QEMU the variables are stored in `envstore.img`. On a board they're usually a file on the SD card's FAT partition (section 9) or an area of the eMMC or flash. To start over, delete `envstore.img`.

---

## 7. Boot Linux from U-Boot

Planned: load Debian's installer kernel and initramfs over TFTP and start it with bootz or booti.

---

## 8. Make it boot by itself

Planned: bootcmd, a boot script made with mkimage, and standard boot with extlinux.conf.

---

## 9. Run it on a board

Planned: the serial console, then U-Boot on a BeagleBone Black SD card and on a Raspberry Pi 4, booting Linux by hand and automatically.

---

## 10. Troubleshooting

Planned: symptoms, causes and fixes, from a silent serial port to a kernel panic.

---

## 11. Memory and load addresses

Planned: RAM start and load address variables on each board, and how to avoid overlapping loads.

---

## 12. Kernel image formats and FIT

Planned: zImage, Image, uImage and FIT; recognising them; a FIT example.

---

## 13. The device tree in the bootloader

Planned: what U-Boot changes in the device tree, the fdt command, and overlays.

---

## 14. Booting over the network

Planned: TFTP and an NFS root from the PC, with the BeagleBone's netboot variable.

---

## 15. The environment from Linux

Planned: fw_printenv and fw_setenv, fw_env.config, and a redundant environment.

---

## 16. Safe updates and verified boot

Planned: bootcount and A/B updates, signed FIT images, and locking down the command line.

---

## 17. Debugging U-Boot

Planned: debug messages, and GDB with QEMU across U-Boot's relocation.

---

## 18. Where the rest is covered

Planned: links to the kernel, root filesystem and build system notes, and what isn't covered.

---

## References

Planned: U-Boot, Linux, QEMU and board documentation.
