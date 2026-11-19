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
obj-m := hellop.o mychar.o mycdev.o mydebugfs.o myip.o ledbtn.o mytmp102.o myspi.o deferred.o mydma.o

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

- **Compiled:** every C example of this note (`hellop`, `mychar`, `mycdev`, `mydebugfs`, `myip`, `ledbtn`, `mytmp102`, `myspi`, `deferred`, `mydma`) and the userspace test program, against mainline Linux 7.3 (commit `c03c1a145`, 18 October 2026), with `W=1` and the tree's `-Werror`, using gcc 11.4. The kernel was an **x86_64** `defconfig` with `OF`, `GPIOLIB`, `I2C`, `SPI`, `HWMON` and `IIO` added, prepared with `make modules_prepare` only. The drivers are architecture-independent, so this checks their API use; it isn't an Arm build. The `modinfo` and `modpost` messages marked **Real output** come from this build.
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

## 5. Character devices: numbers, cdev and the misc shortcut

A **character device** is a file in `/dev` whose operations go to a driver. "Character" means a stream of bytes, as opposed to a block device (fixed-size blocks behind a page cache, for storage) or a network interface (no file at all).

**Device numbers.** The file in `/dev` is only a name. What connects it to the driver is the pair of numbers it carries:

```
# ls -l /dev/ttyS0 /dev/mychar
crw-rw----    1 root     dialout     4,  64 ... /dev/ttyS0
crw-rw-rw-    1 root     root       10, 123 ... /dev/mychar
```

The `c` is "character device"; `4, 64` is the **major** and **minor** number. The kernel finds the driver from the major (and the minor range it registered), and the driver uses the minor to tell its devices apart. `/proc/devices` lists the registered majors. The kernel stores both in a `dev_t`, built with `MKDEV(major, minor)` and split with `MAJOR()` and `MINOR()`.

Nobody creates these files by hand on a current system. When a driver registers a device with the driver core (`device_create()` or `misc_register()`), the kernel creates the node in **devtmpfs** (mounted on `/dev` by the kernel, `CONFIG_DEVTMPFS_MOUNT`), and sends a uevent so that udev or mdev can fix permissions and add symbolic links ([rootfs note](../fundamentals/04-rootfs/rootfs_v1_2026-11-11.md)).

**The shortcut: a misc device.** For a driver that needs one device file, the misc subsystem does all the registration: major 10, a minor chosen dynamically, the `/dev` node, and the class entry under `/sys/class/misc/`. The complete `mychar` driver of sections 5 to 8 registers itself with these lines (**Real**: compiled in the build of section 2):

```c
static const struct file_operations mychar_fops = {
	.owner		= THIS_MODULE,
	.read		= mychar_read,
	.write		= mychar_write,
	.poll		= mychar_poll,
	.unlocked_ioctl	= mychar_ioctl,
	.compat_ioctl	= compat_ptr_ioctl,
};

static struct miscdevice mychar_misc = {
	.minor	= MISC_DYNAMIC_MINOR,
	.name	= "mychar",
	.fops	= &mychar_fops,
	.mode	= 0666,
};

module_misc_device(mychar_misc);
```

`module_misc_device()` generates the init and exit functions that call `misc_register()` and `misc_deregister()`. `.mode` sets the permissions of the node in devtmpfs (**From source**: `drivers/char/misc.c`); `0666` lets any user open it, which is fine for an example and rarely right for a product, where a udev rule should decide.

**The full way: a region, a cdev, a class and a device.** Use it when the driver needs several minors, a major of its own, or its own class name in `/sys/class/`. Four objects, each undone in reverse order (`mycdev.c`, **Real**: compiled):

```c
#define MYCDEV_COUNT	2		/* /dev/mycdev0 and /dev/mycdev1 */
#define MYCDEV_BUFSZ	256

struct mycdev {
	struct cdev cdev;
	struct device *dev;
	char buf[MYCDEV_BUFSZ];
	size_t len;
};

static dev_t mycdev_base;
static struct class *mycdev_class;
static struct mycdev *mycdevs;

static int __init mycdev_init(void)
{
	int i, ret;

	ret = alloc_chrdev_region(&mycdev_base, 0, MYCDEV_COUNT, "mycdev");
	if (ret)
		return ret;

	mycdev_class = class_create("mycdev");
	if (IS_ERR(mycdev_class)) {
		ret = PTR_ERR(mycdev_class);
		goto err_region;
	}

	mycdevs = kcalloc(MYCDEV_COUNT, sizeof(*mycdevs), GFP_KERNEL);
	if (!mycdevs) {
		ret = -ENOMEM;
		goto err_class;
	}

	for (i = 0; i < MYCDEV_COUNT; i++) {
		dev_t devt = MKDEV(MAJOR(mycdev_base), MINOR(mycdev_base) + i);

		cdev_init(&mycdevs[i].cdev, &mycdev_fops);
		mycdevs[i].cdev.owner = THIS_MODULE;
		ret = cdev_add(&mycdevs[i].cdev, devt, 1);
		if (ret)
			goto err_devices;

		mycdevs[i].dev = device_create(mycdev_class, NULL, devt, NULL,
					       "mycdev%d", i);
		if (IS_ERR(mycdevs[i].dev)) {
			ret = PTR_ERR(mycdevs[i].dev);
			cdev_del(&mycdevs[i].cdev);
			goto err_devices;
		}
	}

	pr_info("major %d, minors %d-%d\n", MAJOR(mycdev_base),
		MINOR(mycdev_base), MINOR(mycdev_base) + MYCDEV_COUNT - 1);
	return 0;

err_devices:
	while (--i >= 0) {
		device_destroy(mycdev_class, mycdevs[i].cdev.dev);
		cdev_del(&mycdevs[i].cdev);
	}
	kfree(mycdevs);
err_class:
	class_destroy(mycdev_class);
err_region:
	unregister_chrdev_region(mycdev_base, MYCDEV_COUNT);
	return ret;
}
```

| Call | Does | Undone by |
|---|---|---|
| `alloc_chrdev_region(&base, first_minor, count, name)` | Reserves `count` minors under a free major, chosen counting down from 254 (then from 511) (`include/linux/fs.h`, `fs/char_dev.c`). The name appears in `/proc/devices` | `unregister_chrdev_region()` |
| `class_create(name)` | Creates `/sys/class/<name>/` | `class_destroy()` |
| `cdev_init()` + `cdev_add()` | Connects the numbers to the `file_operations`. From this call on, `open()` can reach the driver | `cdev_del()` |
| `device_create(class, parent, devt, drvdata, fmt, ...)` | Creates the device object, its sysfs directory, and the `/dev` node | `device_destroy()` |

Two orders matter. Call `cdev_add()` only when the driver is ready for an `open()`, because one can arrive immediately. And on removal, remove the device node first and free the memory last; an open file can still call `read()` until it's closed, which is why `cdev.owner = THIS_MODULE` keeps the module from being unloaded while a file is open. A driver whose per-device structure lives as long as a `struct device` uses `cdev_device_add()`, which adds both in the right order.

**Fixed majors** (`register_chrdev_region()` with a number from `Documentation/admin-guide/devices.txt`) are for devices with an official number. New drivers allocate dynamically.

## 6. File operations: open, read, write and release

`struct file_operations` is the table of functions the VFS calls for each system call on the device file:

