# Bootloaders for Embedded Linux: U-Boot

A beginner's path from power-on to a running Linux kernel: what a bootloader does, how to build U-Boot, how to use its command line in QEMU, and how to put it on a BeagleBone Black or a Raspberry Pi 4.

**You need:** the PC, toolchains and settings files from the [cross-compilation note](../cross-compilation-toolchain/cross-compilation-toolchain_v2_2026-11-08.md) (sections 3 and 4). No board is needed until section 9. For a board, you also need a 3.3 V USB-to-serial cable.

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

The [kernel note](../kernel/) builds a kernel. Until then, we use Debian's installer kernel and initramfs. They're built for many Arm machines, including QEMU's `virt` and the BeagleBone Black.

```sh
mkdir -p $BOOTDIR/tftp
wget -O $BOOTDIR/tftp/kernel $NETBOOT/$NETBOOT_KERNEL
wget -O $BOOTDIR/tftp/initrd.gz $NETBOOT/initrd.gz
```

Start QEMU, stop the countdown, and load the two files over the network with TFTP.

**Expected (QEMU)**, BeagleBone Black settings, shortened. Sizes change with Debian updates.

```console
=> setenv autoload no
=> dhcp
BOOTP broadcast 1
DHCP client bound to address 10.0.2.15 (2 ms)
=> setenv serverip 10.0.2.2
=> tftp ${kernel_addr_r} kernel
Using e1000#0 device
TFTP from server 10.0.2.2; our IP address is 10.0.2.15
Filename 'kernel'.
Load address: 0x40400000
Loading: #################################################################
...
done
Bytes transferred = 6046208 (5c4200 hex)
=> tftp ${ramdisk_addr_r} initrd.gz
...
Bytes transferred = 25985449 (18c81a9 hex)
=> setenv bootargs console=ttyAMA0
=> bootz ${kernel_addr_r} ${ramdisk_addr_r}:${filesize} ${fdt_addr}
Kernel image @ 0x40400000 [ 0x000000 - 0x5c4200 ]
...
Starting kernel ...

[    0.000000] Booting Linux on physical CPU 0x0
[    0.000000] Linux version 6.12...
...
```

After the kernel messages, the Debian installer asks for a language. You have started Linux from U-Boot. Press Ctrl-A, then X, to quit.

**With the Raspberry Pi settings, use `booti` instead of `bootz`.** Everything else is the same.

What each step did:

| Step | Why |
|---|---|
| `setenv autoload no` | Otherwise `dhcp` also tries to download a file named by the DHCP server |
| `dhcp` | Gets an address from QEMU's DHCP server. Inside QEMU, the guest is `10.0.2.15` and the PC is `10.0.2.2` |
| `setenv serverip 10.0.2.2` | The TFTP server: QEMU's built-in one, serving `$BOOTDIR/tftp` |
| `tftp ADDR FILE` | Copies the file into RAM at that address. Afterwards `filesize` holds its size |
| `setenv bootargs console=ttyAMA0` | The kernel command line. `ttyAMA0` is the name Linux gives to `virt`'s serial port |
| `bootz KERNEL INITRD:SIZE DTB` | Starts a zImage. The initramfs needs its size, which is why it is loaded last and `${filesize}` is used. Use `-` for "no initramfs" |

**Which command starts the kernel:**

| Command | For |
|---|---|
| `bootz` | A 32-bit Arm zImage |
| `booti` | A 64-bit Arm Image |
| `bootm` | A legacy uImage, or a FIT image (section 12) |
| `bootefi` | An EFI program, such as GRUB, or a kernel started through its EFI stub |

Before jumping, `bootz` and `booti` check the kernel's header, copy the device tree and the initramfs to a safe place, write `bootargs` and the initramfs location into the device tree, and turn off U-Boot's devices. On QEMU, the device tree is the one QEMU generated, at `fdt_addr`. On a real board you load a `.dtb` file, as in section 9.

---

## 8. Make it boot by itself

Typing those commands at every boot doesn't scale. There are three ways to make it automatic, from simplest to most flexible.

### Put the commands in `bootcmd`

**Expected (QEMU)**, BeagleBone Black settings:

```console
=> setenv netboot 'setenv autoload no; dhcp; setenv serverip 10.0.2.2; tftp ${kernel_addr_r} kernel; tftp ${ramdisk_addr_r} initrd.gz; setenv bootargs console=ttyAMA0; bootz ${kernel_addr_r} ${ramdisk_addr_r}:${filesize} ${fdt_addr}'
=> setenv bootcmd 'run netboot'
=> saveenv
=> reset
```

