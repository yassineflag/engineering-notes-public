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

Planned: connect to each board, check what it runs, copy and run the program, with expected output.

---

## 8. Add a shared library

Planned: a small library and program: soname, NEEDED, LD_LIBRARY_PATH, deployment.

---

## 9. Use Make or CMake

Planned: the same project with a Makefile and with a CMake toolchain file; staging and deployment.

---

## 10. Troubleshooting

Planned: symptoms, causes and fixes, marking what was reproduced.

---

## 11. Matching the target

Planned: glibc versions, CPU flags, silent type differences, float ABI, 64-bit time, the kernel.

---

## 12. Sysroots and leaks from the PC

Planned: what a sysroot is, headers leaking from the PC, getting a sysroot.

---

## 13. pkg-config

Planned: search paths, the sysroot prefix, and the differences between pkg-config and pkgconf.

---

## 14. Meson and Autotools

Planned: short recipes: a cross file, --build and --host, and answering checks that cannot run.

---

## 15. Choosing a toolchain

Planned: one rule and a short table.

---

## 16. Debugging with GDB

Planned: QEMU's GDB server, then gdbserver on a board.

---

## 17. Where the rest is covered

Planned: links to the kernel, bootloader, root filesystem and build system notes.

---

## References

Planned: compiler, C library, build system and board documentation.