| System call | Callback | Notes |
|---|---|---|
| `open()` | `int (*open)(struct inode *, struct file *)` | Optional. Allocate per-open state here, store it in `file->private_data` |
| `close()` (last reference) | `int (*release)(struct inode *, struct file *)` | Called once per `struct file`, after the last `close()` of every duplicated descriptor. Free per-open state here |
| `read()` | `ssize_t (*read)(struct file *, char __user *, size_t, loff_t *)` | Return bytes copied, 0 for end of file, or a negative errno |
| `write()` | `ssize_t (*write)(struct file *, const char __user *, size_t, loff_t *)` | Return bytes consumed, or a negative errno |
| `ioctl()` | `long (*unlocked_ioctl)(struct file *, unsigned int, unsigned long)` | Section 7 |
| `poll()`, `select()`, `epoll` | `__poll_t (*poll)(struct file *, struct poll_table_struct *)` | Section 8 |
| `lseek()` | `loff_t (*llseek)(struct file *, loff_t, int)` | Unset: `lseek()` fails with `ESPIPE`. `default_llseek` for a file with positions |
| `mmap()` | `int (*mmap)(struct file *, struct vm_area_struct *)` | Map device memory or a buffer into the process (not covered) |

**Finding your data.** Three ways, from the simplest:

- **One device, global state:** the `mychar` example keeps its FIFO in static variables.
- **A misc device:** `misc_open()` sets `file->private_data` to the `struct miscdevice` before calling your `open` (**From source**: `drivers/char/misc.c`). Embed the `miscdevice` in your structure and use `container_of()` to get back to it.
- **A cdev:** the inode points to the cdev, and the cdev is embedded in your structure, so `open` does

  ```c
  static int mycdev_open(struct inode *inode, struct file *file)
  {
  	/* Find our structure from the cdev embedded in it. */
  	file->private_data = container_of(inode->i_cdev, struct mycdev, cdev);
  	return 0;
  }
  ```

  and every other callback reads `file->private_data`. `container_of(ptr, type, member)` is the kernel's way of going from a member to the structure that contains it; it's everywhere.

**User memory.** `buf` in `read()` and `write()` is an address in the calling process. Dereferencing it directly is a bug, even when it seems to work: the page may be absent (the access must be allowed to fault and sleep), the address may point into the kernel (a security hole), and on systems with privileged-access protection the CPU traps the access. Always copy:

| Function | Returns |
|---|---|
| `copy_to_user(to, from, n)`, `copy_from_user(to, from, n)` | The number of bytes **not** copied; 0 on success. Return `-EFAULT` when it isn't 0 |
| `get_user(x, ptr)`, `put_user(x, ptr)` | One value of the pointer's type; 0 or `-EFAULT` |
| `simple_read_from_buffer()`, `simple_write_to_buffer()` | Bytes copied, handling `*ppos` and the bounds. Ideal for a device that is a small buffer |
| `kfifo_to_user()`, `kfifo_from_user()` | Copy between a kfifo and userspace; the byte count in their last argument |

All of them can sleep, so never call them with a spinlock held.

**The `mychar` read path.** Section 8 explains the loop; the copy itself is one call, done with the mutex held so that two readers can't take the same bytes (**Real**: compiled):

```c
	ret = kfifo_to_user(&fifo, buf, count, &copied);
	mutex_unlock(&fifo_lock);
	if (ret)
		return ret;

	wake_up_interruptible(&writeq);
	return copied;
```

**Return values that userspace understands.** A `read()` that returns fewer bytes than asked is normal; `cat` calls again. A `read()` that returns 0 means end of file and stops `cat`. `-EAGAIN` with `O_NONBLOCK`, `-EFAULT` for a bad pointer, `-EINVAL` for a bad argument, `-ERESTARTSYS` when a signal interrupted a wait (the kernel turns it into `EINTR` or restarts the call). `write()` must not return 0 for a non-zero count, or a well-behaved program loops forever.

**`*ppos`** is the file position. A stream device like `mychar` ignores it. A device that represents a fixed buffer (`mycdev`) uses it, and then sets `.llseek = default_llseek` so that `lseek()` works.

## 7. ioctl

`ioctl()` carries commands that aren't reads or writes: reset the device, get statistics, set a mode. Each command is a 32-bit number that encodes its direction, the size of its argument, a "magic" type byte and a sequence number, built with macros from `<linux/ioctl.h>` (**From source**: `include/uapi/asm-generic/ioctl.h`, 8 bits of number, 8 of type, 14 of size, 2 of direction).

**The commands go in a UAPI header** that both the driver and userspace include (`mychar.h`, **Real**: compiled into both):

```c
/* SPDX-License-Identifier: GPL-2.0 WITH Linux-syscall-note */
#ifndef _UAPI_MYCHAR_H
#define _UAPI_MYCHAR_H

#include <linux/ioctl.h>
#include <linux/types.h>

struct mychar_stats {
	__u32 used;	/* bytes waiting in the FIFO */
	__u32 size;	/* FIFO capacity in bytes */
};

#define MYCHAR_IOC_MAGIC	0xBB
#define MYCHAR_IOC_RESET	_IO(MYCHAR_IOC_MAGIC, 0)
#define MYCHAR_IOC_GET_STATS	_IOR(MYCHAR_IOC_MAGIC, 1, struct mychar_stats)

#endif /* _UAPI_MYCHAR_H */
```