After the countdown, Linux starts with no typing. To get back to the prompt, press a key during the countdown. To undo, run `env default -a` and `saveenv`.

### Use a boot script

The commands can live in a file on the PC instead, so you change them with an editor and no `saveenv`. Create `$BOOTDIR/tftp/boot.cmd`:

```sh
echo "== boot.cmd: loading Linux over TFTP"
setenv bootargs console=ttyAMA0
tftp ${kernel_addr_r} kernel
tftp ${ramdisk_addr_r} initrd.gz
bootz ${kernel_addr_r} ${ramdisk_addr_r}:${filesize} ${fdt_addr}
```

U-Boot runs scripts in a small wrapper made by `mkimage`, which adds a header with a checksum. **Expected (QEMU):**

```console
$ mkimage -A arm -O linux -T script -C none -n "QEMU netboot" \
      -d $BOOTDIR/tftp/boot.cmd $BOOTDIR/tftp/boot.scr
Image Name:   QEMU netboot
...
Image Type:   ARM Linux Script (uncompressed)
...
```

Then make `bootcmd` load and run it:

```console
=> setenv bootcmd 'setenv autoload no; dhcp; setenv serverip 10.0.2.2; tftp ${scriptaddr} boot.scr; source ${scriptaddr}'
=> saveenv
=> reset
...
## Executing script at 40200000
== boot.cmd: loading Linux over TFTP
...
Starting kernel ...
```

From now on, edit `boot.cmd`, run `mkimage` again, and restart QEMU. With the Pi settings, write `booti` in `boot.cmd`, and use `-A arm64`.

### Let standard boot find it

The default `bootcmd` is `bootflow scan -lb`. This is **standard boot**: U-Boot searches devices in the order of `boot_targets`, and on each one looks for something it knows how to boot:

| It looks for | Written by |
|---|---|
| `extlinux/extlinux.conf`, in `/` or `/boot/` | Distributions, Buildroot, Yocto. The format comes from the Syslinux bootloader |
| `boot.scr` | You, as above |
| An EFI program, such as `EFI/BOOT/BOOTAA64.EFI` | Distributions that boot with UEFI |

The first one it finds, it boots. **Expected (QEMU):**

```console
=> printenv boot_targets
boot_targets=qfw usb scsi virtio nvme dhcp
```

An `extlinux.conf` describes the boot in a few lines (**Illustrative**):

```
label linux
    kernel /zImage
    fdt /am335x-boneblack.dtb
    initrd /initrd.gz
    append console=ttyS0,115200n8 root=/dev/mmcblk0p2 rw rootwait
```

Section 9 puts an `extlinux.conf` on the BeagleBone's SD card. Some boards, the BeagleBone among them, still start the older `run distro_bootcmd` instead of `bootflow`. It searches for the same files.

---

## 9. Run it on a board

Everything here is **Expected (board)**: not run on hardware for this note.

### The serial console

You need a USB-to-serial cable with **3.3 V** signals, such as an FTDI TTL-232R-3V3. **A 5 V or RS-232 cable can damage the board.** Connect GND to GND, the cable's TX to the board's RX, and the cable's RX to the board's TX. Don't connect the cable's power wire.

On the PC:

```sh
sudo apt install picocom
sudo usermod -aG dialout $USER     # then log out and in again
picocom -b 115200 /dev/ttyUSB0     # Ctrl-A, then Ctrl-X, quits
```

| | BeagleBone Black | Raspberry Pi 4 |
|---|---|---|
| Where | J1, the 6-pin header next to the P9 header | The 40-pin header |
| GND | Pin 1, marked with a dot | Pin 6 |
| Board's RX (cable TX goes here) | Pin 4 | Pin 10, GPIO 15 |
| Board's TX (cable RX goes here) | Pin 5 | Pin 8, GPIO 14 |

### BeagleBone Black

Build U-Boot for the board:

```console
$ . ~/cross/target-bbb.sh
$ cd ~/boot/u-boot
$ make O=$BOOTDIR/build-board $UBOOT_BOARD
$ make O=$BOOTDIR/build-board -j$(nproc)
$ ls $BOOTDIR/build-board/MLO $BOOTDIR/build-board/u-boot.img
```

