# Cross-Compilation and Toolchains for Embedded Linux

A beginner's path from a PC to a C program running on an Arm board: what a cross toolchain is, how to build a small program with it, how to test it in QEMU, and how to run it on a Raspberry Pi 4 or a BeagleBone Black.

**You need:** a Debian or Ubuntu PC, and basic shell and C. No board is needed until section 7.

**How to read it:**
- **Parts 1 and 2 are essential.** Read them in order and type the commands.
- **Part 3 is optional.** Each section says when you need it. If something fails, start at [section 10](#10-troubleshooting).

**How commands are labelled:**

| Label | Meaning |
|---|---|
| **Tested (QEMU)** | Run in November 2026 on Ubuntu 22.04 (GCC 11.4, glibc 2.35, QEMU 6.2, CMake 3.22, Meson 0.61, pkg-config 0.29). The output is pasted from that run, with paths shortened. Other versions print slightly different text |
| **Expected (board)** | **Not run on real hardware.** The commands are what you should type, and the output is what you should see, based on the board's documentation and the test results under QEMU |
| **Illustrative** | Shows the form of a command or message. Not run for this note |

## Table of contents

**Part 1: Concepts (essential)**

- [1. What cross-compiling is](#1-what-cross-compiling-is)
- [2. The words you need](#2-the-words-you-need)
- [3. The boards, and why their software matters](#3-the-boards-and-why-their-software-matters)

**Part 2: A working example (essential)**

- [4. Install and check the toolchain](#4-install-and-check-the-toolchain)
- [5. Build your first program](#5-build-your-first-program)
- [6. Run it in QEMU](#6-run-it-in-qemu)
- [7. Run it on a board](#7-run-it-on-a-board)
- [8. Add a shared library](#8-add-a-shared-library)
- [9. Use Make or CMake](#9-use-make-or-cmake)

**Part 3: Going further (optional)**

- [10. Troubleshooting](#10-troubleshooting)
- [11. Matching the target](#11-matching-the-target)
- [12. Sysroots and leaks from the PC](#12-sysroots-and-leaks-from-the-pc)
- [13. pkg-config](#13-pkg-config)
- [14. Meson and Autotools](#14-meson-and-autotools)
- [15. Choosing a toolchain](#15-choosing-a-toolchain)
- [16. Debugging with GDB](#16-debugging-with-gdb)
- [17. Where the rest is covered](#17-where-the-rest-is-covered)
- [References](#references)

---

## 1. What cross-compiling is

Normally you compile a program on the machine that will run it. An embedded board is often too small or too slow for that, so you compile on your PC and copy the result to the board. That is **cross-compiling**.

The difficulty is that the result can't run on the machine that built it, and the PC's own files are the wrong ones for the board. For a program to run on the board, three things must match:

| What must match | In plain words | If it doesn't |
|---|---|---|
| **The CPU** | The board's processor must understand the instructions | `Exec format error` or `Illegal instruction` |
| **The ABI** | The agreed way compiled code is laid out and calls other code. The word comes back in section 2 | The program won't start |
| **The libraries** | The libraries the program needs must exist on the board, in a compatible version | `error while loading shared libraries`, or ``version `GLIBC_2.34' not found`` |

Most problems in cross-compiling are one of these three, or a file from the PC that ended up in the board's build by accident.

---

## 2. The words you need

| Word | Meaning |
|---|---|
| **Build machine** | The PC where the compiler runs. Here, x86_64 |
| **Target** | The machine that will run the program. Here, an Arm board |
| **Toolchain** | The set of tools and files that turn source into a target program: compiler, assembler, linker, and the target's C library |
| **Cross compiler** | A compiler that runs on the build machine and produces code for the target |
| **Architecture** | The CPU family. Here, `arm` (32-bit) or `aarch64` (64-bit) |
| **C library** | The library almost every program uses for basics like `printf` and file access. On these boards it is **glibc** |
| **ABI** (application binary interface) | The binary rules that let compiled pieces work together: how arguments are passed, the size of types, and how a program is started |
| **Userspace** | Everything on a Linux system above the kernel: programs, libraries, including the C library |

A toolchain is named by a **triplet**, and its commands start with that name:

```
arm-linux-gnueabihf-gcc                 aarch64-linux-gnu-gcc
│    │     │                            │       │     │
│    │     └─ glibc, Arm ABI,           │       │     └─ glibc
│    │        hard floating point       │       └─────── runs on Linux
│    └─────── runs on Linux             └─────────────── 64-bit Arm
└──────────── 32-bit Arm
```

**A warning about the word "host".** Tools don't agree on it. In Autotools and Meson the "host" is the machine that runs the program (the board). In CMake and the Linux kernel, the "host" is the PC. This note says **build machine** and **target** to avoid the confusion.

---

## 3. The boards, and why their software matters

**Why does a toolchain care about the operating system on the board?** Because a compiled program isn't complete on its own. It uses two C libraries, at two different moments:

```
On the PC, when building                 On the board, when running
─────────────────────────                ──────────────────────────
hello.c                                  hello
  + the toolchain's headers                + the board's libc.so.6
  + the toolchain's C library              + the board's loader
        │                                        │
        └── produces ──► hello ── copied ────────┘
```

The toolchain's C library has to be compatible with the one installed on the board. So before choosing a toolchain, you need to know what the board runs. **This is what "userspace assumptions" means here: which CPU and which C library the board has.**

This note uses two boards. The same commands work for both, because the differences are kept in one small settings file (section 4).

| | BeagleBone Black | Raspberry Pi 4 |
|---|---|---|
| Processor | TI AM3358, Cortex-A8 | Broadcom BCM2711, Cortex-A72 |
| Architecture used | 32-bit Arm (ARMv7-A) | 64-bit Arm (AArch64) |
| Triplet | `arm-linux-gnueabihf` | `aarch64-linux-gnu` |
| Operating system | [BeagleBoard Debian 12 IoT image](https://www.beagleboard.org/distros) for AM335x | [Raspberry Pi OS Lite 64-bit](https://www.raspberrypi.com/software/operating-systems/), based on Debian 13 |
| C library | glibc 2.36 | glibc 2.41 |
| QEMU CPU model | `cortex-a8` | `cortex-a72` |

**QEMU** is the third way to run the program. It emulates an Arm CPU on your PC, so you can test with no board at all (section 6).

**Your PC matters too.** Debian and Ubuntu's cross compilers come with the C library of that release, and it must not be newer than the board's:

| PC | Toolchain's glibc | BeagleBone Black (2.36) | Raspberry Pi OS (2.41) |
|---|---|---|---|
| Ubuntu 22.04 | 2.35 | Yes | Yes |
| Debian 12 | 2.36 | Yes | Yes |
| Ubuntu 24.04 | 2.39 | **No** | Yes |
| Debian 13 | 2.41 | **No** | Yes |

If your row says No, build inside a Debian 12 container, for example with `docker run --rm -it -v "$HOME/cross:/root/cross" debian:12` (**Illustrative**). Section 11 explains this rule.

---

## 4. Install and check the toolchain

Install the cross compilers, QEMU and the build tools:

```sh
sudo apt install gcc-arm-linux-gnueabihf gcc-aarch64-linux-gnu \
    qemu-user-static make cmake
mkdir -p ~/cross/hello ~/cross/project/lib ~/cross/project/app
```

Create one settings file per board. These are the only places where the two boards differ.

`~/cross/target-bbb.sh`:

```sh
export CROSS_COMPILE=arm-linux-gnueabihf-
export QEMU="qemu-arm-static -cpu cortex-a8 -L /usr/arm-linux-gnueabihf"
export STAGING=$HOME/cross/staging-bbb
export BOARD=debian@192.168.6.2
```

`~/cross/target-pi4.sh`:

```sh
export CROSS_COMPILE=aarch64-linux-gnu-
export QEMU="qemu-aarch64-static -cpu cortex-a72 -L /usr/aarch64-linux-gnu"
export STAGING=$HOME/cross/staging-pi4
export BOARD=yourname@raspberrypi.local
```

Load the one for your board. Do this in every new terminal. If you have no board, either one works:

```sh
. ~/cross/target-bbb.sh
```

If your distribution installs `qemu-arm` and `qemu-aarch64` instead, without `-static`, change those names here.

`CROSS_COMPILE` holds the start of the tool names, so `${CROSS_COMPILE}gcc` becomes the right compiler. The Linux kernel uses the same variable name.

Check the compiler, and find which glibc it builds against.

**Tested (QEMU)**, BeagleBone Black settings:

```console
$ ${CROSS_COMPILE}gcc --version | head -1
arm-linux-gnueabihf-gcc (Ubuntu 11.4.0-1ubuntu1~22.04.3) 11.4.0
$ ${CROSS_COMPILE}gcc -dumpmachine
arm-linux-gnueabihf
$ ${CROSS_COMPILE}gcc -dM -E -include features.h -x c /dev/null | grep -E '__GLIBC(_MINOR)?__ '
#define __GLIBC__ 2
#define __GLIBC_MINOR__ 35
```

The last command says this toolchain uses glibc 2.35.

---

## 5. Build your first program

`~/cross/hello/hello.c` prints the version of the C library it is running with:

```c
#include <stdio.h>
#include <gnu/libc-version.h>

int main(void)
{
    printf("hello from glibc %s\n", gnu_get_libc_version());
    return 0;
}
```

Compile it the way you would with `gcc`, using the cross compiler. It won't run on the PC, and that's expected.

**Tested (QEMU)**, BeagleBone Black settings:

```console
$ cd ~/cross/hello
$ ${CROSS_COMPILE}gcc -Wall -O2 hello.c -o hello
$ ./hello
./hello: Exec format error
```

Your PC's CPU can't execute Arm code. (If `qemu-user-static` set up automatic emulation, you get a different error about a missing `ld-linux` file. Section 6 fixes that one.)

**Always check what you built.** `file` and `readelf` read a program without running it:

```console
$ file hello
hello: ELF 32-bit LSB pie executable, ARM, EABI5 version 1 (SYSV), dynamically linked,
interpreter /lib/ld-linux-armhf.so.3, for GNU/Linux 3.2.0, not stripped
$ readelf -d hello | grep NEEDED
 0x00000001 (NEEDED)                     Shared library: [libc.so.6]
```

With the Raspberry Pi settings, `file` says `ELF 64-bit LSB pie executable, ARM aarch64` and the interpreter is `/lib/ld-linux-aarch64.so.1`.

| What the output says | What it means |
|---|---|
| `ARM`, or `ARM aarch64` | The architecture the program is built for. **Check this first** |
| `interpreter /lib/ld-linux-armhf.so.3` | The **loader**, the small program that starts this one and connects its libraries. It must exist at exactly this path on the board |
| `NEEDED libc.so.6` | A library that must be found when the program starts |

---

## 6. Run it in QEMU

QEMU's "user mode" runs one Arm program on your PC. `$QEMU` from your settings file does it.

**Tested (QEMU)**, same result with either settings file:

```console
$ $QEMU ./hello
hello from glibc 2.35
```

The two options in `$QEMU` matter:

| Option | Why |
|---|---|
| `-L /usr/arm-linux-gnueabihf` | The program needs an Arm loader and C library, and those on your PC are x86. This folder holds Arm copies, installed with the cross compiler. Without `-L`: `Could not open '/lib/ld-linux-armhf.so.3'` |
| `-cpu cortex-a8` | Emulate exactly the board's CPU. Without it, QEMU enables every feature it knows and hides mistakes (section 11) |

A folder that looks like the target's filesystem and holds its headers and libraries is called a **sysroot**. `/usr/arm-linux-gnueabihf` is a small one. Section 12 has more.

**What QEMU proves, and what it doesn't.** QEMU shows that the program is correctly built for the CPU and starts. It doesn't test your real kernel, your devices, your memory limits or timing. For those you need the board.

---

## 7. Run it on a board

Everything here is **Expected (board)**: not run on hardware for this note. The version numbers come from each distribution's packages, and the patch numbers on your board may differ.

### Connect to the board

**BeagleBone Black:**

1. Write the [Debian 12 IoT microSD image for AM335x](https://www.beagleboard.org/distros) to a card.
2. Insert the card. Hold the S2 button while powering on, so the board boots from the card instead of its internal eMMC.
3. Connect the board to your PC by USB. It appears as a network adapter.
4. Log in. The default password is `temppwd`.

```sh
ssh debian@192.168.6.2
```

The [board's address over USB](https://docs.beagleboard.org/intro/support/getting-started.html) is `192.168.6.2` or `192.168.7.2`, depending on the USB network adapter your PC uses. BeagleBoard's table lists `.6.2` for Linux. If the other one is yours, change `BOARD` in `target-bbb.sh`.

**Raspberry Pi 4:**

1. In [Raspberry Pi Imager](https://www.raspberrypi.com/software/), choose **Raspberry Pi OS Lite (64-bit)**.
2. In the OS customisation settings, set the hostname to `raspberrypi`, choose a username and password, and enable SSH.
3. Boot the Pi on your network with Ethernet.
4. Put your username in `BOARD` in `target-pi4.sh`, then log in:

```sh
ssh yourname@raspberrypi.local
```

### Check what the board runs

Run this on the board, and compare it with the table in section 3:

```console
$ uname -m
armv7l                                   # BeagleBone Black
aarch64                                  # Raspberry Pi 4, 64-bit OS
$ ldd --version | head -1
ldd (Debian GLIBC 2.36-9+deb12u...) 2.36  # BeagleBone Black
ldd (Debian GLIBC 2.41-...) 2.41          # Raspberry Pi OS
```

The board's glibc must be the same as or newer than the toolchain's version from section 4.

### Copy and run

From the PC, with your settings file loaded:

```console
$ cd ~/cross/hello
$ scp hello $BOARD:
$ ssh $BOARD ./hello
hello from glibc 2.36                    # BeagleBone Black
hello from glibc 2.41                    # Raspberry Pi OS
```

In QEMU the program said 2.35, and on the board it says the board's version. **It's the same file.** The C library is loaded from wherever the program runs, which is the point of section 3.

---

## 8. Add a shared library

Real programs use libraries. This small project has one library, `libchk`, and one program, `crcsum`.

`~/cross/project/lib/chk.h`:

```c
#ifndef CHK_H
#define CHK_H
#include <stddef.h>
#include <stdint.h>
uint32_t chk_crc32(const void *data, size_t len);
const char *chk_version(void);
#endif
```

`~/cross/project/lib/chk.c`:

```c
#include "chk.h"

uint32_t chk_crc32(const void *data, size_t len)
{
    const uint8_t *p = data;
    uint32_t crc = 0xffffffffu;

    while (len--) {
        crc ^= *p++;
        for (int i = 0; i < 8; i++)
            crc = (crc >> 1) ^ (0xedb88320u & -(crc & 1));
    }
    return ~crc;
}

const char *chk_version(void)
{
    return "1.0.0";
}
```

`~/cross/project/app/crcsum.c`:

```c
#include <stdio.h>
#include <chk.h>

int main(void)
{
    static char buf[65536];
    size_t n = fread(buf, 1, sizeof(buf), stdin);

    printf("%08x  (libchk %s)\n", chk_crc32(buf, n), chk_version());
    return 0;
}
```

Build the library, then the program that uses it.

**Tested (QEMU)**, BeagleBone Black settings:

```console
$ cd ~/cross/project
$ mkdir -p out
$ ${CROSS_COMPILE}gcc -O2 -Wall -fPIC -shared -Wl,-soname,libchk.so.1 lib/chk.c -o out/libchk.so.1
$ ${CROSS_COMPILE}gcc -O2 -Wall -Ilib app/crcsum.c out/libchk.so.1 -o out/crcsum
$ readelf -d out/crcsum | grep NEEDED
 0x00000001 (NEEDED)                     Shared library: [libchk.so.1]
 0x00000001 (NEEDED)                     Shared library: [libc.so.6]
```

`-Wl,-soname,libchk.so.1` gives the library its official name, the **soname**. Programs that link to it record that name in `NEEDED`, and that's the file name the loader will look for on the target.

Running it needs the library to be found:

```console
$ echo -n 123456789 | $QEMU out/crcsum
out/crcsum: error while loading shared libraries: libchk.so.1: cannot open shared object file: No such file or directory

$ echo -n 123456789 | $QEMU -E LD_LIBRARY_PATH=out out/crcsum
cbf43926  (libchk 1.0.0)
```

`cbf43926` is the standard check value of CRC-32 for that input, so the result is correct. `-E` sets a variable for the emulated program.

**On the board** (**Expected (board)**):

```console
$ scp out/libchk.so.1 out/crcsum $BOARD:
$ echo -n 123456789 | ssh $BOARD 'LD_LIBRARY_PATH=$HOME ./crcsum'
cbf43926  (libchk 1.0.0)
```

The single quotes make `$HOME` expand on the board. To install the library permanently: `sudo cp libchk.so.1 /usr/local/lib/ && sudo ldconfig`. The program then runs without `LD_LIBRARY_PATH`, at least on systems where `/usr/local/lib` is on the loader's path, as on Debian.

---

## 9. Use Make or CMake

Typing those commands doesn't scale. Either tool below builds the same project, for the board chosen by your settings file.

### Make

`~/cross/project/Makefile`:

```make
CC     := $(CROSS_COMPILE)gcc
CFLAGS := -O2 -Wall
OUT    := build/$(CROSS_COMPILE:-=)

all: $(OUT)/crcsum

$(OUT)/libchk.so.1: lib/chk.c lib/chk.h
	mkdir -p $(OUT)
	$(CC) $(CFLAGS) -fPIC -shared -Wl,-soname,libchk.so.1 $< -o $@

$(OUT)/crcsum: app/crcsum.c $(OUT)/libchk.so.1
	$(CC) $(CFLAGS) -Ilib $^ -o $@

clean:
	rm -rf build

.PHONY: all clean
```

Two details do the cross-compiling work. The compiler comes from `CROSS_COMPILE`, and each target gets its own output folder (`build/arm-linux-gnueabihf`), so files for one board can't get mixed with the other's.

**Tested (QEMU)**, BeagleBone Black settings:

```console
$ make
mkdir -p build/arm-linux-gnueabihf
arm-linux-gnueabihf-gcc -O2 -Wall -fPIC -shared -Wl,-soname,libchk.so.1 lib/chk.c -o build/arm-linux-gnueabihf/libchk.so.1
arm-linux-gnueabihf-gcc -O2 -Wall -Ilib app/crcsum.c build/arm-linux-gnueabihf/libchk.so.1 -o build/arm-linux-gnueabihf/crcsum
$ T=build/${CROSS_COMPILE%-}
$ echo -n 123456789 | $QEMU -E LD_LIBRARY_PATH=$T $T/crcsum
cbf43926  (libchk 1.0.0)
```

### CMake

`~/cross/project/CMakeLists.txt` says nothing about cross-compiling:

```cmake
cmake_minimum_required(VERSION 3.16)
project(crcsum C)

add_library(chk SHARED lib/chk.c)
set_target_properties(chk PROPERTIES VERSION 1.0.0 SOVERSION 1)
target_include_directories(chk PUBLIC lib)

add_executable(crcsum app/crcsum.c)
target_link_libraries(crcsum PRIVATE chk)

install(TARGETS chk crcsum)
```

The target is described in a separate **toolchain file**. `~/cross/bbb.cmake`:

```cmake
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR arm)
set(CMAKE_C_COMPILER arm-linux-gnueabihf-gcc)

# Search for libraries and headers only in the staging folder, never on the PC
set(CMAKE_FIND_ROOT_PATH "$ENV{STAGING}")
set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
```

For the Raspberry Pi, create `~/cross/pi4.cmake` with `aarch64` as the processor and `aarch64-linux-gnu-gcc` as the compiler.

The **staging folder** (`$STAGING`) is where we install what we built, laid out like the board's filesystem. `DESTDIR` sends the files there, and `--prefix /usr` is where they will live on the board.

**Tested (QEMU)**, BeagleBone Black settings:

```console
$ cmake -S . -B build-cmake-bbb -DCMAKE_TOOLCHAIN_FILE=$HOME/cross/bbb.cmake
-- The C compiler identification is GNU 11.4.0
-- Check for working C compiler: /usr/bin/arm-linux-gnueabihf-gcc - skipped
$ cmake --build build-cmake-bbb
[100%] Built target crcsum
$ DESTDIR=$STAGING cmake --install build-cmake-bbb --prefix /usr
-- Installing: /home/me/cross/staging-bbb/usr/bin/crcsum
-- Set runtime path of ".../usr/bin/crcsum" to ""
```

CMake reads the toolchain file only the first time in a build folder. To change target, use a new folder.

**Deploy from `$STAGING`, not from the build folder.** A built program contains a path to the build folder on your PC so that it can run there, and installing removes it: that's the `Set runtime path` line.

**On the board** (**Expected (board)**): `scp` follows the symlink, so the board gets a file named `libchk.so.1`.

```console
$ scp $STAGING/usr/bin/crcsum $STAGING/usr/lib/libchk.so.1 $BOARD:
$ echo -n 123456789 | ssh $BOARD 'LD_LIBRARY_PATH=$HOME ./crcsum'
cbf43926  (libchk 1.0.0)
```

**This is the end of the essential path.** You can now build a project with a library, test it in QEMU, and run it on a board.

---

## 10. Troubleshooting

**Read this when something fails.** First ask: did it fail on the PC or on the board? Then run `file` on the program.

The last column says whether the error was reproduced for this note.

| Symptom | Likely cause | What to do | Reproduced |
|---|---|---|---|
| `Exec format error` on the PC | You ran a target program directly. Normal | Use `$QEMU` | Yes |
| `Exec format error` on the board | Built for the wrong architecture, or by the PC's `gcc` | `file` on it. Is it `ARM`? | On PC only |
| `Could not open '/lib/ld-linux-...'` from QEMU | QEMU can't find the target's loader | Add `-L` | Yes |
| `No such file or directory` for a file that exists | The **loader** is missing on the board: built for another ABI or C library | `readelf -l` shows the path. Compare it with `/lib` on the board | Yes |
| `error while loading shared libraries: libX.so.1` | The library isn't on the board, or isn't in the search path | `readelf -d`, `LD_LIBRARY_PATH`, `ldconfig` | Yes |
| ``version `GLIBC_2.x' not found`` | Toolchain's glibc is newer than the board's | Section 11 | No |
| `Illegal instruction` | Program uses instructions this CPU doesn't have | Section 11. Test with the board's `-cpu` | Yes |
| `fatal error: foo.h: No such file or directory` | The library's development files aren't where the compiler looks | Section 12 | Yes |
| `cannot find -lfoo` | Library not found at link time | Check the `-L` flags | Yes |
| `skipping incompatible .../libfoo.so` | **Found the library, but it's for the wrong architecture**, often the PC's | `file` on it | Yes |
| `file in wrong format`, or `Relocations in generic ELF` | An object file was built by a different compiler. `EM: 62` means x86_64 | Delete build files and rebuild | Yes |
| `fatal error: gnu/stubs-soft.h` | Flags ask for an ABI the toolchain has no C library for | Section 11 | Yes |
| `cannot run test program while cross compiling` | A `configure` script wants to run target code | Section 14 | Yes |
| `Exec format error` in the middle of `make` | The project built a helper tool with the cross compiler, but it must run on the PC | Give the project a compiler for the build machine, usually `HOSTCC` or `CC_FOR_BUILD` | No |
| Works in QEMU, fails on the board | Real kernel, real devices, less memory, or QEMU's CPU was too generous | Test with `-cpu`, then debug on the board | Not applicable |

---

## 11. Matching the target

**Read this when** a program builds but fails on the board, or you pick a toolchain for a real product.

### C library version

glibc labels each function with the version that introduced it. Programs built against a newer glibc may need a label an older glibc doesn't have. Compatibility goes one way:

- Built against an **older** glibc, the program runs on a newer one.
- Built against a **newer** glibc, the program may refuse to start on an older one (**Illustrative**):

  ```
  ./app: /lib/arm-linux-gnueabihf/libc.so.6: version `GLIBC_2.38' not found (required by ./app)
  ```

Find what a program requires. Use the `objdump` of the toolchain that built it, because each one reads only its own architecture.

**Tested (QEMU)**, Raspberry Pi settings:

```console
$ ${CROSS_COMPILE}objdump -T hello | grep -o 'GLIBC_[0-9.]*' | sort -Vu | tail -1
GLIBC_2.34
```

Every dynamically linked program built against glibc 2.34 or newer requires at least 2.34, because the program's startup code uses a function versioned 2.34.

**The rule: build against a C library no newer than the one on the target.** The same applies to `libstdc++` for C++ (``version `GLIBCXX_3.4.x'``).

`-static` puts the C library inside the program and avoids this, at the cost of size, but glibc features such as name lookups still need shared libraries at run time. Prefer the matching toolchain.

### CPU flags

By default, the Debian and Ubuntu toolchains produce code for a baseline CPU, which runs on both boards. Flags can raise that baseline:

| Flag | Meaning |
|---|---|
| `-mcpu=` | Build for this CPU: what it may use, and what it's tuned for |
| `-march=` | Build for this instruction set |
| `-mtune=` | Tune for this CPU, without changing what it needs |

Typical values: `-mcpu=cortex-a8 -mfpu=neon -mfloat-abi=hard` on the BeagleBone Black, and `-mcpu=cortex-a72` on the Raspberry Pi 4.

The classic mistake is building for a newer CPU than the board's. Here `div.c` divides two numbers from the command line, built for a Cortex-A7, which has a hardware divide instruction that the BeagleBone's Cortex-A8 doesn't have.

**Tested (QEMU)**, BeagleBone Black settings:

```console
$ ${CROSS_COMPILE}gcc -O2 -mcpu=cortex-a7 -mfpu=neon-vfpv4 -mfloat-abi=hard div.c -o div-a7
$ ${CROSS_COMPILE}objdump -d div-a7 | grep -m1 -w sdiv
 45e:	fb94 f2f2 	sdiv	r2, r4, r2

$ $QEMU ./div-a7 7 2
qemu: uncaught target signal 4 (Illegal instruction) - core dumped
Illegal instruction (core dumped)

$ qemu-arm-static -L /usr/arm-linux-gnueabihf ./div-a7 7 2
3
```

The last command has no `-cpu`, and it runs. **That's why `$QEMU` sets `-cpu`:** QEMU's default CPU would have hidden this bug. Remember that `-march=native` means the PC's CPU, so remove it from any build for a board.

### Differences that compile silently

| Difference | PC (x86_64) | Pi 4 (AArch64) | BBB (32-bit Arm) | Risk |
|---|---|---|---|---|
| Size of `long` and pointers | 8 bytes | 8 bytes | 4 bytes | File formats, network messages |
| Plain `char` | Signed | **Unsigned** | **Unsigned** | `char c = getchar(); if (c == EOF)` is never true |

Use types like `uint32_t` for anything stored or sent.

### 32-bit Arm only: float ABI and 2038

**Float ABI.** `gnueabihf` means floating-point values are passed in FPU registers (`hard`). All code in a program must agree. Asking for another ABI fails with `gnu/stubs-soft.h: No such file or directory`, or at link time with `uses VFP register arguments, X does not`.

**64-bit time.** On 32-bit glibc, `time_t` may be 32 bits, and a 32-bit time overflows in January 2038. It depends on the distribution: [Debian 13 moved its 32-bit Arm packages to 64-bit `time_t`](https://www.debian.org/releases/trixie/release-notes/whats-new.html), while the Debian 12 on the BeagleBone Black doesn't. With Ubuntu 22.04's toolchain, a small program `t.c` that prints both sizes gives:

```console
$ arm-linux-gnueabihf-gcc t.c -o t32 && $QEMU ./t32
time_t=4 off_t=4
$ arm-linux-gnueabihf-gcc -D_FILE_OFFSET_BITS=64 -D_TIME_BITS=64 t.c -o t64 && $QEMU ./t64
time_t=8 off_t=8
```

(**Tested (QEMU)**, BeagleBone Black settings.) These flags change how libraries exchange time values, so a program and its libraries must use the same setting.

### The kernel

The toolchain contains the kernel's user-facing headers. [Older headers work on a newer kernel, but not always the reverse](https://docs.kernel.org/kbuild/headers_install.html), so [keep them the same as or older than the kernel you run](https://buildroot.org/downloads/manual/manual.html).

You may read about `FATAL: kernel too old`. Programs printed it when the kernel was below glibc's minimum, but [glibc 2.36 removed that check](https://sourceware.org/git/?p=glibc.git;a=blob_plain;f=NEWS;hb=HEAD), so you will only see it with older versions.

---

## 12. Sysroots and leaks from the PC

**Read this when** your project needs libraries from the board, such as zlib or OpenSSL.

A **sysroot** is a folder laid out like the target's filesystem, with the headers and libraries the compiler needs: `usr/include`, `usr/lib`, and so on. `--sysroot=DIR` tells the compiler and linker to use it.

Ubuntu 22.04's cross compilers also search the PC's `/usr/include`, and other releases may too. Check yours with `${CROSS_COMPILE}gcc -E -v - </dev/null`. A header installed on your PC can be picked up with no error.

**Tested (QEMU)**, with zlib's development package installed on the PC only:

```console
$ printf '#include <zlib.h>\nint main(void) { return 0; }\n' > z.c
$ ${CROSS_COMPILE}gcc -H -c z.c 2>&1 | head -1
. /usr/include/zlib.h
$ ${CROSS_COMPILE}gcc --sysroot=/nonexistent -c z.c 2>&1 | head -1
z.c:1:10: fatal error: zlib.h: No such file or directory
```

The first compile succeeded with the **PC's** header. With a sysroot, the missing header is reported, and that failure is what you want. `-H` lists the headers used.

**Getting a sysroot** (**Illustrative**):

```sh
# Copy from a running board that has the -dev packages installed
rsync -aR $BOARD:/lib $BOARD:/usr/include $BOARD:/usr/lib ~/cross/sysroot/
# Convert absolute symlinks, which would otherwise point into your PC
symlinks -rc ~/cross/sysroot

# Then build against it
${CROSS_COMPILE}gcc --sysroot=$HOME/cross/sysroot app.c -lz -o app
```

Ubuntu 22.04's cross compiler keeps its own glibc headers ahead of the sysroot's, so use the same distribution release for both. Buildroot and Yocto generate matching toolchain and sysroot pairs, and they avoid this problem.

---

## 13. pkg-config

**Read this when** a project finds its libraries with `pkg-config`.

`pkg-config` prints the flags needed to use a library, and by default it reads the PC's files. For a cross build, point it at the target's:

| Variable | Effect |
|---|---|
| `PKG_CONFIG_LIBDIR` | **Replaces** the search path. Use this one |
| `PKG_CONFIG_PATH` | **Adds to** the search path, so the PC's libraries stay visible. Avoid it for a cross build |
| `PKG_CONFIG_SYSROOT_DIR` | Added in front of the paths in the output |

```sh
export PKG_CONFIG_LIBDIR=$STAGING/usr/lib/pkgconfig
export PKG_CONFIG_SYSROOT_DIR=$STAGING
```

The two common implementations behave differently here. For a library with `prefix=/usr` in `$STAGING`:

| Tool (tested version) | Output |
|---|---|
| `pkgconf` 1.8.0 | `-I$STAGING/usr/include -L$STAGING/usr/lib -lchk` |
| `pkg-config` 0.29.2 | `-lchk` |

`pkg-config` 0.29 treats `/usr/include` and `/usr/lib` as standard paths and drops them. That's fine with a real `--sysroot`, but in a separate staging folder the compile can't find the headers. Setting `PKG_CONFIG_ALLOW_SYSTEM_CFLAGS=1` and `PKG_CONFIG_ALLOW_SYSTEM_LIBS=1` keeps them. (**Tested (QEMU)**, BeagleBone Black settings.)

---

## 14. Meson and Autotools

**Read this when** a project you're building uses one of these.

### Meson

Meson uses a cross file, the equivalent of CMake's toolchain file. `~/cross/bbb.ini`:

```ini
[binaries]
c = 'arm-linux-gnueabihf-gcc'
ar = 'arm-linux-gnueabihf-ar'
strip = 'arm-linux-gnueabihf-strip'
pkg-config = 'pkg-config'
pkgconfig = 'pkg-config'

[host_machine]
system = 'linux'
cpu_family = 'arm'
cpu = 'cortex-a8'
endian = 'little'
```

```sh
meson setup build-meson --cross-file ~/cross/bbb.ini --prefix=/usr
meson compile -C build-meson
```

**Tested (QEMU).** Both `pkg-config` keys are there on purpose: [Meson 1.3 renamed `pkgconfig` to `pkg-config`](https://mesonbuild.com/Release-notes-for-1-3-0.html), and setting both works on old and new versions.

### Autotools

```sh
./configure --build=x86_64-linux-gnu --host=arm-linux-gnueabihf --prefix=/usr
make
make install DESTDIR=$STAGING
```

Give both `--build` and `--host`: with `--host` alone, `configure` tries running a test program to decide if it's cross-compiling, and with automatic QEMU emulation that test can succeed. (`--host` here means the board, as warned in section 2.)

When a check needs to run target code, `configure` stops. Find the variable in the project's `configure.ac`, work out the right answer for your board, and pass it.

**Tested (QEMU)**, with a test check named `my_cv_mmap_ok`:

```console
$ ./configure --host=aarch64-linux-gnu
checking whether mmap works here... configure: error: in `/home/me/runtest':
configure: error: cannot run test program while cross compiling

$ ./configure --host=aarch64-linux-gnu my_cv_mmap_ok=yes
checking whether mmap works here... (cached) yes
```

---

## 15. Choosing a toolchain

**Read this when** you go beyond this tutorial, on a real product.

**The rule: use the toolchain from whoever built the board's software.**

| Your board's software | Use |
|---|---|
| Built with Buildroot | Its toolchain (`output/host/bin`), or `make sdk` |
| Built with Yocto | Its SDK (`bitbake <image> -c populate_sdk`) |
| A Debian or Ubuntu release | The cross packages of the same release, plus a sysroot from that release |
| A vendor's image | The vendor's SDK |
| None of those | A toolchain whose glibc isn't newer than the board's: [Arm GNU Toolchain](https://developer.arm.com/downloads/-/arm-gnu-toolchain-downloads), [Bootlin](https://toolchains.bootlin.com), or one you configure with [crosstool-NG](https://crosstool-ng.github.io/docs/) |

A newer compiler isn't automatically better for a product. Keep one fixed toolchain per release, because a compiler update changes every binary.

Other options exist, like `clang --target=arm-linux-gnueabihf` (one compiler for all architectures), which still needs a target C library. Prebuilt toolchains weren't tested for this note.

---

## 16. Debugging with GDB

**Read this when** you need to step through a program on the target. Install `gdb-multiarch` on the PC.

Build with `-g`. QEMU has a built-in GDB server, so you can try this without a board. The `-g 1234` given to `$QEMU` makes it wait for the debugger.

**Tested (QEMU)**, Raspberry Pi settings, with the first lines of GDB's output trimmed:

```console
$ ${CROSS_COMPILE}gcc -g -O0 hello.c -o dbg
$ $QEMU -g 1234 ./dbg &
$ gdb-multiarch -q -batch -ex 'set sysroot /usr/aarch64-linux-gnu' \
      -ex 'target remote :1234' -ex 'break main' -ex c -ex bt -ex c ./dbg
Breakpoint 1, main () at hello.c:5
5	    printf("hello from glibc %s\n", gnu_get_libc_version());
#0  main () at hello.c:5
[Inferior 1 (process 1) exited normally]
```

On a board, `gdbserver` plays the part of QEMU's server.

**Expected (board):** with `gdbserver` installed on the board and `dbg` copied there:

```sh
# on the board
gdbserver :2345 ./dbg

# on the PC
gdb-multiarch -q ./dbg -ex 'set sysroot /path/to/sysroot' -ex 'target remote BOARD_IP:2345'
```

`set sysroot` must point at copies of the board's libraries, or backtraces through them are unusable. You should see the same breakpoint output as above.

---

## 17. Where the rest is covered

- **Cross-compiling the kernel and U-Boot** (`ARCH=` and `CROSS_COMPILE=`): the [kernel](../kernel/) and [bootloader](../bootloader/bootloader.md) notes.
- **Root filesystems, and full-system emulation in QEMU:** the [root filesystem](../rootfs/) note.
- **Toolchains and SDKs from build systems:** the [Buildroot](../../linux-build-systems/buildroot/) and [Yocto](../../linux-build-systems/yocto/) notes.
- **How a toolchain is built, bare-metal toolchains, Go and Rust:** not covered here.

---

## References

Compilers and tools:

- GCC manual: [ARM options](https://gcc.gnu.org/onlinedocs/gcc/ARM-Options.html), [AArch64 options](https://gcc.gnu.org/onlinedocs/gcc/AArch64-Options.html), [`--sysroot`](https://gcc.gnu.org/onlinedocs/gcc/Directory-Options.html)
- [QEMU user-mode emulation](https://www.qemu.org/docs/master/user/main.html)
- [GDB remote debugging](https://sourceware.org/gdb/current/onlinedocs/gdb.html/Remote-Debugging.html)

C library and kernel:

- [glibc NEWS](https://sourceware.org/git/?p=glibc.git;a=blob_plain;f=NEWS;hb=HEAD), including the removal of the kernel version check in 2.36
- [Kernel: exporting headers](https://docs.kernel.org/kbuild/headers_install.html)
- [Debian 13 release notes](https://www.debian.org/releases/trixie/release-notes/whats-new.html), section on 64-bit `time_t`
- glibc in Debian: [12](https://packages.debian.org/bookworm/libc6), [13](https://packages.debian.org/trixie/libc6)

Build systems:

- [CMake toolchains manual](https://cmake.org/cmake/help/latest/manual/cmake-toolchains.7.html)
- [Meson cross compilation](https://mesonbuild.com/Cross-compilation.html)
- [Autoconf manual](https://www.gnu.org/software/autoconf/manual/autoconf.html), "Manual Configuration"
- [Buildroot manual](https://buildroot.org/downloads/manual/manual.html)

Boards:

- BeagleBone Black: [board page](https://www.beagleboard.org/boards/beaglebone-black), [images](https://www.beagleboard.org/distros), [getting started](https://docs.beagleboard.org/intro/support/getting-started.html)
- Raspberry Pi 4: [specifications](https://www.raspberrypi.com/products/raspberry-pi-4-model-b/specifications/), [operating systems](https://www.raspberrypi.com/software/operating-systems/)