- **`_IO`** has no argument, **`_IOR`** returns data to userspace ("read" from the caller's point of view), **`_IOW`** sends data to the driver, **`_IOWR`** both. The size of the third argument becomes part of the number, so changing the structure changes the command, which is what you want.
- **Pick a magic number nobody uses.** `Documentation/userspace-api/ioctl/ioctl-number.rst` lists the registered ones; `0xBB` isn't in the 7.3 list. A collision doesn't break your driver, but it lets a program that opened the wrong file run another driver's command.
- **Use fixed-size types** (`__u32`, `__u64`) and lay out structures so they have no padding and the same layout on 32-bit and 64-bit userspace. Then `.compat_ioctl = compat_ptr_ioctl` is enough for a 32-bit program on a 64-bit kernel.
- The `Linux-syscall-note` exception lets non-GPL programs include the header.

**The driver side** (**Real**: compiled):

```c
static long mychar_ioctl(struct file *file, unsigned int cmd, unsigned long arg)
{
	struct mychar_stats st = { };

	switch (cmd) {
	case MYCHAR_IOC_RESET:
		scoped_guard(mutex, &fifo_lock)
			kfifo_reset(&fifo);
		wake_up_interruptible(&writeq);
		return 0;

	case MYCHAR_IOC_GET_STATS:
		scoped_guard(mutex, &fifo_lock) {
			st.used = kfifo_len(&fifo);
			st.size = kfifo_size(&fifo);
		}
		if (copy_to_user((void __user *)arg, &st, sizeof(st)))
			return -EFAULT;
		return 0;

	default:
		return -ENOTTY;
	}
}
```

- **Unknown commands return `-ENOTTY`** ("inappropriate ioctl for device"), which is what userspace tools expect, not `-EINVAL`.
- **Zero the structure** (`= { }`) before filling it. Any byte you copy to userspace that you didn't set leaks old kernel memory.
- **Check permissions** for commands that change the device for everyone: `if (!capable(CAP_SYS_ADMIN)) return -EPERM;`.
- **Prefer something else when it fits.** Every ioctl is new ABI that you must keep forever. A sysfs attribute (section 9) is better for a single value, and a subsystem (section 17) is better than either when one exists.

## 8. Blocking I/O and poll

A `read()` on an empty `mychar` shouldn't return 0, which would mean "end of file", and shouldn't spin. It should put the caller to sleep until data arrives, unless the file was opened with `O_NONBLOCK`. The kernel's mechanism is the **wait queue**: a list of sleeping tasks, woken by whoever makes the condition true.

```c
static DEFINE_KFIFO(fifo, unsigned char, MYCHAR_FIFO_SIZE);
static DEFINE_MUTEX(fifo_lock);		/* serializes readers and writers */
static DECLARE_WAIT_QUEUE_HEAD(readq);	/* readers wait for data */
static DECLARE_WAIT_QUEUE_HEAD(writeq);	/* writers wait for room */

static ssize_t mychar_read(struct file *file, char __user *buf,
			   size_t count, loff_t *ppos)
{
	unsigned int copied;
	int ret;

	for (;;) {
		if (mutex_lock_interruptible(&fifo_lock))
			return -ERESTARTSYS;
		if (!kfifo_is_empty(&fifo))
			break;			/* data, and the lock is held */
		mutex_unlock(&fifo_lock);

		if (file->f_flags & O_NONBLOCK)
			return -EAGAIN;
		if (wait_event_interruptible(readq, !kfifo_is_empty(&fifo)))
			return -ERESTARTSYS;	/* a signal arrived */
	}

	ret = kfifo_to_user(&fifo, buf, count, &copied);
	mutex_unlock(&fifo_lock);
	if (ret)
		return ret;

	wake_up_interruptible(&writeq);
	return copied;
}
```

The pattern, which every blocking driver repeats:

1. **Check the condition with the lock held.** If it's true, keep the lock and go on.
2. **Otherwise drop the lock before sleeping.** Sleeping with the lock held would block the writer that is supposed to wake you.
3. **`wait_event_interruptible(queue, condition)`** sleeps until `condition` is true, re-checking it after each wake-up. It returns non-zero when a signal arrives, and the driver returns `-ERESTARTSYS` so that Ctrl-C works.
4. **Loop**, because another reader may have taken the data between the wake-up and the lock.
5. **The other side wakes the queue** after changing the condition: `mychar_write()` calls `wake_up_interruptible(&readq)` after adding bytes.

The `_interruptible` variants are the right default for anything a user waits on. `wait_event_timeout()` and `wait_event_interruptible_timeout()` add a timeout in jiffies (`msecs_to_jiffies(500)`); `wait_event()` without `interruptible` makes the process unkillable while it waits, so use it only for short waits on hardware.

**poll.** `poll()`, `select()` and `epoll` ask the driver two questions: "which operations would not block right now?" and "which wait queues should wake me when that changes?". The driver answers both in one callback (**Real**: compiled):

```c
static __poll_t mychar_poll(struct file *file, poll_table *wait)
{
	__poll_t mask = 0;

	poll_wait(file, &readq, wait);
	poll_wait(file, &writeq, wait);

	guard(mutex)(&fifo_lock);
	if (!kfifo_is_empty(&fifo))
		mask |= EPOLLIN | EPOLLRDNORM;
	if (!kfifo_is_full(&fifo))
		mask |= EPOLLOUT | EPOLLWRNORM;
	return mask;
}
```

`poll_wait()` doesn't sleep: it registers the queues. The kernel sleeps afterwards, if the mask is empty, and calls `->poll()` again after any wake-up of those queues. So the only rule is that whatever changes the answer must wake one of the registered queues, which `mychar_read()` and `mychar_write()` already do. The same callback serves `select`, `poll` and `epoll`.

**Testing it from userspace.** A short program exercises each path (**Real**: compiled with the build machine's gcc; the output below is **Expected**, since the module wasn't loaded):

```c
	fd = open("/dev/mychar", O_RDWR | O_NONBLOCK);
	n = read(fd, buf, sizeof(buf));			/* empty FIFO */
	printf("read on empty FIFO: %zd (%s)\n", n, n < 0 ? strerror(errno) : "data");
	write(fd, "hello", 5);
	ioctl(fd, MYCHAR_IOC_GET_STATS, &st);
	printf("stats: used %u of %u bytes\n", st.used, st.size);
	pfd.fd = fd;
	pfd.events = POLLIN;
	if (poll(&pfd, 1, 1000) == 1 && (pfd.revents & POLLIN)) { ... read ... }
	ioctl(fd, 0);					/* unknown command */
```

```
# ./mychar_test
read on empty FIFO: -1 (Resource temporarily unavailable)
stats: used 5 of 4096 bytes
poll: readable, read 5 bytes: "hello"
unknown ioctl: Inappropriate ioctl for device
```

And from the shell, the blocking path: `cat /dev/mychar` waits in one terminal; `echo hi > /dev/mychar` in another makes it print `hi`. Ctrl-C ends the `cat` through the `-ERESTARTSYS` path.

## 9. sysfs, debugfs and procfs

Three virtual filesystems carry driver information besides `/dev`. They're for different readers:

| | sysfs (`/sys`) | debugfs (`/sys/kernel/debug`) | procfs (`/proc`) |
|---|---|---|---|
| For | Configuration and status that tools and scripts rely on | Developers | Processes and the kernel's own state |
| ABI | **Stable.** Every attribute is ABI, documented in `Documentation/ABI/` | None. Can change at any time; may not be mounted | Stable, but closed to new device files |
| Format | One value per file, text | Anything | Anything, historically |
| Use it for in a driver | A mode, a threshold, a counter, a state | Register dumps, internal statistics, fault injection | Nothing new: don't add driver files here |

**sysfs attributes on the device.** Every `struct device` has a directory under `/sys/devices/`, reachable through `/sys/bus/<bus>/devices/<name>/` and, for classes, `/sys/class/<class>/<name>/`. A driver adds files to it with attributes. The `ledbtn` driver of section 11 has three; one of them (**Real**: compiled):

```c
static ssize_t led_show(struct device *dev, struct device_attribute *attr,
			char *buf)
{
	struct ledbtn *lb = dev_get_drvdata(dev);

	guard(mutex)(&lb->lock);
	return sysfs_emit(buf, "%d\n", lb->led_on);
}

static ssize_t led_store(struct device *dev, struct device_attribute *attr,
			 const char *buf, size_t count)
{
	struct ledbtn *lb = dev_get_drvdata(dev);
	bool on;
	int ret;

	ret = kstrtobool(buf, &on);
	if (ret)
		return ret;

	guard(mutex)(&lb->lock);
	if (lb->blink_ms)
		return -EBUSY;
	ledbtn_set_led(lb, on);
	return count;
}
static DEVICE_ATTR_RW(led);

static struct attribute *ledbtn_attrs[] = {
	&dev_attr_led.attr,
	&dev_attr_blink_ms.attr,
	&dev_attr_presses.attr,
	NULL
};
ATTRIBUTE_GROUPS(ledbtn);
```

- **`DEVICE_ATTR_RW(led)`** declares `dev_attr_led` with mode 0644 and expects functions named exactly `led_show` and `led_store`. `DEVICE_ATTR_RO` (0444, `_show` only) and `DEVICE_ATTR_WO` (0200, `_store` only) follow the same naming.
- **`show()` writes into a page-sized buffer** with `sysfs_emit()`, which knows the bounds; `sprintf()` into `buf` is what old code does and `checkpatch` flags it.
- **`store()` parses** with the `kstrto*()` family (`kstrtouint`, `kstrtobool`, `kstrtoint`), which reject trailing garbage and accept the newline `echo` adds. It returns `count` on success.
- **Register the group through the driver**, `.driver.dev_groups = ledbtn_groups` (section 11). The driver core then creates the files after `probe()` succeeds and removes them before `remove()` (`drivers/base/dd.c`), so userspace can never see a file whose device isn't ready. Creating attributes by hand with `device_create_file()` in `probe()` opens a window where udev has already seen the device but the files don't exist yet.
- **Let userspace wait for a change.** `sysfs_notify(&dev->kobj, NULL, "presses")` wakes anyone polling the file. A sysfs file always reports `EPOLLERR|EPOLLPRI` to `poll()` after a notification (**From source**: `fs/kernfs/file.c`), so a program reads the file, polls for `POLLPRI`, then `lseek(fd, 0, SEEK_SET)` and reads again.

On the board (**Expected**):

```
# cd /sys/bus/platform/devices/ledbtn/
# echo 1 > led
# echo 250 > blink_ms
# echo 1 > led
sh: write error: Device or resource busy
# cat presses
3
```

**debugfs** for everything you only need while developing (`mydebugfs.c`, **Real**: compiled):

```c
static struct dentry *dir;
static u32 errors;
static u32 irqs;

static int status_show(struct seq_file *s, void *unused)
{
	seq_printf(s, "irqs:   %u\n", irqs);
	seq_printf(s, "errors: %u\n", errors);
	return 0;
}
DEFINE_SHOW_ATTRIBUTE(status);

static int __init mydebugfs_init(void)
{
	/* Errors are ignored on purpose: debugfs is optional. */
	dir = debugfs_create_dir("mydebugfs", NULL);
	debugfs_create_u32("errors", 0644, dir, &errors);
	debugfs_create_file("status", 0444, dir, NULL, &status_fops);
	return 0;
}

static void __exit mydebugfs_exit(void)
{
	debugfs_remove(dir);	/* removes everything below it too */
}
```

`debugfs_create_u32()` exposes a variable as a readable and writable file in one line; `DEFINE_SHOW_ATTRIBUTE(status)` turns a `seq_file` show function into `status_fops`. The debugfs API is designed so that drivers **don't check its return values**: when debugfs is disabled or the directory couldn't be created, the later calls do nothing, and the driver must work the same. `mount -t debugfs none /sys/kernel/debug` if it isn't mounted, then `cat /sys/kernel/debug/mydebugfs/status`.

**procfs.** `/proc/<pid>/`, `/proc/interrupts`, `/proc/iomem` and the other kernel files are where you *read* information about drivers (section 19). New driver files in `/proc` aren't accepted upstream; old tutorials that create them with `proc_create()` predate sysfs.

## 10. The driver model: buses, devices, drivers and probe

Sections 5 to 9 created devices from a module's init function. Real hardware works the other way round: the device exists first, described by the device tree, and the driver is attached to it by the kernel's **driver model** (`drivers/base/`).

**Three kinds of objects:**

| Object | Represents | Created by | In sysfs |
|---|---|---|---|
| `struct bus_type` | A way devices are connected and found: platform, I2C, SPI, USB, PCI, MDIO | The bus core, once | `/sys/bus/<bus>/` |
| `struct device` | One piece of hardware | The device tree code at boot, a bus controller driver when it scans (I2C children of an adapter, USB enumeration), or code | `/sys/devices/...`, linked from `/sys/bus/<bus>/devices/` |
| `struct device_driver` | Code that can operate some devices | A module or built-in code calling `platform_driver_register()`, `i2c_add_driver()`... | `/sys/bus/<bus>/drivers/<driver>/` |

**The platform bus** is for devices that can't be discovered: SoC peripherals at fixed addresses, and anything else described only by the device tree. Every node under the SoC's buses with a `compatible` and `status = "okay"`, and every such node directly under the root, becomes a platform device at boot (`drivers/of/platform.c`). Its name comes from its address and node name (`44e09000.serial`), or the node name alone when it has no address (`ledbtn`, **From source**: `of_device_make_bus_id()` in `drivers/of/device.c`). Devices on real buses, such as I2C or SPI, are children of their controller's node, and the controller's driver creates them (section 16).

**Matching.** Whenever a device or a driver is added to a bus, the bus compares them. For the platform bus, in this order (`drivers/base/platform.c`, `platform_match()`): a `driver_override` written in sysfs, then the device tree `compatible` strings against the driver's `of_match_table`, then ACPI, then the driver's `id_table`, then the device and driver names. The `compatible` list of a node goes from the most specific to the most generic, and the first string a driver knows wins.

**Probe.** On a match, the driver core calls the driver's `probe()` with the device. What happens next depends on the return value (**From source**: `drivers/base/dd.c`, `really_probe()` and `call_driver_probe()`):

| `probe()` returns | Result |
|---|---|
| `0` | The driver is bound. Its `dev_groups` attributes are created, and `/sys/bus/<bus>/devices/<dev>/driver` points to it |
| `-EPROBE_DEFER` | Not now: a resource isn't ready yet (a clock, a GPIO controller, a regulator whose driver hasn't probed). The device goes on a list and is retried each time another driver binds |
| `-ENODEV` or `-ENXIO` | "This isn't mine after all". Logged only at debug level |
| Anything else | Failure, logged as `probe with driver <name> failed with error <errno>`. The device stays without a driver |