`MLO` is the SPL with the header the boot ROM expects. `u-boot.img` is U-Boot proper, packed with the device trees of the supported BeagleBones so one build runs on all of them.

**Prepare a spare microSD card. Everything on it will be erased.** Find its name with `lsblk`, here `/dev/sdX`, and make one FAT partition marked bootable. The AM335x boot ROM looks for `MLO` on a FAT partition with the bootable flag set.

```sh
sudo sfdisk /dev/sdX <<'EOF'
label: dos
start=2048, size=256MiB, type=c, bootable
EOF
sudo mkfs.vfat -F 32 -n BOOT /dev/sdX1
sudo mount /dev/sdX1 /mnt
sudo cp $BOOTDIR/build-board/MLO /mnt/
sudo cp $BOOTDIR/build-board/u-boot.img /mnt/
sudo cp $BOOTDIR/tftp/kernel $BOOTDIR/tftp/initrd.gz /mnt/
sudo wget -P /mnt ${NETBOOT%/netboot}/device-tree/am335x-boneblack.dtb
sudo umount /mnt
```

The kernel and initramfs are Debian's from section 7, and Debian also publishes the BeagleBone's device tree. Copying `MLO` first is an old habit from earlier TI chips. It costs nothing.

**Boot from the card.** The BeagleBone boots from its eMMC first. To boot from the SD card, hold the S2 button, near the card slot, while you apply power, then release it. In picocom:

```
U-Boot SPL 2026.10 (Nov 09 2026 - 21:30:12 +0100)
...
U-Boot 2026.10 (Nov 09 2026 - 21:30:12 +0100)

...
Model: TI AM335x BeagleBone Black
DRAM:  512 MiB
...
Hit any key to stop autoboot:  2
=>
```

The SPL line proves the boot ROM found `MLO`. If the version or date isn't yours, the board booted from eMMC: hold S2 again.

Stop the countdown, find the device tree's name, and boot. The commands are the ones from QEMU, with `load` from the SD card instead of `tftp`:

```console
=> run findfdt
=> printenv board_name fdtfile
board_name=A335BNLT
fdtfile=am335x-boneblack.dtb
=> ls mmc 0:1
=> load mmc 0:1 ${kernel_addr_r} kernel
=> load mmc 0:1 ${fdt_addr_r} ${fdtfile}
=> load mmc 0:1 ${ramdisk_addr_r} initrd.gz
=> setenv bootargs console=ttyS0,115200n8
=> bootz ${kernel_addr_r} ${ramdisk_addr_r}:${filesize} ${fdt_addr_r}
...
Starting kernel ...
```

| Detail | Why |
|---|---|
| `run findfdt` | Sets `fdtfile` from the board name stored in the board's EEPROM. The default `bootcmd` runs it, so it's missing when you stop the countdown |
| `mmc 0:1` | Device 0, partition 1. On the BeagleBone, MMC 0 is the SD card and MMC 1 is the eMMC |
| `${fdt_addr_r}` | Unlike on QEMU, the device tree comes from a file |
| `ttyS0` | Debian's kernel names the serial port `ttyS0`. U-Boot's `console` variable says `ttyO0`, the name used by an older driver |

**Make it boot by itself** with an `extlinux.conf` on the card, in `/mnt/extlinux/extlinux.conf`:

```
label debian-installer
    kernel /kernel
    fdt /am335x-boneblack.dtb
    initrd /initrd.gz
    append console=ttyS0,115200n8
```

At the next boot, the default `bootcmd` finds it:

```
Found /extlinux/extlinux.conf
Retrieving file: /extlinux/extlinux.conf
1:	debian-installer
Retrieving file: /kernel
...
Starting kernel ...
```

`saveenv` works too. On this board it writes the variables to a file named `uboot.env` on the card's FAT partition.

**About `uEnv.txt`.** Many BeagleBone guides, and BeagleBoard.org's own images, use a `uEnv.txt` file with a `uenvcmd` variable. Mainline U-Boot's `am335x_evm` configuration doesn't read it in v2026.10. Use `extlinux.conf` or `boot.scr`, or import it yourself: `load mmc 0:1 ${loadaddr} uEnv.txt; env import -t ${loadaddr} ${filesize}; run uenvcmd`.

### Raspberry Pi 4

Build U-Boot for the board:

```console
$ . ~/cross/target-pi4.sh
$ cd ~/boot/u-boot
$ make O=$BOOTDIR/build-board $UBOOT_BOARD
$ make O=$BOOTDIR/build-board -j$(nproc)
$ scp $BOOTDIR/build-board/u-boot.bin $BOARD:
```

This uses the Raspberry Pi OS card from the cross-compilation note. On the Pi, the FAT boot partition is mounted at `/boot/firmware`. Install U-Boot and keep a copy of `config.txt`:

```sh
ssh $BOARD
sudo cp u-boot.bin /boot/firmware/
sudo cp /boot/firmware/config.txt /boot/firmware/config.txt.orig
```

Add these lines at the end of `/boot/firmware/config.txt`:

```ini
[all]
enable_uart=1
arm_64bit=1
kernel=u-boot.bin
```

`kernel=u-boot.bin` makes the firmware start U-Boot instead of Linux. `enable_uart=1` turns on the serial port on pins 8 and 10. `arm_64bit=1` starts the CPU in 64-bit mode, which `rpi_4_defconfig` needs. Run `sudo reboot`, and in picocom stop the countdown:

```
U-Boot 2026.10 (Nov 09 2026 - 21:40:51 +0100)

DRAM:  ...
RPI 4 Model B (0x...)
...
Hit any key to stop autoboot:  2
U-Boot>
```

This build's prompt is `U-Boot>` instead of `=>`. Start Raspberry Pi OS:

```console
U-Boot> fatls mmc 0:1
U-Boot> load mmc 0:1 ${kernel_addr_r} kernel8.img
U-Boot> booti ${kernel_addr_r} - ${fdt_addr}
...
Starting kernel ...
```

Raspberry Pi OS boots to its login prompt.

| Detail | Why |
|---|---|
| `kernel8.img` | Raspberry Pi OS's 64-bit kernel. If it's compressed, `booti` decompresses it using `kernel_comp_addr_r` and `kernel_comp_size`, which the Pi's default variables set |
| `-` | No initramfs: the kernel can mount the SD card by itself |
| `${fdt_addr}` | The device tree the **firmware** prepared, with its overlays and the contents of `cmdline.txt`. Because `bootargs` is empty, U-Boot keeps that command line |

**Make it boot by itself** with a boot script on the boot partition. Standard boot finds `boot.scr` there. On the PC, create `boot.cmd`:

```sh
load mmc 0:1 ${kernel_addr_r} kernel8.img
booti ${kernel_addr_r} - ${fdt_addr}
```

```sh
mkimage -A arm64 -O linux -T script -C none -d boot.cmd boot.scr
scp boot.scr $BOARD:
ssh $BOARD sudo cp boot.scr /boot/firmware/
```

When Raspberry Pi OS updates its kernel, it replaces `kernel8.img`, and the script loads the new one.

**To undo**, put the original `config.txt` back: `sudo cp /boot/firmware/config.txt.orig /boot/firmware/config.txt`. If the Pi doesn't boot, do it from the PC by mounting the card's first partition.

**This is the end of the essential path.** You can now build U-Boot, use its command line, start Linux by hand and automatically, in QEMU and on a board.

---

## 10. Troubleshooting

**Read this when something fails.** First find the last line that worked: nothing at all, the SPL line, U-Boot's prompt, or `Starting kernel ...`.

Nothing in this table was reproduced for this note.

