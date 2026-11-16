# Everyday Linux: The Shell, Files, Users, Packages and Services

This note covers the Linux you use every day on your own machine: the shell, files and permissions, users and `sudo`, pipes and small scripts, processes, systemd services and logs, packages, disks, devices, the network from the user's side, SSH, archives and backups. It is the note to read before the embedded Linux notes, and the one to come back to when something on the PC gets in the way: a `Permission denied` on a serial port, a full disk, a service that won't start, an SD card to write without wiping the wrong disk.

Each part starts from the concepts, then shows the commands you'll actually type, then the traps. Where a task belongs to embedded work (a USB serial adapter, an SD card image, SSH into a board), the example uses the BeagleBone Black of the [embedded Linux notes](../../embedded-linux/). The [networking fundamentals note](../../networking/fundamentals/fundamentals_v1_2026-10-29.md) covers the network in depth; section 19 here only covers what a user needs.

**Assumptions:**
- The machine is a PC (x86_64) running Debian 13 "trixie", with Bash as the login shell, systemd as the init system and APT for packages. Ubuntu 24.04 LTS and 26.04 LTS work the same way for almost everything here; the differences are noted where they matter (Snap, netplan, `sudo-rs`).
- The user is `user`, the PC is `pc`, and the user can run `sudo` (section 7 shows how to set it up).
- The board, where one appears, is a BeagleBone Black at `192.168.0.50` with a USB serial adapter on its console, as in the other notes. Device names (`/dev/sdX`, `/dev/ttyUSB0`) are placeholders: section 17 shows how to find yours.

**Conventions:**
- `$` is a shell as your normal user, `#` a root shell (or a command that needs `sudo`).
- **Expected** marks output reconstructed from the tools' usual messages, not captured. Version numbers, sizes, PIDs and dates in it are placeholders.
- **Illustrative** marks a file or script that shows the shape of a solution and needs adapting.
- `<...>` in a command is a value to replace.

**Verification status:** nothing in this note was run for it. The commands and file formats follow the manual pages and Debian documentation as known for Debian 12 and 13 (Bash 5.2, GNU coreutils 9, systemd 257, APT 3.0, OpenSSH 9 and 10); they were not re-read line by line for this note. Output blocks are Expected. Scripts and unit files are Illustrative and untested: read them before using them, especially anything that writes to a disk.

**How to read it:**
- **New to Linux:** Parts 1 to 4 in order, then the rest as needed.
- **Setting up a PC for embedded work:** sections 7 (groups), 15 (packages), 17 (SD cards), 18 (serial console) and 20 (SSH).
- **Something is broken:** section 23, the symptom tables, then the section they point to.

## Table of contents

**Part 1: The system and the shell**

- [1. The mental model: kernel, distribution, shell, files](#1-the-mental-model-kernel-distribution-shell-files)
- [2. The terminal, the shell and getting help](#2-the-terminal-the-shell-and-getting-help)

**Part 2: Files**

- [3. The directory tree and paths](#3-the-directory-tree-and-paths)
- [4. Working with files: create, copy, move, delete, link](#4-working-with-files-create-copy-move-delete-link)
- [5. Reading and editing text](#5-reading-and-editing-text)
- [6. Finding files and text](#6-finding-files-and-text)

**Part 3: Users and permissions**

- [7. Users, groups and sudo](#7-users-groups-and-sudo)
- [8. Permissions and ownership](#8-permissions-and-ownership)

**Part 4: The shell as a tool**

- [9. Pipes, redirection and text processing](#9-pipes-redirection-and-text-processing)
- [10. Variables, the environment and shell startup files](#10-variables-the-environment-and-shell-startup-files)
- [11. Small shell scripts](#11-small-shell-scripts)

**Part 5: Processes, services and logs**

- [12. Processes and signals](#12-processes-and-signals)
- [13. Services with systemd](#13-services-with-systemd)
- [14. Logs: the journal, /var/log and dmesg](#14-logs-the-journal-varlog-and-dmesg)

**Part 6: Software**

- [15. Packages with APT and dpkg](#15-packages-with-apt-and-dpkg)
- [16. Software outside APT](#16-software-outside-apt)

**Part 7: Disks and hardware**

- [17. Disks, partitions, filesystems and SD cards](#17-disks-partitions-filesystems-and-sd-cards)
- [18. Hardware, devices and the serial console](#18-hardware-devices-and-the-serial-console)

**Part 8: Networks and remote work**

- [19. The network from the user's side](#19-the-network-from-the-users-side)
- [20. SSH, file transfer and tmux](#20-ssh-file-transfer-and-tmux)
- [21. Archives, checksums and backups](#21-archives-checksums-and-backups)

**Part 9: Keeping it running**

- [22. Housekeeping: time, names, updates and disk space](#22-housekeeping-time-names-updates-and-disk-space)
- [23. Troubleshooting by symptom](#23-troubleshooting-by-symptom)
- [24. Not covered here](#24-not-covered-here)
- [References](#references)

## 1. The mental model: kernel, distribution, shell, files

## 2. The terminal, the shell and getting help

## 3. The directory tree and paths

## 4. Working with files: create, copy, move, delete, link

## 5. Reading and editing text

## 6. Finding files and text

## 7. Users, groups and sudo

## 8. Permissions and ownership

## 9. Pipes, redirection and text processing

## 10. Variables, the environment and shell startup files

## 11. Small shell scripts

## 12. Processes and signals

## 13. Services with systemd

## 14. Logs: the journal, /var/log and dmesg

## 15. Packages with APT and dpkg

## 16. Software outside APT

## 17. Disks, partitions, filesystems and SD cards

## 18. Hardware, devices and the serial console

## 19. The network from the user's side

## 20. SSH, file transfer and tmux

## 21. Archives, checksums and backups

## 22. Housekeeping: time, names, updates and disk space

## 23. Troubleshooting by symptom

## 24. Not covered here

## References

---

Version 1 · 2026-11-16