Deferred probing is how the kernel handles initialization order without anyone writing it down: a driver that needs a GPIO asks for it, gets `-EPROBE_DEFER` if the GPIO controller hasn't probed yet, and returns that error unchanged. Use **`dev_err_probe(dev, err, "message")`** for every error in `probe()`: it prints the message for real errors, stays silent for `-EPROBE_DEFER` but records the reason, and returns `err`. The reasons are listed in `/sys/kernel/debug/devices_deferred` (**From source**: `drivers/base/dd.c`), which is the first place to look when a device never appears.

**Module autoloading.** `MODULE_DEVICE_TABLE(of, ledbtn_of_match)` copies the match table into the module as aliases (**Real output**):

```
$ modinfo -F alias ledbtn.ko
of:N*T*Cmycompany,ledbtnC*
of:N*T*Cmycompany,ledbtn
```

At boot, the kernel announces each device with a uevent carrying its `MODALIAS`, for this node `of:NledbtnT(null)Cmycompany,ledbtn` (**Expected**, from the format in `drivers/of/module.c`: name, then the absent `device_type`, then each `compatible`). udev runs `modprobe` with it, `modprobe` matches it against `modules.alias`, and the driver loads (kernel note, section 14). Without `MODULE_DEVICE_TABLE`, the driver works when loaded by hand but never loads by itself.

**Binding by hand**, useful while developing:

```sh
# ls -l /sys/bus/platform/devices/ledbtn/driver       # bound to which driver?
# echo ledbtn > /sys/bus/platform/drivers/ledbtn/unbind
# echo ledbtn > /sys/bus/platform/drivers/ledbtn/bind
```

`unbind` calls the driver's removal path without unloading the module, which is the fastest way to test it.

## 11. A platform driver from the device tree

The `ledbtn` driver controls an LED and reads a button, both on GPIOs, as described by a device tree node. It's the pattern of most SoC peripheral drivers: a node, a match table, a `probe()` that gets every resource, and no `remove()`.

**The device tree.** The node goes in the board's own `.dts`, which includes the BeagleBone Black's (kernel note, section 12). This file holds every device of this note (**Real**: compiled with the 7.3 `dtc`, 99,277 bytes):

```dts
// SPDX-License-Identifier: GPL-2.0
/*
 * BeagleBone Black with the example devices of the driver note:
 * an LED on P9_12, a button on P9_15, a TMP102 on I2C2 (P9_19/P9_20)
 * and a BMP280 on SPI0 (P9_17, P9_18, P9_21, P9_22).
 */
/dts-v1/;
#include "am335x-boneblack.dts"

/ {
	model = "TI AM335x BeagleBone Black (driver note)";

	ledbtn {
		compatible = "mycompany,ledbtn";
		pinctrl-names = "default";
		pinctrl-0 = <&ledbtn_pins>;
		led-gpios = <&gpio1 28 GPIO_ACTIVE_HIGH>;	/* P9_12 */
		button-gpios = <&gpio1 16 GPIO_ACTIVE_LOW>;	/* P9_15, to GND */
	};
};

&am33xx_pinmux {
	ledbtn_pins: ledbtn-pins {
		pinctrl-single,pins = <
			AM33XX_PADCONF(AM335X_PIN_GPMC_BEN1, PIN_OUTPUT, MUX_MODE7)	/* P9_12 gpio1_28 */
			AM33XX_PADCONF(AM335X_PIN_GPMC_A0, PIN_INPUT_PULLUP, MUX_MODE7)	/* P9_15 gpio1_16 */
		>;
	};

	spi0_pins: spi0-pins {
		pinctrl-single,pins = <
			AM33XX_PADCONF(AM335X_PIN_SPI0_SCLK, PIN_INPUT, MUX_MODE0)	/* P9_22 */
			AM33XX_PADCONF(AM335X_PIN_SPI0_D0, PIN_INPUT_PULLUP, MUX_MODE0)	/* P9_21, MISO */
			AM33XX_PADCONF(AM335X_PIN_SPI0_D1, PIN_OUTPUT, MUX_MODE0)	/* P9_18, MOSI */
			AM33XX_PADCONF(AM335X_PIN_SPI0_CS0, PIN_OUTPUT, MUX_MODE0)	/* P9_17 */
		>;
	};
};

&i2c2 {
	temperature-sensor@48 {
		compatible = "mycompany,mytmp102";
		reg = <0x48>;
	};
};

&spi0 {
	pinctrl-names = "default";
	pinctrl-0 = <&spi0_pins>;
	status = "okay";

	pressure-sensor@0 {
		compatible = "mycompany,mybmp280";
		reg = <0>;				/* chip select 0 */
		spi-max-frequency = <1000000>;
	};
};
```