| Symptom | Likely cause | What to do |
|---|---|---|
| Nothing on the serial console | TX and RX not crossed, GND missing, wrong speed, or the wrong `/dev/ttyUSB*` | Swap TX and RX. Check `-b 115200`. Look at `dmesg` after plugging the cable |
| Pi: nothing on the serial console, but HDMI shows the rainbow screen | `enable_uart=1` missing, or `kernel=` names a file that isn't there | Check `config.txt` from the PC |
| BBB: the U-Boot banner shows an old version or date | The board booted from its eMMC | Hold S2 while applying power |
| BBB: nothing at all, even with S2 held | `MLO` not found: the partition isn't FAT or isn't marked bootable | `sudo fdisk -l /dev/sdX` should show `*` under Boot |
| BBB: `U-Boot SPL` line, then an error | The SPL can't find `u-boot.img` on the FAT partition | Check the file name, and that both files come from the same build |
| `Unknown command 'bootz' - try 'help'` | That command isn't enabled in this configuration | `make O=... menuconfig`, or use the command `help` lists |
| `zimage: Bad magic!` | The file at that address isn't a zImage: the load failed, it's the wrong file, or it's a 64-bit Image | Check `filesize` after loading. Use `booti` for 64-bit |
| `Bad Linux ARM64 Image magic!` | Not a 64-bit Image: a zImage, a compressed Image without `kernel_comp_addr_r`, or the wrong file | Use the matching command and file (section 12) |
| `Wrong Ramdisk Image Format` | The initramfs was given without its size | `${ramdisk_addr_r}:${filesize}`, with `filesize` from the initramfs load |
| `Starting kernel ...` and then nothing | Wrong `console=` for this kernel, or the wrong device tree | Use `ttyAMA0` on QEMU, `ttyS0` on the BBB. Add `earlycon` to `bootargs` to see early messages |
| `Kernel panic - not syncing: VFS: Unable to mount root fs` | No initramfs and a wrong or missing `root=`, or the kernel lacks the driver for that disk | Check `bootargs`. The [root filesystem](../rootfs/) note covers this |
| `*** Warning - bad CRC, using default environment` | Nothing saved yet, or the saved copy was erased | Normal on a first boot. `saveenv` |
| `saveenv` fails in QEMU | QEMU started without the `-drive if=pflash,index=1` option | Use `run-qemu.sh` |
| `Retry count exceeded; starting again` during `tftp` or `dhcp` | No answer from the server | `printenv serverip ipaddr`. On a board, check the cable and the PC's firewall |
| `TFTP error: 'File not found' (1)` | The file isn't in the TFTP folder, or the name differs | File names are case-sensitive |
| `** Reading file would overwrite reserved memory **` | The load address is where U-Boot itself lives | Use the `*_addr_r` variables (section 11) |
| `** Bad device specification mmc 2 **` | That device or partition doesn't exist | `mmc list`, then `part list mmc 0` |
| BBB: `WARNING: Could not determine device tree to use` | `fdtfile` is `undefined` | `run findfdt`, or `setenv fdtfile am335x-boneblack.dtb` |
| BBB: U-Boot ignores `uEnv.txt` | Mainline U-Boot doesn't read it | Section 9, "About `uEnv.txt`" |
| The board reboots in a loop | `bootcmd` fails or starts a kernel that crashes | Press a key during the countdown, then fix or reset the variables |

---

## 11. Memory and load addresses

**Read this when** you choose where to load files, or a kernel crashes right after a load that looked fine.

Each board's RAM starts at a different address, and each default environment has variables with safe places to load things. The `_r` means "in RAM".

| Variable | Used for | QEMU `virt` | BeagleBone Black | Raspberry Pi 4 |
|---|---|---|---|---|
| RAM start | | `0x40000000` | `0x80000000` | `0x00000000` |
| `kernel_addr_r` | The kernel | `0x40400000` | `0x82000000` | `0x00080000` |
| `fdt_addr_r` | A device tree loaded from a file | Not set | `0x88000000` | `0x05600000` |
| `fdt_addr` | The device tree a previous stage provided | `0x40000000`, from QEMU | Not set | Set at boot, from the firmware |
| `ramdisk_addr_r` | The initramfs | `0x44000000` | `0x88080000` | `0x05700000` |
| `scriptaddr` | A boot script | `0x40200000` | `0x80000000` | `0x05400000` |

Values from U-Boot v2026.10's default environments.

**Nothing stops a load from overwriting another one.** On the BeagleBone, the device tree at `0x88000000` has `0x80000` bytes (512 KiB) before the initramfs starts, which is plenty for a `.dtb`. A kernel bigger than the gap to the next address would be silently cut by the next load. After each load, compare `filesize` with the gap.

**Why the kernel isn't at the start of RAM.** A zImage decompresses itself near the start of RAM. The BeagleBone's `0x82000000` leaves 32 MiB below it for that.

**What U-Boot protects.** U-Boot moves itself to the top of RAM, and refuses loads over that area. Before starting the kernel, it copies the device tree and initramfs to a place the kernel can reach early. `bootm_size` limits that place: the BeagleBone's `0x10000000` keeps them in the first 256 MiB of RAM.

---

## 12. Kernel image formats and FIT

**Read this when** a kernel won't start with "bad magic", or you need a single signed boot file.

