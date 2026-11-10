# Embedded Linux

Notes on building and running Linux on embedded targets: bootloaders, kernel, root filesystems, Yocto/Buildroot, and on-target debugging.

## Notes

### Fundamentals

- [Cross-Compilation and Toolchains for Embedded Linux](fundamentals/cross-compilation-toolchain/cross-compilation-toolchain.md): a beginner's path from a PC to a C program on an Arm board: the concepts and terms, a working example with a shared library, Make and CMake, testing in QEMU, running on a BeagleBone Black or Raspberry Pi 4, then optional depth (glibc versions, CPU flags, sysroots, pkg-config, Meson, Autotools, choosing a toolchain, remote GDB) and troubleshooting by symptom.
- [Bootloaders for Embedded Linux: U-Boot](fundamentals/bootloader/bootloader.md): a beginner's path from power-on to a running kernel: what a bootloader does and how each board boots, building U-Boot, its command line and environment in QEMU, starting Linux by hand and automatically (boot scripts, extlinux.conf), U-Boot on a BeagleBone Black and a Raspberry Pi 4, then optional depth (load addresses, FIT images, device tree overlays, network boot, fw_setenv, safe updates and verified boot, GDB) and troubleshooting by symptom.
- [The Linux Kernel for Embedded Linux](fundamentals/kernel/kernel.md): from the bootloader's jump to `init` on a BeagleBone Black with Linux 7.1-rc1: what the kernel receives and must find, choosing a kernel, Kconfig and config fragments, building out of tree, the release string and module vermagic, installing on the SD card, the command line, the boot log stage by stage, the device tree from the kernel's side, root mounting, initramfs and init, modules and out-of-tree modules, keeping patches across releases, then debugging (earlycon, oops decoding, pstore, debugfs, ftrace), size and boot time, maintenance and hardening, arm64 and the Raspberry Pi 4, and troubleshooting by symptom.
- [Root filesystem](fundamentals/rootfs/)

### Linux build systems

- [Yocto](linux-build-systems/yocto/)
- [Buildroot](linux-build-systems/buildroot/)