- **`compatible = "mycompany,ledbtn"`** is the only link to the driver. The vendor prefix comes before the comma. `mycompany` is a placeholder: upstream, prefixes are registered in `Documentation/devicetree/bindings/vendor-prefixes.yaml`, and `make dtbs_check` warns about unknown ones. Don't borrow a short name that looks free; `acme`, for example, is registered to a real company.
- **`led-gpios`** is a GPIO reference: the controller (`&gpio1`), the line (28), and flags. The driver asks for it by the part before `-gpios`: `devm_gpiod_get(dev, "led", ...)`. `GPIO_ACTIVE_LOW` on the button means "pressed" reads as 1 in the driver, although the pin is at 0 V; the polarity lives in the device tree, not in the code.
- **`pinctrl-0`** selects the pad configuration: the AM3358's pads each carry up to eight functions, and mode 7 is GPIO. The driver core applies the `default` state before calling `probe()` (`pinctrl_bind_pins()` in `drivers/base/dd.c`), so the driver never mentions pins. `PIN_INPUT_PULLUP` enables the internal pull-up, so a button to ground needs no resistor.
- **No `reg`**: the device has no registers of its own, so the node sits at the root, and its device is named `ledbtn`.

Build it as in the kernel note: put the file in `arch/arm/boot/dts/ti/omap/`, add `dtb-$(CONFIG_SOC_AM33XX) += am335x-boneblack-mydrivers.dtb` to the `Makefile` there, then `make O=~/bbb/build ti/omap/am335x-boneblack-mydrivers.dtb`, and point the bootloader's `fdt` line at it.

**The driver.** The structure holds everything the device needs, and `probe()` fills it (`ledbtn.c`, **Real**: compiled; the sysfs part is in section 9, the interrupt and work parts in sections 13 and 14):

```c
struct ledbtn {
	struct device *dev;
	struct gpio_desc *led;
	struct gpio_desc *button;
	struct delayed_work blink_work;
	struct mutex lock;		/* protects everything below */
	bool led_on;
	unsigned int blink_ms;
	unsigned long presses;
	ktime_t last_press;
};

static int ledbtn_probe(struct platform_device *pdev)
{
	struct device *dev = &pdev->dev;
	struct ledbtn *lb;
	int irq, ret;

	lb = devm_kzalloc(dev, sizeof(*lb), GFP_KERNEL);
	if (!lb)
		return -ENOMEM;
	lb->dev = dev;
	platform_set_drvdata(pdev, lb);

	ret = devm_mutex_init(dev, &lb->lock);
	if (ret)
		return ret;

	lb->led = devm_gpiod_get(dev, "led", GPIOD_OUT_LOW);
	if (IS_ERR(lb->led))
		return dev_err_probe(dev, PTR_ERR(lb->led),
				     "cannot get the led GPIO\n");

	lb->button = devm_gpiod_get(dev, "button", GPIOD_IN);
	if (IS_ERR(lb->button))
		return dev_err_probe(dev, PTR_ERR(lb->button),
				     "cannot get the button GPIO\n");

	ret = devm_delayed_work_autocancel(dev, &lb->blink_work, ledbtn_blink);
	if (ret)
		return ret;

	irq = gpiod_to_irq(lb->button);
	if (irq < 0)
		return dev_err_probe(dev, irq, "button GPIO has no interrupt\n");

	ret = devm_request_threaded_irq(dev, irq, NULL, ledbtn_irq_thread,
					IRQF_TRIGGER_FALLING | IRQF_ONESHOT,
					dev_name(dev), lb);
	if (ret)
		return dev_err_probe(dev, ret, "cannot request IRQ %d\n", irq);

	dev_info(dev, "ready, button on IRQ %d\n", irq);
	return 0;
}

static const struct of_device_id ledbtn_of_match[] = {
	{ .compatible = "mycompany,ledbtn" },
	{ }
};
MODULE_DEVICE_TABLE(of, ledbtn_of_match);

static struct platform_driver ledbtn_driver = {
	.probe	= ledbtn_probe,
	.driver	= {
		.name		= "ledbtn",
		.of_match_table	= ledbtn_of_match,
		.dev_groups	= ledbtn_groups,
	},
};
module_platform_driver(ledbtn_driver);
```

- **One structure per device**, allocated in `probe()`, never a global: the same driver may run several devices. `platform_set_drvdata()` stores it in the device so that every other callback finds it with `dev_get_drvdata()` or `platform_get_drvdata()`. Store it before anything that can call back into the driver, such as the interrupt.
- **The match table ends with an empty entry.** It's a sentinel, not decoration.
- **`module_platform_driver()`** writes the module's init and exit functions, which only register and unregister the driver. Nothing else should happen in them; all the work is per device, in `probe()`.

**Managed resources: why there is no `remove()`.** Every `devm_*` call registers its undo action with the device. When `probe()` fails, or when the driver is unbound, the driver core runs those actions in reverse order (`devres_release_all()` in `drivers/base/devres.c`). For `ledbtn`, unbinding does:

1. remove the sysfs attributes (driver core, before anything else);
2. call `remove()`, if the driver has one;
3. free the interrupt, so the handler can no longer run;
4. cancel the blink work and wait for it (`devm_delayed_work_autocancel`);
5. release the two GPIOs, destroy the mutex, free the structure.

The order is the reverse of `probe()`, which is exactly the order in which it's safe to tear things down: the interrupt goes before the data it uses. This works as long as **everything** in `probe()` is managed. Mixing a non-managed resource into a managed `probe()` is the classic bug: if `remove()` frees something that a still-registered `devm` interrupt handler uses, the handler can run on freed memory between steps 2 and 3. When a resource has no `devm_` variant, wrap it with `devm_add_action_or_reset(dev, undo_fn, data)`, which keeps everything in the same ordered list.

| Common managed calls | Section |
|---|---|
| `devm_kzalloc()`, `devm_kcalloc()`, `devm_kstrdup()` | 18 |
| `devm_platform_ioremap_resource()` | 12 |
| `devm_clk_get_enabled()`, `devm_regulator_get_enable()`, `devm_reset_control_get_exclusive_deasserted()` | 12 |
| `devm_gpiod_get()`, `devm_request_threaded_irq()` | 13 |
| `devm_mutex_init()`, `devm_delayed_work_autocancel()`, `devm_add_action_or_reset()` | 14, 15 |
| `devm_regmap_init_i2c()`, `devm_hwmon_device_register_with_info()`, `devm_iio_device_register()`, `devm_led_classdev_register()` | 16, 17 |

`Documentation/driver-api/driver-model/devres.rst` lists them all.