| Format | How to recognise it | Command | Made by |
|---|---|---|---|
| **zImage** | 32-bit Arm. Bytes `18 28 6f 01` at offset `0x24` | `bootz` | The kernel build: `arch/arm/boot/zImage` |
| **Image** | 64-bit Arm. Text `ARMd` at offset `0x38` | `booti` | The kernel build: `arch/arm64/boot/Image` |
| **Image.gz** | A compressed Image | `booti`, if `kernel_comp_addr_r` and `kernel_comp_size` are set | The kernel build |
| **uImage** | A 64-byte U-Boot header in front of a kernel. Starts with `27 05 19 56` | `bootm` | `mkimage -T kernel`. Older guides use it |
| **FIT** (`.itb`) | A device tree that contains files. Starts with `d0 0d fe ed` | `bootm` | `mkimage -f` |

Check the first bytes of the kernels from section 7. **Expected (QEMU):**

```console
$ od -A x -t x1z -j 0x24 -N 4 ~/boot/bbb/tftp/kernel
000024 18 28 6f 01                                      >.(o.<
000028
$ od -A x -t x1z -j 0x38 -N 4 ~/boot/pi4/tftp/kernel
000038 41 52 4d 64                                      >ARMd<
00003c
```

### FIT images

A **FIT** (flattened image tree) puts the kernel, one or more device trees and the initramfs in one file, with a checksum for each part. It is the format to use for products, because the parts can't get mixed up, and the file can be signed (section 16).

`bbb.its` describes the file (**Illustrative**):

```dts
/dts-v1/;

/ {
    description = "BeagleBone Black: kernel, device tree, initramfs";
    #address-cells = <1>;

    images {
        kernel {
            data = /incbin/("kernel");
            type = "kernel_noload";
            arch = "arm";
            os = "linux";
            compression = "none";
            load = <0>;
            entry = <0>;
            hash { algo = "sha256"; };
        };
        fdt {
            data = /incbin/("am335x-boneblack.dtb");
            type = "flat_dt";
            arch = "arm";
            compression = "none";
            hash { algo = "sha256"; };
        };
        initrd {
            data = /incbin/("initrd.gz");
            type = "ramdisk";
            arch = "arm";
            os = "linux";
            compression = "none";
            hash { algo = "sha256"; };
        };
    };

    configurations {
        default = "bbb";
        bbb {
            kernel = "kernel";
            fdt = "fdt";
            ramdisk = "initrd";
        };
    };
};
```

`kernel_noload` runs the zImage where it is in the file, since a zImage can run from any address. Build it, list it, and boot it (**Illustrative**):

```sh
mkimage -f bbb.its bbb.itb
mkimage -l bbb.itb

# in U-Boot
load mmc 0:1 ${loadaddr} bbb.itb
setenv bootargs console=ttyS0,115200n8
bootm ${loadaddr}
```

---

## 13. The device tree in the bootloader

**Read this when** you need to change the hardware description at boot, or add an overlay.

U-Boot edits the device tree before starting the kernel. It writes `bootargs` and the initramfs location into the `/chosen` node, fills in the real RAM size, and adds details such as MAC addresses. You can look at it and change it with the `fdt` command.

**Expected (QEMU):**

```console
=> fdt addr ${fdt_addr}
=> fdt print /chosen
chosen {
	...
	stdout-path = "/pl011@9000000";
};
```

`stdout-path` is how the device tree says which serial port is the console. That's also what `earlycon`, with no options, uses.

**Overlays** are small device tree fragments applied on top of the board's device tree, for example for a cape or a sensor. On the BeagleBone (**Illustrative**):

```
load mmc 0:1 ${fdt_addr_r} am335x-boneblack.dtb
fdt addr ${fdt_addr_r}
fdt resize 8192
load mmc 0:1 ${fdtoverlay_addr_r} my-sensor.dtbo
fdt apply ${fdtoverlay_addr_r}
```

`fdt resize` makes room for the new nodes. Both the base `.dtb` and the overlay must be compiled with symbols (`dtc -@`), or `fdt apply` fails. In `extlinux.conf`, an `fdtoverlays /my-sensor.dtbo` line does the same.

On the Raspberry Pi, keep using `dtoverlay=` lines in `config.txt`. The firmware applies them before U-Boot starts, and U-Boot passes the result on.

---

## 14. Booting over the network

**Read this when** you rebuild the kernel or root filesystem often. Booting from the PC avoids rewriting the SD card each time.

The board loads the kernel and device tree with TFTP, and the kernel mounts its root filesystem from the PC with **NFS**. On the PC (**Illustrative**):

