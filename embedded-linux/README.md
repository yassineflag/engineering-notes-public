# Embedded Linux

Notes on building and running Linux on embedded targets: bootloaders, kernel, root filesystems, Yocto/Buildroot, and on-target debugging.

## Notes

### Fundamentals

- [Cross-Compilation and Toolchains for Embedded Linux](fundamentals/cross-compilation-toolchain/cross-compilation-toolchain.md): a beginner's path from a PC to a C program on an Arm board: the concepts and terms, a working example with a shared library, Make and CMake, testing in QEMU, running on a BeagleBone Black or Raspberry Pi 4, then optional depth (glibc versions, CPU flags, sysroots, pkg-config, Meson, Autotools, choosing a toolchain, remote GDB) and troubleshooting by symptom.
- [Bootloaders for Embedded Linux: U-Boot](fundamentals/bootloader/bootloader.md): a beginner's path from power-on to a running kernel: what a bootloader does and how each board boots, building U-Boot, its command line and environment in QEMU, starting Linux by hand and automatically (boot scripts, extlinux.conf), U-Boot on a BeagleBone Black and a Raspberry Pi 4, then optional depth (load addresses, FIT images, device tree overlays, network boot, fw_setenv, safe updates and verified boot, GDB) and troubleshooting by symptom.
- [Kernel](fundamentals/kernel/)
- [Root filesystem](fundamentals/rootfs/)

### Linux build systems

- [Yocto](linux-build-systems/yocto/)
- [Buildroot](linux-build-systems/buildroot/)
