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

## 1. The mental model: what a driver is

A **driver** is kernel code that knows how to operate one kind of hardware and presents it to the rest of the system through a standard interface. Userspace never touches the hardware: it opens a file in `/dev`, reads a file in `/sys`, sends a packet on a network interface, and the kernel routes each request to the driver that owns the device.

```
 userspace        cat /sys/class/hwmon/hwmon0/temp1_input      read(fd) on /dev/mychar
                                 │                                     │
 ────────────────────────────────┼─────────────────────────────────────┼────────
 kernel                          ▼                                     ▼
 subsystem        hwmon core: sysfs files, units            VFS: file_operations
                                 │                                     │
 driver           mytmp102: ->read() = one I2C register     mychar: ->read()
                                 │
 bus core         I2C core: i2c_transfer() ──► I2C controller driver (i2c-omap)
                                 │
 hardware         I2C2 controller registers ──► wires ──► TMP102 at address 0x48
```

Three ideas carry the whole note:

- **Drivers sit between a subsystem and a bus.** Above, a subsystem (hwmon, input, IIO, network, block, or the generic character device layer) defines what userspace sees. Below, a bus (platform, I2C, SPI, USB, PCI) defines how the driver reaches its device. A driver fills in callbacks for both: `->read()` for the subsystem, `->probe()` for the bus.
- **Devices and drivers are separate objects that the kernel matches.** The device tree (or ACPI, or a USB enumeration) creates *devices*. Modules register *drivers*. When a driver's match table names a device's `compatible` string or ID, the kernel calls the driver's `probe()` with that device (section 10). No driver goes looking for its hardware.
- **Most of a driver is error handling and teardown.** Getting a register to change is a few lines. Doing it so that a probe failure, a module unload, a signal or a second user leaves nothing behind is the rest. The kernel's managed resources (`devm_*`, section 11) exist to make that short.

**What a driver may expose**, from the most to the least specific:

| Interface | Userspace sees | Use it for |
|---|---|---|
| A subsystem class (section 17) | `/sys/class/hwmon/`, `/dev/input/event*`, `/sys/bus/iio/`, `/sys/class/leds/`, a network interface | Anything a subsystem exists for. Existing tools work with it unchanged |
| A character device (sections 5 to 8) | `/dev/<name>` with `read`, `write`, `ioctl`, `poll`, `mmap` | Devices with no matching subsystem, streams of data, custom protocols |
| sysfs attributes (section 9) | One value per file under `/sys/devices/...` | Configuration and status |
| debugfs (section 9) | Files under `/sys/kernel/debug/` | Debugging only, no stable ABI |

## 2. The example board, the hardware and the development loop

The board is the BeagleBone Black of the [kernel note](../fundamentals/03-kernel/kernel_v1_2026-11-10.md), booting its `zImage`, `am335x-boneblack.dtb` and modules from the SD card. The examples add four parts on a breadboard:

| Part | BeagleBone pin | SoC signal | Used in |
|---|---|---|---|
| LED with a 470 Ω to 1 kΩ resistor to GND | P9_12 | `gpio1_28`, pad `gpmc_ben1`, mode 7 | Sections 11, 13, 14 |
| Push button to GND (internal pull-up) | P9_15 | `gpio1_16`, pad `gpmc_a0`, mode 7 | Sections 13, 14 |
| TMP102 temperature sensor, address 0x48 (ADD0 to GND) | P9_19 (SCL), P9_20 (SDA), P9_3 (3.3 V), P9_1 (GND) | I2C2, already enabled by the board's device tree | Sections 16, 17 |
| BMP280 pressure sensor in SPI mode | P9_17 (CS), P9_18 (MOSI, to SDI), P9_21 (MISO, to SDO), P9_22 (SCLK) | SPI0, disabled by default | Section 16 |

The AM3358's pins are 3.3 V and not 5 V tolerant; power the sensors from P9_3 or P9_4. The pin names are those of the BeagleBone Black's device tree, which names every GPIO line after its header pin (**From source**: `gpio-line-names` in `arch/arm/boot/dts/ti/omap/am335x-boneblack.dts` lists `"P9_12"` as line 28 and `"P9_15A"` as line 16 of `gpio1`). On the board, `gpioinfo` (from libgpiod) shows the same names.

**The development loop.** Drivers are built out of tree against the kernel's build directory, exactly like the `hello.ko` of the kernel note's section 15, then copied to the board and loaded by hand:

```sh
$ export ARCH=arm CROSS_COMPILE=arm-none-linux-gnueabihf-
$ cd ~/drivers && make KDIR=~/bbb/build          # builds every module in the Makefile
$ scp ledbtn.ko root@192.168.0.50:/tmp/
# insmod /tmp/ledbtn.ko && dmesg | tail
# rmmod ledbtn
```

The `Makefile` lists every example of this note:

```make
obj-m := hellop.o mychar.o mycdev.o ledbtn.o mytmp102.o myspi.o deferred.o mydma.o

KDIR ?= $(HOME)/bbb/build

all:
	$(MAKE) -C $(KDIR) M=$(CURDIR) modules

clean:
	$(MAKE) -C $(KDIR) M=$(CURDIR) clean
```

Three habits make the loop fast and safe:

- **Keep a serial console open** (`picocom -b 115200 /dev/ttyUSB0`) and `dmesg -w` in an SSH session. A driver bug can hang the board before anything reaches the network.
- **Boot over NFS** once the module grows (kernel note, section 17): `make` then `cp *.ko /srv/nfs/bbb/root/` replaces `scp`.
- **Rebuild against the board's exact kernel.** The module's vermagic must match `uname -r` (kernel note, section 8). After any kernel rebuild, rebuild the modules too.

**Verification status.** In November 2026, on an x86_64 Ubuntu 22.04 machine:

- **Compiled:** every C example of this note (`hellop`, `mychar`, `mycdev`, `ledbtn`, `mytmp102`, `myspi`, `deferred`, `mydma`) and the userspace test program, against mainline Linux 7.3 (commit `c03c1a145`, 18 October 2026), with `W=1` and the tree's `-Werror`, using gcc 11.4. The kernel was an **x86_64** `defconfig` with `OF`, `GPIOLIB`, `I2C`, `SPI`, `HWMON` and `IIO` added, prepared with `make modules_prepare` only. The drivers are architecture-independent, so this checks their API use; it isn't an Arm build. The `modinfo` and `modpost` messages marked **Real output** come from this build.
- **Checked:** every example passes `scripts/checkpatch.pl` with no warnings. The device tree of section 11 was compiled with the 7.3 tree's `dtc` on top of `am335x-boneblack.dts` and adds no warnings to the 13 the board's own tree produces. The API names and signatures were read in the 7.3 headers, and their presence was checked again at the `v7.1-rc1` tag.
- **Not done:** nothing was cross-compiled for Arm, loaded into a kernel, or run on a board or in an emulator. Console output from the board is **Expected**.

## 3. Kernel code is not userspace code

A driver is C, but it runs with rules that differ from a program's. Breaking them doesn't give an error message: it gives a hang, a corrupted system or an oops.

| In a program | In the kernel |
|---|---|
| The C library: `printf`, `malloc`, `strcpy` | No C library. The kernel has its own: `pr_info()`, `kmalloc()`, `strscpy()`, `kstrtouint()`, lists, FIFOs, bit operations (`include/linux/`) |
| A bug kills the process | A bug can kill the whole system, or silently corrupt another subsystem's memory |
| Stack of megabytes | A fixed kernel stack per thread: 8 KiB on 32-bit Arm, 16 KiB with KASAN (`arch/arm/include/asm/thread_info.h`). No large arrays on the stack; allocate them |
| Floating point anywhere | No floating point in normal kernel code. Use integers in fixed units (millidegrees, microvolts) |
| Pointers into your own memory | User pointers (`char __user *`) can't be dereferenced: copy with `copy_from_user()` and `copy_to_user()` (section 6) |
| Errors in `errno`, return -1 | Return a **negative errno**: `return -EINVAL;`. Functions returning pointers encode errors with `ERR_PTR(-ENOMEM)`, tested with `IS_ERR()` and decoded with `PTR_ERR()` |
| One thread unless you create more | Everything is concurrent: several processes call your `read()` at once, an interrupt arrives in the middle of it, a second CPU runs the same code (section 15) |

**Contexts.** Every line of kernel code runs in one of these, and the context decides what is allowed:

| Context | Examples | May sleep? |
|---|---|---|
| Process context | `probe()`, `read()`, `write()`, `ioctl()`, sysfs `show()`/`store()`, workqueue items, threaded interrupt handlers, kernel threads | Yes: may take a mutex, allocate with `GFP_KERNEL`, wait, copy to and from userspace |
| Softirq (bottom half) context | Timer callbacks (`timer_list`), tasklets, network receive | No |
| Hard interrupt context | The primary handler passed to `request_irq()`, hrtimer callbacks | No, and must be short |
| Any code holding a spinlock, or with interrupts or preemption disabled | | No |