```sh
sudo apt install tftpd-hpa nfs-kernel-server
sudo cp zImage am335x-boneblack.dtb /srv/tftp/
echo '/srv/nfs/bbb *(rw,sync,no_subtree_check,no_root_squash)' | sudo tee -a /etc/exports
sudo exportfs -ra
```

`/srv/tftp` is the folder the `tftpd-hpa` package serves. `/srv/nfs/bbb` holds a root filesystem, which the [root filesystem](../rootfs/) note builds.

The BeagleBone's default environment already has a `netboot` variable that does all of it: `dhcp`, `tftp` of `${bootfile}` and `${fdtfile}`, then `bootz` with an NFS root. Connect the board's Ethernet port to your network. **Expected (board)**, with the PC at `192.168.1.10`:

```console
=> run findfdt
=> setenv console ttyS0,115200n8
=> setenv serverip 192.168.1.10
=> setenv bootfile zImage
=> setenv rootpath /srv/nfs/bbb
=> run netboot
Booting from network ...
...
```

The kernel needs the Ethernet driver and NFS root support built in, not as modules, because it needs them before any root filesystem exists. If your network's DHCP server names another TFTP server, it replaces `serverip`; check with `printenv serverip`.

---

## 15. The environment from Linux

**Read this when** a program on the board must change U-Boot's variables, for example to switch to a new system after an update.

`fw_printenv` and `fw_setenv` read and write U-Boot's saved variables from Linux. They come in the `libubootenv-tool` package, or `u-boot-tools` on older releases. They need `/etc/fw_env.config` to say where the variables are and how big the area is. On the Raspberry Pi 4, where they're a file on the boot partition (**Illustrative**):

```
# File or device          Offset    Size
/boot/firmware/uboot.env  0x0000    0x4000
```

```sh
fw_printenv bootdelay
sudo fw_setenv bootdelay 5
```

**The size must equal U-Boot's `CONFIG_ENV_SIZE`**: `0x4000` in `rpi_4_defconfig`, `0x20000` for the BeagleBone. Otherwise the checksum doesn't match and the tools report a bad CRC. Find it with `grep CONFIG_ENV_SIZE $BOOTDIR/build-board/.config`.

**Redundant environment.** If power is lost during `saveenv`, the saved variables are corrupted, and U-Boot falls back to its defaults. With `CONFIG_ENV_REDUNDANT`, U-Boot keeps two copies and writes them in turn, so one is always valid. Products should enable it.

---

## 16. Safe updates and verified boot

**Read this when** you design how a product updates itself, or must ensure it only runs your software.

### Falling back after a bad update

U-Boot can count boots. The **`bootcount`** value goes up at each boot, and Linux resets it once the system is known to work. If it goes above **`bootlimit`**, U-Boot runs **`altbootcmd`** instead of `bootcmd`.

With two copies of the system, A and B, an update writes the inactive copy and switches `bootcmd` to it. If the new system fails to reset `bootcount` a few times in a row, `altbootcmd` boots the old one. Update tools such as RAUC, SWUpdate and Mender work this way with U-Boot. The BeagleBone's configuration stores `bootcount` in a register of the chip's real-time clock, which survives a reset.

### Verified boot