**Loading it on the board** (**Expected**, from the format strings in the driver):

```
# insmod ledbtn.ko
ledbtn ledbtn: ready, button on IRQ 53
# ls /sys/bus/platform/drivers/ledbtn/
bind  ledbtn  module  uevent  unbind
```

`dev_info()` prints the driver name, then the device name (here both `ledbtn`), then the message. The IRQ number is assigned at run time; it means nothing outside this boot.

## 12. Reaching the hardware: registers, clocks, regulators and pins

`ledbtn` only uses GPIOs, which another driver owns. A driver for an SoC peripheral talks to its registers, and must turn on the clock, the power and the reset line that the block depends on. The device tree lists all of them in the node:

```dts
	myip@48300000 {
		compatible = "mycompany,myip";
		reg = <0x48300000 0x1000>;
		clocks = <&some_clock_provider 4>;
		clock-names = "fck";
		resets = <&some_reset_provider 2>;
		interrupts = <42>;
	};
```

(**Illustrative**: the addresses, the providers and the numbers are made up.) The driver gets each resource by name or index, all managed (`myip.c`, **Real**: compiled):

```c
#define MYIP_ID		0x00		/* read-only: version register */
#define MYIP_CTRL	0x04
#define MYIP_CTRL_EN	BIT(0)
#define MYIP_CTRL_DIV	GENMASK(11, 8)

static int myip_probe(struct platform_device *pdev)
{
	struct device *dev = &pdev->dev;
	struct reset_control *rst;
	struct myip *ip;
	u32 id, ctrl;
	int ret;

	ip = devm_kzalloc(dev, sizeof(*ip), GFP_KERNEL);
	if (!ip)
		return -ENOMEM;
	platform_set_drvdata(pdev, ip);

	ip->base = devm_platform_ioremap_resource(pdev, 0);
	if (IS_ERR(ip->base))
		return PTR_ERR(ip->base);

	ip->clk = devm_clk_get(dev, "fck");
	if (IS_ERR(ip->clk))
		return dev_err_probe(dev, PTR_ERR(ip->clk), "no functional clock\n");

	rst = devm_reset_control_get_optional_exclusive_deasserted(dev, NULL);
	if (IS_ERR(rst))
		return dev_err_probe(dev, PTR_ERR(rst), "cannot deassert reset\n");

	ret = clk_prepare_enable(ip->clk);
	if (ret)
		return ret;

	id = readl(ip->base + MYIP_ID);
	ctrl = readl(ip->base + MYIP_CTRL);
	ctrl &= ~MYIP_CTRL_DIV;
	ctrl |= FIELD_PREP(MYIP_CTRL_DIV, 4) | MYIP_CTRL_EN;
	writel(ctrl, ip->base + MYIP_CTRL);
	...
}
```

**Registers.** `devm_platform_ioremap_resource(pdev, 0)` takes the first `reg` entry, reserves the physical range (it then appears in `/proc/iomem` with the device's name, and a second driver can't claim it), and maps it into the kernel's virtual address space. The result is a `void __iomem *`: a cookie for the accessors, never dereferenced directly.

| Accessor | Use |
|---|---|
| `readl(addr)`, `writel(val, addr)` (and `readb`/`readw`, `writeb`/`writew`) | 32-bit (8, 16-bit) little-endian register access, ordered with respect to the CPU's other memory accesses and to DMA. The default |
| `readl_relaxed()`, `writel_relaxed()` | Same access without the barriers: faster in loops of register writes, but not ordered against DMA buffers |
| `readl_poll_timeout(addr, val, cond, sleep_us, timeout_us)` (`<linux/iopoll.h>`) | Poll a status bit until a condition holds, with a timeout. `_atomic` variant for contexts that can't sleep |
| `FIELD_PREP(mask, val)`, `FIELD_GET(mask, reg)`, `GENMASK(h, l)`, `BIT(n)` (`<linux/bitfield.h>`, `<linux/bits.h>`) | Build and extract register fields without shift constants |

A write to a register is posted: `writel()` may return before the device has seen it. When the next step depends on the write having landed (before a delay, before turning a clock off), read a register of the same device back.

**regmap** puts a uniform API (`regmap_read()`, `regmap_write()`, `regmap_update_bits()`, `regmap_field_*()`) on top of MMIO, I2C, SPI and other buses, with optional caching, locking, and a debugfs view of every register (`/sys/kernel/debug/regmap/<device>/registers`). For MMIO, `devm_regmap_init_mmio(dev, base, &config)`. It costs a few function calls per access, which only matters in very hot paths; for I2C and SPI devices, it's the default choice (section 16).

**Clocks.** A peripheral's registers often read as zeros or hang the bus when its clock is off; on the AM335x, an access to a module whose clock is gated gives an external abort. `devm_clk_get(dev, "fck")` gets the clock named in `clock-names`; `clk_prepare_enable()` turns it on, `clk_disable_unprepare()` off. `devm_clk_get_enabled()` gets and enables it, and disables it at unbind, which is all most drivers need. `clk_get_rate()` gives the frequency for divider calculations.

**Resets.** `devm_reset_control_get_exclusive_deasserted(dev, NULL)` takes the reset line out of reset and puts it back at unbind; the `_optional_` variant used in `myip` returns `NULL` instead of an error when the node has no `resets` property.

**Regulators.** External chips often need their supply turned on: `devm_regulator_get_enable(dev, "vdd")` for a `vdd-supply = <&reg>;` property. Without a property, the regulator core hands out a dummy regulator and warns `supply vdd not found, using dummy regulator` (**From source**: `drivers/regulator/core.c`), so the same driver works on boards where the supply is always on.

**Pins.** Applied automatically from `pinctrl-0` (section 11). A driver with several pin states, such as a `sleep` state, selects them with `pinctrl_pm_select_sleep_state()` or `pinctrl_lookup_state()` and `pinctrl_select_state()`.

**Runtime power management** turns a device's clocks and power off between uses. On the AM335x, the SoC's power domains are driven this way by the `ti-sysc` interconnect driver, which is why many drivers for this SoC call `pm_runtime_get_sync()` before touching registers. `myip` keeps its functional clock off when idle (**Real**: compiled):

```c
static int myip_runtime_suspend(struct device *dev)
{
	struct myip *ip = dev_get_drvdata(dev);

	clk_disable_unprepare(ip->clk);
	return 0;
}

static int myip_runtime_resume(struct device *dev)
{
	struct myip *ip = dev_get_drvdata(dev);

	return clk_prepare_enable(ip->clk);
}

static DEFINE_RUNTIME_DEV_PM_OPS(myip_pm_ops, myip_runtime_suspend,
				 myip_runtime_resume, NULL);

	/* at the end of probe(): the clock is on, the device is active */
	pm_runtime_set_active(dev);
	pm_runtime_set_autosuspend_delay(dev, 1000);
	pm_runtime_use_autosuspend(dev);
	ret = devm_pm_runtime_enable(dev);
	if (ret) {
		clk_disable_unprepare(ip->clk);
		return ret;
	}
	pm_runtime_put_autosuspend(dev);	/* idle in one second */
```

and `.driver.pm = pm_ptr(&myip_pm_ops)`. Every later access is bracketed by `pm_runtime_resume_and_get(dev)` and `pm_runtime_put_autosuspend(dev)`; the callbacks run when the usage count reaches zero, after the delay. Runtime PM is a subject of its own (`Documentation/power/runtime_pm.rst`). This skeleton doesn't balance the clock when it's unbound while active; a real driver resumes the device and disables the clock in its teardown.

## 13. GPIOs and interrupts

**GPIOs: descriptors, not numbers.** A driver gets each GPIO it uses from its device's description, as an opaque `struct gpio_desc *`:

| Call | Does |
|---|---|
| `devm_gpiod_get(dev, "led", GPIOD_OUT_LOW)` | Looks up `led-gpios` in the node, requests the line, sets it as an output at logical 0 |
| `devm_gpiod_get_optional(dev, "reset", GPIOD_OUT_HIGH)` | Same, but returns `NULL` (not an error) when the property is absent |
| `devm_gpiod_get_index(dev, "data", i, GPIOD_IN)`, `devm_gpiod_get_array()` | One line of a multi-line property, or all of them |
| `gpiod_set_value(desc, 1)`, `gpiod_get_value(desc)` | Set or read the **logical** value: 1 means active, whatever the `GPIO_ACTIVE_LOW` flag makes that on the wire |
| `gpiod_set_value_cansleep()`, `gpiod_get_value_cansleep()` | The same, allowed to sleep: needed for GPIOs on I2C or SPI expanders |
| `gpiod_to_irq(desc)` | The interrupt number of an input line, if its controller can interrupt |

**`_cansleep` or not.** The AM3358's own GPIO banks are memory-mapped and never sleep, but the same driver may one day get its LED from an I2C GPIO expander, where every access is an I2C transfer. Use the `_cansleep` variants in process context (as `ledbtn` does), and the plain ones only where sleeping isn't allowed, on a controller that doesn't sleep; `gpiod_cansleep(desc)` tells which. In 7.3 `gpiod_set_value()` returns an `int` (**From source**: `include/linux/gpio/consumer.h`), so a driver that cares can check for errors.

The old integer API (`gpio_request(60, ...)`, `gpio_set_value()`) appears in most tutorials. Don't use it: the global numbers depend on probe order and are no longer stable, and the polarity has to be handled in code. From userspace, the `gpioinfo`, `gpioget`, `gpioset` and `gpiomon` tools of libgpiod use the character device `/dev/gpiochipN`; `/sys/class/gpio` is the deprecated sysfs interface.

**Interrupts.** A device signals an event (a button press, data ready, a transfer done) on an interrupt line. The CPU stops what it's doing and runs the kernel's interrupt entry, which calls every handler registered for that line. The handler runs in **hard interrupt context**: interrupts on that line are masked, nothing may sleep, and every microsecond spent there is latency for the rest of the system.

**Two halves.** The kernel's answer is to split the work:

```
 button pressed
      │
      ▼
 hard IRQ handler (optional)    interrupt context, can't sleep
   read/acknowledge the status,     return IRQ_WAKE_THREAD
      │
      ▼
 threaded handler               a kernel thread "irq/53-ledbtn", may sleep:
   the real work: I2C reads,        mutexes, I2C, SPI, GPIO expanders
   locking, waking readers
```

`request_threaded_irq(irq, handler, thread_fn, flags, name, data)` registers both. The `ledbtn` driver passes no hard handler (`NULL`) and does everything in the thread (**Real**: compiled):

```c
#define LEDBTN_DEBOUNCE_MS	20

/* Threaded handler: runs in a kernel thread, so it may sleep. */
static irqreturn_t ledbtn_irq_thread(int irq, void *data)
{
	struct ledbtn *lb = data;
	ktime_t now = ktime_get();

	scoped_guard(mutex, &lb->lock) {
		if (ktime_ms_delta(now, lb->last_press) < LEDBTN_DEBOUNCE_MS)
			return IRQ_HANDLED;	/* contact bounce */
		lb->last_press = now;
		lb->presses++;
		if (!lb->blink_ms)
			ledbtn_set_led(lb, !lb->led_on);
	}
	sysfs_notify(&lb->dev->kobj, NULL, "presses");
	return IRQ_HANDLED;
}

	/* in probe() */
	ret = devm_request_threaded_irq(dev, irq, NULL, ledbtn_irq_thread,
					IRQF_TRIGGER_FALLING | IRQF_ONESHOT,
					dev_name(dev), lb);
```

- **`IRQF_ONESHOT`** keeps the line masked until the thread has finished. It's mandatory when there is no hard handler: without it, a level-triggered line would fire again immediately, and the kernel refuses the request with `Threaded irq requested with handler=NULL and !ONESHOT for ledbtn (irq 53)` (**From source**: `kernel/irq/manage.c`).
- **`IRQF_TRIGGER_FALLING`**: the button pulls the pin from 3.3 V to 0 V when pressed. Trigger flags describe the wire, not the logical value, so `GPIO_ACTIVE_LOW` doesn't change them. For a device whose node has an `interrupts` property, leave the trigger out of the flags; it comes from the device tree.
- **The last argument** (`lb`) is passed back to the handler as `data`. It's also the cookie that identifies this handler on a shared line.
- **Mechanical buttons bounce**: one press gives several edges within a few milliseconds. The handler ignores edges within 20 ms of the last accepted one. A GPIO controller that supports it can debounce in hardware instead: `gpiod_set_debounce(desc, 20000)`.

**When a hard handler is needed.** When the device must be acknowledged within microseconds, or the line is shared and the handler must check whether its device raised it:

```c
static irqreturn_t myip_irq(int irq, void *data)
{
	struct myip *ip = data;
	u32 status = readl(ip->base + MYIP_IRQ_STATUS);

	if (!status)
		return IRQ_NONE;		/* not ours: shared line */
	writel(status, ip->base + MYIP_IRQ_STATUS);	/* acknowledge */
	ip->pending |= status;
	return IRQ_WAKE_THREAD;			/* run the thread */
}
```

(**Illustrative**.) The handler returns `IRQ_HANDLED` when it dealt with the event, `IRQ_WAKE_THREAD` to run the threaded handler, `IRQ_NONE` when its device didn't interrupt. A line that keeps firing with every handler returning `IRQ_NONE` is eventually disabled with `irq 53: nobody cared (try booting with the "irqpoll" option)` and `Disabling IRQ #53` (**From source**: `kernel/irq/spurious.c`): usually a device whose interrupt wasn't acknowledged.

**Where the interrupt number comes from:** `gpiod_to_irq()` for a GPIO, `platform_get_irq(pdev, 0)` for the `interrupts` property of a platform device, `client->irq` for an I2C device and `spi->irq` for an SPI device. Never hard-code a number: they're allocated at boot.

