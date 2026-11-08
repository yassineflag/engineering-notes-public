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

Planned: why build on a PC, and the three things that must match: CPU, ABI, libraries.

---

## 2. The words you need

Planned: build machine, target, toolchain, ABI, C library, userspace, triplets, and the confusing word "host".

---

## 3. The boards, and why their software matters

Planned: why the board's software matters to a toolchain; the BeagleBone Black and Raspberry Pi 4 assumptions; which PCs work for each.

---

## 4. Install and check the toolchain

Planned: install the cross compilers and QEMU; one settings file per board; check the compiler and its glibc.

---

## 5. Build your first program

Planned: compile hello.c, see why it does not run on the PC, read it with file and readelf.

---

## 6. Run it in QEMU

Planned: QEMU user mode, why -L and -cpu matter, what QEMU does and does not prove.

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