"May sleep" means "may wait for something and let the scheduler run another task". `mutex_lock()`, `msleep()`, `kmalloc(..., GFP_KERNEL)`, `copy_to_user()`, `i2c_transfer()` and `gpiod_set_value_cansleep()` can all sleep. Calling them from a context that can't sleep is the most common driver bug; `CONFIG_DEBUG_ATOMIC_SLEEP` turns it into a clear `BUG: sleeping function called from invalid context` message (section 19).

**Error handling with unwinding.** A function that acquires several resources releases them in reverse order on failure. The kernel idiom is `goto`, used this way only:

```c
	ret = alloc_chrdev_region(&base, 0, 1, "mycdev");
	if (ret)
		return ret;
	cls = class_create("mycdev");
	if (IS_ERR(cls)) {
		ret = PTR_ERR(cls);
		goto err_region;
	}
	...
	return 0;

err_region:
	unregister_chrdev_region(base, 1);
	return ret;
```

Two newer tools remove most of these labels: **managed resources** (`devm_*`, freed automatically when the device goes away, section 11) and **scope-based cleanup** from `include/linux/cleanup.h`, such as `guard(mutex)(&lock);`, which unlocks when the variable goes out of scope (section 15). Both are used throughout the examples.

**APIs change, and old tutorials don't.** The kernel has no stable internal API: when a function's signature changes, every in-tree caller is fixed in the same release, and out-of-tree code breaks. The changes you'll meet when reading older drivers, books and tutorials, all checked in 7.3 (**From source**):

| Older code | Linux 7.3 | Since |
|---|---|---|
| `class_create(THIS_MODULE, "name")` | `class_create("name")` (`include/linux/device/class.h`) | 6.4 |
| `int (*probe)(struct i2c_client *, const struct i2c_device_id *)`, or `.probe_new` | `int (*probe)(struct i2c_client *)`; `i2c_client_get_device_id()` if the ID is needed | 6.3 |
| `int (*remove)(...)` in platform, I2C and SPI drivers; `.remove_new` | `void (*remove)(...)` | 6.11 for platform; earlier for I2C (6.1) and SPI (6.0) |
| `from_timer()`, `del_timer()`, `del_timer_sync()` | `timer_container_of()`, `timer_delete()`, `timer_delete_sync()`, and `timer_shutdown_sync()` before freeing | 6.15 to 6.16 |
| `hrtimer_init()` then setting `.function` | `hrtimer_setup(timer, fn, clock, mode)` | 6.15 |
| `gpio_request()`, `gpio_direction_output()`, global GPIO numbers | Descriptors: `devm_gpiod_get()`, `gpiod_set_value()` (section 13) | Deprecated for years; the global numbers are no longer stable |
| `.llseek = no_llseek` | Leave `.llseek` unset: the file then refuses `lseek()` with `ESPIPE` (`fs/open.c`, `fs/read_write.c`) | 6.12 |
| `system_wq` | `system_percpu_wq` or `system_dfl_wq`; `system_wq` remains as an alias "that will be removed" (`include/linux/workqueue.h`) | 6.17 |
| Tasklets | Threaded interrupts or workqueues; `WQ_BH` workqueues when it really must run in softirq context | Tasklets still exist but are discouraged |
| `MODULE_IMPORT_NS(FOO)` | `MODULE_IMPORT_NS("FOO")`, a string | 6.13 |

