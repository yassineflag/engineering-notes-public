# Linux Kernel Device Drivers for Embedded Linux

How to write, build, load, debug and ship a Linux device driver for an embedded board, from a first module to drivers for real hardware on the device tree: a character device, a platform driver with a GPIO and an interrupt, and an I2C sensor driver that plugs into a kernel subsystem. The board is the BeagleBone Black of the fundamentals notes, so the kernel, the device tree and the module directory are the ones the [kernel note](../fundamentals/03-kernel/kernel_v1_2026-11-10.md) built and installed.

This note starts where the kernel note's section 15 stops: an out-of-tree `hello.ko` that loads. Configuring and building the kernel itself, the boot and the device tree's role at boot are in the kernel note; the cross-compiler is in [Cross-Compilation and Toolchains](../fundamentals/01-cross-compilation-toolchain/cross-compilation-toolchain_v2_2026-11-08.md); packaging a module in an image is in the [Buildroot](../linux-build-systems/buildroot/buildroot_v1_2026-11-15.md) and [Yocto](../linux-build-systems/yocto/yocto_v1_2026-11-14.md) notes. The userspace side of device files (`open`, `read`, `ioctl`, `poll`) belongs to [Linux systems programming](../../linux-systems-programming/).

**Assumptions:**
- The board is a BeagleBone Black (TI AM3358, one Cortex-A8, 512 MiB DDR3L), running a kernel built as in the kernel note, with its build directory at `~/bbb/build` and the serial console on J1 at 115200 8N1.
- The driver APIs are those of mainline Linux 7.3. The kernel note's board kernel is a 7.1-rc1 snapshot; every API used here exists in both. Section 3 lists the API changes you'll meet in older drivers and tutorials.
- The build machine is x86_64 Debian or Ubuntu with the `arm-none-linux-gnueabihf-` cross-compiler. `ARCH=arm` and `CROSS_COMPILE=arm-none-linux-gnueabihf-` are exported for every command, as in the kernel note.
- The external hardware is an LED, a push button and a TI TMP102 temperature sensor on a breadboard, wired to the P9 header (section 2). Any LED, button and I2C device work the same way.
- Network addresses (board `192.168.0.50`) and device names are placeholders.

**Conventions:**
- `$` is a shell on the build machine, `#` a root shell on the board.
- **Real output** marks text pasted from a command run for this note.
- **From source** marks behavior read in the Linux 7.3 source, with the file cited next to it.
- **Expected** marks output reconstructed from the format strings in the source, not captured from a board. Addresses, numbers and timestamps in it are placeholders.
- **Illustrative** marks code that shows the shape of a solution and was not compiled.
- Every other command is generic: it follows the kernel's documentation and was not run for this note.

**Verification status:** written in November 2026 against a shallow clone of mainline Linux 7.3. The details, and what was and wasn't run, are in the verification paragraph at the end of section 2.

**How to read it:**
- **To learn:** Parts 1 to 3 in order. Each part reuses the code of the one before.
- **To write a driver for a device on the device tree:** sections 10 to 13, then 16 for I2C and SPI.
- **To pick the interface userspace sees:** section 17 before writing a character device.
- **When it doesn't work:** section 22 lists symptoms and points back to the explanation.

## Table of contents

**Part 1: Foundations**

- [1. The mental model: what a driver is](#1-the-mental-model-what-a-driver-is)
- [2. The example board, the hardware and the development loop](#2-the-example-board-the-hardware-and-the-development-loop)
- [3. Kernel code is not userspace code](#3-kernel-code-is-not-userspace-code)
- [4. A module in detail](#4-a-module-in-detail)

**Part 2: Talking to userspace**

- [5. Character devices: numbers, cdev and the misc shortcut](#5-character-devices-numbers-cdev-and-the-misc-shortcut)
- [6. File operations: open, read, write and release](#6-file-operations-open-read-write-and-release)
- [7. ioctl](#7-ioctl)
- [8. Blocking I/O and poll](#8-blocking-io-and-poll)
- [9. sysfs, debugfs and procfs](#9-sysfs-debugfs-and-procfs)

**Part 3: Drivers for real hardware**

- [10. The driver model: buses, devices, drivers and probe](#10-the-driver-model-buses-devices-drivers-and-probe)
- [11. A platform driver from the device tree](#11-a-platform-driver-from-the-device-tree)
- [12. Reaching the hardware: registers, clocks, regulators and pins](#12-reaching-the-hardware-registers-clocks-regulators-and-pins)
- [13. GPIOs and interrupts](#13-gpios-and-interrupts)
- [14. Deferred work and time](#14-deferred-work-and-time)
- [15. Concurrency and locking](#15-concurrency-and-locking)
- [16. I2C and SPI device drivers](#16-i2c-and-spi-device-drivers)
- [17. Use a subsystem, not a character device](#17-use-a-subsystem-not-a-character-device)
- [18. Memory and DMA](#18-memory-and-dma)

**Part 4: Debugging, testing and shipping**

- [19. Debugging a driver](#19-debugging-a-driver)
- [20. Checking a driver without the board](#20-checking-a-driver-without-the-board)
- [21. In tree or out of tree, and going upstream](#21-in-tree-or-out-of-tree-and-going-upstream)
- [22. Troubleshooting by symptom](#22-troubleshooting-by-symptom)
- [23. Not covered here](#23-not-covered-here)
- [References](#references)
