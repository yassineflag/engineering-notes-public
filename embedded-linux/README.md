# Embedded Linux

Notes on building and running Linux on embedded targets: bootloaders, kernel, root filesystems, Yocto/Buildroot, and on-target debugging.

## Notes

### Fundamentals

- [Cross-Compilation and Toolchains for Embedded Linux](fundamentals/cross-compilation-toolchain/cross-compilation-toolchain.md): a beginner's path from a PC to a C program on an Arm board: the concepts and terms, a working example with a shared library, Make and CMake, testing in QEMU, running on a BeagleBone Black or Raspberry Pi 4, then optional depth (glibc versions, CPU flags, sysroots, pkg-config, Meson, Autotools, choosing a toolchain, remote GDB) and troubleshooting by symptom.
- [Bootloader](fundamentals/bootloader/)
- [Kernel](fundamentals/kernel/)
- [Root filesystem](fundamentals/rootfs/)

### Linux build systems

- [Yocto](linux-build-systems/yocto/)
- [Buildroot](linux-build-systems/buildroot/)