U-Boot can refuse a kernel that isn't signed with your key. You sign a FIT image (section 12) with a private key, and U-Boot checks it with the public key stored in its own device tree. Both defconfigs here enable it (`CONFIG_FIT_SIGNATURE`). U-Boot's documentation has a [complete example for the BeagleBone Black](https://docs.u-boot.org/en/latest/usage/fit/beaglebone_vboot.html).

This only helps if U-Boot itself can't be replaced, and that needs the chip's boot ROM to verify the first stage. The BeagleBone's AM3358 is a general-purpose part that doesn't do this. TI's high-security parts do, and Raspberry Pi has its own signed-boot scheme for the Pi 4. In a product, also stop people from stopping the countdown: `CONFIG_AUTOBOOT_KEYED` requires a secret string, and disabling `CONFIG_CMDLINE` removes the command line entirely.

---

## 17. Debugging U-Boot

**Read this when** U-Boot itself crashes or misbehaves, or you change its code.

**Print more.** Add `#define DEBUG` at the top of a source file, before its `#include` lines, and rebuild: that file's `debug()` messages are printed.

**Use GDB with QEMU.** `-s` starts QEMU's GDB server on port 1234, and `-S` waits for the debugger before running. Install `gdb-multiarch` on the PC. **Expected (QEMU):**

```sh
# Terminal 1
~/boot/run-qemu.sh -s -S

# Terminal 2
gdb-multiarch $BOOTDIR/build-qemu/u-boot -ex 'target remote :1234'
```

**U-Boot moves itself during boot**, so after the move the symbols in the ELF file no longer match the addresses. Find where it went: `relocaddr` in `bdinfo`, or from GDB, where the `r9` register (`x18` on 64-bit) points to U-Boot's global data. Then reload the symbols at that address (**Illustrative**):

```
(gdb) p/x (*(struct global_data *)$r9)->relocaddr
$1 = 0x7f...
(gdb) symbol-file
(gdb) add-symbol-file build-qemu/u-boot 0x7f...
```

Breakpoints and backtraces then work as usual. U-Boot's [GDB guide](https://docs.u-boot.org/en/latest/develop/gdb.html) has the details.

---

## 18. Where the rest is covered

- **Building the kernel and device trees** (`zImage`, `Image`, `.dtb`): the [kernel](../kernel/) note.
- **Root filesystems, initramfs, and an NFS root:** the [root filesystem](../rootfs/) note.
- **U-Boot built for you as part of a whole system:** the [Buildroot](../../linux-build-systems/buildroot/) and [Yocto](../../linux-build-systems/yocto/) notes.
- **Not covered here:** Trusted Firmware-A and OP-TEE, which many 64-bit chips run before U-Boot; UEFI boot with GRUB; other bootloaders such as barebox; porting U-Boot to a new board.

---

## References

U-Boot:

- [U-Boot documentation](https://docs.u-boot.org/en/latest/), and the [source](https://git.u-boot-project.org/u-boot/u-boot)
- [Building with GCC](https://docs.u-boot.org/en/latest/build/gcc.html)
- Boards: [QEMU Arm](https://docs.u-boot.org/en/latest/board/emulation/qemu-arm.html), [AM335x](https://docs.u-boot.org/en/latest/board/ti/am335x_evm.html), [Raspberry Pi](https://docs.u-boot.org/en/latest/board/broadcom/raspberrypi.html)
- [Environment variables](https://docs.u-boot.org/en/latest/usage/environment.html), [standard boot](https://docs.u-boot.org/en/latest/develop/bootstd/index.html), [extlinux.conf format](https://docs.u-boot.org/en/latest/usage/pxe.html)
- Commands: [bootz](https://docs.u-boot.org/en/latest/usage/cmd/bootz.html), [booti](https://docs.u-boot.org/en/latest/usage/cmd/booti.html), [source](https://docs.u-boot.org/en/latest/usage/cmd/source.html), [fdt](https://docs.u-boot.org/en/latest/usage/cmd/fdt.html)
- [FIT images](https://docs.u-boot.org/en/latest/usage/fit/index.html), [device tree overlays](https://docs.u-boot.org/en/latest/usage/fdt_overlays.html), [boot count](https://docs.u-boot.org/en/latest/api/bootcount.html), [debugging with GDB](https://docs.u-boot.org/en/latest/develop/gdb.html)

Linux:

- Booting the kernel: [32-bit Arm](https://docs.kernel.org/arch/arm/booting.html), [64-bit Arm](https://docs.kernel.org/arch/arm64/booting.html)
- [Mounting the root filesystem over NFS](https://docs.kernel.org/admin-guide/nfs/nfsroot.html)
- [Debian installer images for armhf](https://deb.debian.org/debian/dists/trixie/main/installer-armhf/current/images/) and [arm64](https://deb.debian.org/debian/dists/trixie/main/installer-arm64/current/images/)

QEMU:

- [The `virt` machine](https://www.qemu.org/docs/master/system/arm/virt.html), [network emulation](https://www.qemu.org/docs/master/system/devices/net.html)

Boards:

- BeagleBone Black: [documentation](https://docs.beagleboard.org/boards/beaglebone/black/index.html), [AM335x technical reference manual](https://www.ti.com/lit/pdf/spruh73), chapter "Initialization"
- Raspberry Pi 4: [config.txt](https://www.raspberrypi.com/documentation/computers/config_txt.html), [boot sequence and EEPROM](https://www.raspberrypi.com/documentation/computers/raspberry-pi.html), [UARTs](https://www.raspberrypi.com/documentation/computers/configuration.html)

---

Version 2 · 2026-11-09