When a tutorial's code doesn't compile, look up the function in the version of the kernel you build against (`git log -S '<function>' -- include/` or https://elixir.bootlin.com) before changing anything else.

## 4. A module in detail

The kernel note's `hello.c` is the smallest module. A slightly bigger one shows the rest of what every module carries (**Real**: this file compiles without warnings in the build of section 2):

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * hellop - a module with parameters
 */
#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt

#include <linux/init.h>
#include <linux/module.h>
#include <linux/moduleparam.h>

static char *whom = "world";
module_param(whom, charp, 0444);
MODULE_PARM_DESC(whom, "Who to greet");

static int count = 1;
module_param(count, int, 0644);
MODULE_PARM_DESC(count, "How many times to greet (1 to 10)");

static int __init hellop_init(void)
{
	int i;

	if (count < 1 || count > 10) {
		pr_err("count=%d is out of range\n", count);
		return -EINVAL;
	}
	for (i = 0; i < count; i++)
		pr_info("hello, %s (%d/%d)\n", whom, i + 1, count);
	return 0;
}

static void __exit hellop_exit(void)
{
	pr_info("goodbye, %s\n", whom);
}

module_init(hellop_init);
module_exit(hellop_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Your Name <you@example.com>");
MODULE_DESCRIPTION("Hello module with parameters");
```

Line by line:

- **`SPDX-License-Identifier`** on the first line is required for files in the kernel tree, and `checkpatch` asks for it everywhere.
- **`pr_fmt`** is defined before any include: every `pr_*()` in the file then starts with the module name (`hellop: hello, world (1/1)`). In drivers, prefer `dev_info(dev, ...)`, `dev_err()` and `dev_dbg()`, which add the driver and device name.
- **`module_init()` and `module_exit()`** name the functions run at `insmod` and `rmmod`. When the module is built into the kernel, the init function runs at boot and the exit function is discarded. `__init` and `__exit` put them in sections the kernel frees when they can no longer run.
- **The init function's return value** decides whether the module stays loaded. A negative errno makes `insmod` fail with that error (for `-EINVAL`, kmod's `insmod` prints `could not insert module hellop.ko: Invalid parameters`), and the module is freed. Never leave anything registered when returning an error.
- **`module_param(name, type, perm)`** makes a variable settable at load time (`insmod hellop.ko whom=board count=3`) or on the kernel command line for a built-in driver (`hellop.count=3`). `perm` is the mode of `/sys/module/hellop/parameters/<name>`: `0` hides it, `0444` makes it read-only, `0644` lets root change it at run time, after the init function has already checked it. A writable parameter that the code reads only at init is a trap; use `0444` unless the code rereads it.
- **`MODULE_LICENSE("GPL")`** is mandatory, and not only legally. Without it, `modpost` refuses the module (**Real output**: `ERROR: modpost: nolic.ko: missing MODULE_LICENSE()`). A non-GPL license also hides every symbol exported with `EXPORT_SYMBOL_GPL()`, which includes most of the driver core, GPIO, interrupts and device tree APIs: the module then fails to load with `Unknown symbol`.
- **`MODULE_DESCRIPTION()`** is checked too. Without it (**Real output**): `WARNING: modpost: hellop.ko: missing MODULE_DESCRIPTION()`.

What the build records, **Real output** of `modinfo` on the x86_64 compile-check build:

```
$ modinfo hellop.ko
description:    Hello module with parameters
author:         Your Name <you@example.com>
license:        GPL
vermagic:       7.3.0 SMP preempt mod_unload
parm:           whom:Who to greet (charp)
parm:           count:How many times to greet (1 to 10) (int)
```

On the board the vermagic is the board kernel's release and options, ending in `ARMv7 p2v8` (kernel note, section 8).

**Log levels.** `pr_info()` is `printk(KERN_INFO ...)`. The levels, from `include/linux/kern_levels.h`, are `emerg` (0), `alert`, `crit`, `err` (3), `warn` (4), `notice`, `info` (6) and `debug` (7). `pr_debug()` and `dev_dbg()` compile to nothing unless `DEBUG` is defined for the file or `CONFIG_DYNAMIC_DEBUG` is on, in which case they can be switched on at run time (section 19). Every message ends with `\n`; without it, the next message may be glued onto the same line.

**Several source files, one module.** The object list names the module, and `<module>-y` lists its parts:

```make
obj-m    := mydrv.o
mydrv-y  := core.o i2c.o sysfs.o
```

The module's name is then `mydrv`; no source file may be called `mydrv.c`.

**Exporting symbols to other modules.** A module that provides functions to others marks them:

```c
int mylib_compute(int x) { ... }
EXPORT_SYMBOL_GPL(mylib_compute);
```

Only exported symbols are visible to other modules. `EXPORT_SYMBOL_GPL` restricts users to GPL-compatible modules, and is what new code uses. `EXPORT_SYMBOL_NS_GPL(sym, "MYLIB")` puts the symbol in a **namespace**; users must declare `MODULE_IMPORT_NS("MYLIB");`. Without it, `modpost` stops with `module uses symbol '...' from namespace 'MYLIB', but does not import it.`, and a module built anyway is refused at load time with the same message (**From source**: `scripts/mod/modpost.c`, `kernel/module/main.c`). Out of tree, the user module needs the provider's `Module.symvers` (`KBUILD_EXTRA_SYMBOLS`, kernel note section 15), and `modprobe` loads the provider first once both are installed and `depmod` has run.