**On the board**, the interrupt and its thread are visible (**Expected**; the GPIO bank's interrupt controller prints its device name, from `drivers/gpio/gpio-omap.c`):

```
# grep ledbtn /proc/interrupts
 53:          7  4804c000.gpio  16 Edge      ledbtn
# ps | grep irq/53
  245 root       0:00 [irq/53-ledbtn]
```

The count rises with every edge, bounces included.

## 14. Deferred work and time

Kernel code often has to do something later: blink an LED every 250 ms, poll a sensor every second, finish in process context what an interrupt started, wait 10 ms after a reset. The tools differ by the context the work runs in:

| Mechanism | Runs in | May sleep? | Resolution | Use it for |
|---|---|---|---|---|
| **Workqueue** (`struct work_struct`, `struct delayed_work`) | A kernel worker thread (process context) | Yes | Delayed work: jiffies | The default for "later", and for anything that sleeps |
| **Timer** (`struct timer_list`) | Softirq | No | Jiffies (10 ms at the BeagleBone's `HZ=100`) | Timeouts and watchdogs whose callback is short |
| **hrtimer** (`struct hrtimer`) | Hard interrupt | No | Nanoseconds (as good as the hardware timer) | Precise periodic events |
| **Kernel thread** (`kthread_run()`) | Its own thread | Yes | | A loop that waits for events for the driver's whole life |
| **Threaded IRQ** (section 13) | A thread per interrupt | Yes | | The second half of an interrupt |
| Tasklet, softirq | Softirq | No | | Legacy; new code uses threaded IRQs or workqueues |

**Workqueues.** `ledbtn` blinks the LED with a delayed work item that reschedules itself (**Real**: compiled):

```c
static void ledbtn_blink(struct work_struct *work)
{
	struct ledbtn *lb = container_of(to_delayed_work(work),
					 struct ledbtn, blink_work);

	guard(mutex)(&lb->lock);
	if (!lb->blink_ms)
		return;				/* blinking was switched off */
	ledbtn_set_led(lb, !lb->led_on);
	schedule_delayed_work(&lb->blink_work, msecs_to_jiffies(lb->blink_ms));
}

static ssize_t blink_ms_store(struct device *dev, struct device_attribute *attr,
			      const char *buf, size_t count)
{
	struct ledbtn *lb = dev_get_drvdata(dev);
	unsigned int ms;
	int ret;

	ret = kstrtouint(buf, 0, &ms);
	if (ret)
		return ret;
	if (ms && (ms < 10 || ms > 10000))
		return -EINVAL;

	guard(mutex)(&lb->lock);
	lb->blink_ms = ms;
	if (ms)
		schedule_delayed_work(&lb->blink_work, 0);
	return count;
}
```

- **`INIT_WORK()`/`INIT_DELAYED_WORK()`** prepare the item; `devm_delayed_work_autocancel()` does it and cancels the work at unbind (**From source**: `include/linux/devm-helpers.h`).
- **`schedule_work()`** and **`schedule_delayed_work(dw, delay)`** put the item on the system workqueue; an item already pending isn't queued twice. `mod_delayed_work()` changes the delay of a pending item.
- **Stopping**: writing 0 to `blink_ms` makes the next run return without rescheduling. `cancel_delayed_work_sync()` cancels and waits for a running instance to finish; never call it while holding a lock the work function takes, or the two wait for each other forever. That's why `blink_ms_store()` stops the work through the flag, not by cancelling it under the mutex.
- **Your own workqueue**, `alloc_workqueue("name", flags, 0)`, isolates work that is slow or must not be delayed by others. `WQ_UNBOUND` lets the scheduler run it on any CPU, `WQ_HIGHPRI` gives its workers a higher priority, `WQ_MEM_RECLAIM` guarantees progress under memory pressure (needed on the path to storage or the network).

**Timers, hrtimers, kthreads and completions**, side by side in one module (`deferred.c`, **Real**: compiled):

```c
struct demo {
	struct timer_list timer;	/* jiffies resolution, softirq context */
	struct hrtimer hrtimer;		/* ns resolution, hardirq context */
	struct work_struct work;	/* process context, may sleep */
	struct workqueue_struct *wq;	/* our own queue */
	struct task_struct *thread;
	struct completion done;
	spinlock_t lock;		/* timer and hrtimer callbacks vs. others */
	unsigned long ticks;
	unsigned long hr_ticks;
};

static struct demo demo;

static void demo_timer_fn(struct timer_list *t)
{
	struct demo *d = timer_container_of(d, t, timer);

	spin_lock(&d->lock);
	d->ticks++;
	spin_unlock(&d->lock);

	queue_work(d->wq, &d->work);		/* the sleeping part */
	mod_timer(&d->timer, jiffies + HZ);	/* again in one second */
}

static enum hrtimer_restart demo_hrtimer_fn(struct hrtimer *t)
{
	struct demo *d = container_of(t, struct demo, hrtimer);

	spin_lock(&d->lock);
	d->hr_ticks++;
	spin_unlock(&d->lock);

	hrtimer_forward_now(t, ms_to_ktime(100));
	return HRTIMER_RESTART;
}

static void demo_work_fn(struct work_struct *work)
{
	struct demo *d = container_of(work, struct demo, work);
	unsigned long ticks, hr_ticks;

	spin_lock_irq(&d->lock);	/* hrtimer runs in hardirq context */
	ticks = d->ticks;
	hr_ticks = d->hr_ticks;
	spin_unlock_irq(&d->lock);

	usleep_range(10000, 11000);	/* sleeping: fine here, never in a timer */
	pr_info("work: %lu timer ticks, %lu hrtimer ticks\n", ticks, hr_ticks);
}

static int demo_thread_fn(void *data)
{
	struct demo *d = data;

	complete(&d->done);		/* tell init we're running */
	while (!kthread_should_stop()) {
		/* A real thread would wait for an event here. */
		schedule_timeout_interruptible(5 * HZ);
	}
	return 0;
}

static int __init deferred_init(void)
{
	spin_lock_init(&demo.lock);
	init_completion(&demo.done);
	INIT_WORK(&demo.work, demo_work_fn);

	demo.wq = alloc_workqueue("deferred", WQ_UNBOUND, 0);
	if (!demo.wq)
		return -ENOMEM;

	demo.thread = kthread_run(demo_thread_fn, &demo, "deferred-thread");
	if (IS_ERR(demo.thread)) {
		destroy_workqueue(demo.wq);
		return PTR_ERR(demo.thread);
	}
	wait_for_completion(&demo.done);

	timer_setup(&demo.timer, demo_timer_fn, 0);
	mod_timer(&demo.timer, jiffies + HZ);

	hrtimer_setup(&demo.hrtimer, demo_hrtimer_fn, CLOCK_MONOTONIC,
		      HRTIMER_MODE_REL);
	hrtimer_start(&demo.hrtimer, ms_to_ktime(100), HRTIMER_MODE_REL);
	return 0;
}

static void __exit deferred_exit(void)
{
	/* Stop the sources first, then what they feed. */
	hrtimer_cancel(&demo.hrtimer);
	timer_shutdown_sync(&demo.timer);
	cancel_work_sync(&demo.work);
	destroy_workqueue(demo.wq);
	kthread_stop(demo.thread);
}
```

What it shows:

- **The timer callback can't sleep**, so it counts and hands the rest to a work item. That split, a short callback and a work item for the slow part, is the normal shape.
- **`timer_container_of()`** (formerly `from_timer()`) goes from the `timer_list` back to the structure; hrtimers use plain `container_of()`.
- **`hrtimer_forward_now()` and `HRTIMER_RESTART`** make a periodic hrtimer without drift: the next expiry is computed from the previous one, not from when the callback ran.
- **Teardown order matters**: stop what generates work (the timers), then cancel the work, then destroy its queue, then stop the thread. `timer_shutdown_sync()` waits for a running callback and prevents re-arming, which a self-rearming timer needs before its memory is freed.
- **`kthread_should_stop()`** is how a kernel thread learns that `kthread_stop()` was called; `kthread_stop()` wakes the thread and waits for it to return.
- **A completion** is a one-shot "it happened" signal between two contexts: `wait_for_completion()` sleeps until another context calls `complete()`. Drivers use it to wait for an interrupt that signals the end of a transfer, with a timeout: `wait_for_completion_timeout(&done, msecs_to_jiffies(100))` returns 0 on timeout.

**Delays.** `Documentation/timers/delay_sleep_functions.rst` in 7.3 gives the rules:

| Context | Call | Notes |
|---|---|---|
| May sleep | `fsleep(us)` | Picks the right mechanism for the duration; the safe default |
| May sleep, ≥ 20 ms | `msleep(ms)`, `msleep_interruptible(ms)` | Jiffy-based: `msleep(1)` sleeps up to two jiffies, 20 ms at `HZ=100`. `checkpatch` warns about `msleep()` below 20 ms |
| May sleep, 10 µs to 20 ms | `usleep_range(min, max)` | hrtimer-based; the range lets the kernel merge wake-ups |
| Can't sleep | `udelay(us)`, `ndelay(ns)` | Busy-waits: burns the CPU. Keep it to a few microseconds |

`jiffies` counts timer ticks since boot; compare them with `time_after(a, b)`, never with `>`, because the counter wraps. For timestamps and durations, `ktime_get()` gives a monotonic time in nanoseconds (`ledbtn` uses it for debouncing).
