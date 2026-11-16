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

A Linux system is layers of programs. Only the bottom one is "Linux" in the strict sense:

```
 you ── terminal window ── shell (bash) ── commands: ls, cp, apt, systemctl, ssh ...
                                                │
                       userspace ───────────────┤  every program that isn't the kernel:
                                                │  shell, tools, services, desktop, C library
 ───────────────────────────────── system calls (open, read, write, fork, exec ...)
                       kernel ──────────────────┤  Linux: processes, memory, files,
                                                │  drivers, network, permissions
 ─────────────────────────────────
                       hardware: CPU, RAM, disks, USB, network cards
```

- **The kernel** runs the hardware and shares it between programs. Programs never touch the hardware directly; they ask the kernel through **system calls** ("open this file", "start this program").
- **Userspace** means everything that runs outside the kernel: your shell, the tools, the services, the desktop. The word appears in every Linux text; when it does, it means "not the kernel".
- **A distribution** (Debian, Ubuntu, Fedora, Arch) is a kernel plus thousands of userspace programs, built to work together, plus a package manager to install and update them, plus an init system that starts everything at boot. Most commands in this note are the same on every distribution; the package manager and some configuration files differ.
- **The shell** is an ordinary program that reads a line, splits it into a command and arguments, runs the command and waits. Bash is the default on Debian and Ubuntu; zsh and fish are alternatives. The shell is not the terminal: the terminal (GNOME Terminal, Konsole, a serial line, an SSH session) only shows characters and sends your keystrokes.

**Everything is a file.** Linux shows most things through file names, so the same tools work on all of them:

| What | Example | What reading it gives |
|---|---|---|
| A regular file | `/etc/hostname` | Its contents |
| A directory | `/home/user` | The list of names in it (through `ls`) |
| A device | `/dev/ttyUSB0`, `/dev/sda` | Bytes from the serial port, sectors of the disk |
| Kernel information | `/proc/cpuinfo`, `/proc/1/status` | Generated on each read by the kernel |
| Kernel settings | `/sys/class/leds/*/brightness` | A value you can also write to change it |
| A pipe or socket | `/run/systemd/journal/socket` | A channel between two programs |

Those files under `/proc` and `/sys` aren't on any disk: the kernel makes them up when you read them.

**What am I running?**

```
$ cat /etc/os-release        # the distribution and its version
$ uname -r                   # the running kernel's release
$ echo "$SHELL"              # your login shell
$ ps -p 1 -o comm=           # the init system (PID 1)
```

**Expected:**

```
PRETTY_NAME="Debian GNU/Linux 13 (trixie)"
NAME="Debian GNU/Linux"
VERSION_ID="13"
VERSION="13 (trixie)"
VERSION_CODENAME=trixie
ID=debian
...
6.12.x-amd64
/bin/bash
systemd
```

## 2. The terminal, the shell and getting help

**Where you type.** A terminal window in the desktop is the usual place. Without a desktop, `Ctrl+Alt+F3` to `F6` switch to text consoles (`Ctrl+Alt+F1` or `F2` return to the desktop on Debian and Ubuntu, depending on the version). Over the network, SSH gives you the same shell (section 20); on a board, so does the serial console (section 18).

**The prompt** tells you who and where you are:

```
user@pc:~/work$          normal user "user" on host "pc", in ~/work, "$" = not root
root@pc:/etc#            root, in /etc, "#" = be careful
```

**Anatomy of a command:**

```
$ ls  -l -a  --human-readable  /etc  /var/log
  │   └──┬─┘  └──────┬──────┘  └──────┬──────┘
  │   short options   long option     arguments
  command
```

- Short options can be grouped: `ls -la` is `ls -l -a`. Some take a value: `head -n 5` or `head -n5`.
- Long options start with `--`: `--human-readable`, `--color=auto`.
- `--` alone ends the options, for an argument that starts with `-`: `rm -- -strange-file`.
- Spaces separate arguments. A file name with a space must be quoted: `ls "My Documents"` (section 4).

**Keys worth learning on day one:**

| Key | Effect |
|---|---|
| `Tab` | Complete a command, file name or option; twice lists the choices |
| `↑` / `↓` | Previous / next command from history |
| `Ctrl+R` | Search the history as you type; `Ctrl+R` again for older matches |
| `Ctrl+C` | Interrupt the running command (sends SIGINT, section 12) |
| `Ctrl+D` | End of input: closes the shell if the line is empty |
| `Ctrl+Z` | Suspend the running command (section 12) |
| `Ctrl+A` / `Ctrl+E` | Start / end of the line |
| `Ctrl+U` / `Ctrl+W` | Delete to the start of the line / the previous word |
| `Ctrl+L` | Clear the screen (same as `clear`) |
| `Alt+.` | Insert the last argument of the previous command |
| `!!` | The previous command: `sudo !!` re-runs it with `sudo` |

In a terminal window, copy and paste are `Ctrl+Shift+C` and `Ctrl+Shift+V`, because `Ctrl+C` already means "interrupt". Selecting text with the mouse and clicking the middle button also pastes it.

**Getting help.** Every tool documents itself; learning to read that documentation saves more time than any list of commands:

| Command | Gives |
|---|---|
| `man ls` | The manual page: every option, with examples near the end on many pages |
| `ls --help` | A short summary of the options |
| `help cd` | Help for a shell builtin (`cd`, `export`, `read`, `for` ...), which has no man page of its own |
| `type cd` / `type ls` | Whether a name is a builtin, an alias, a function or a program, and where |
| `man -k partition` | Search the man page titles (same as `apropos partition`) |
| `tldr tar` | Short, example-first pages (package `tealdeer` on Debian) |
| `/usr/share/doc/<package>/` | Package notes, examples, changelogs |

Inside `man`, the keys are those of `less` (section 5): `/word` searches, `n` goes to the next match, `q` quits. Man pages are grouped in numbered **sections**, which is why you'll see names like `crontab(5)`:

| Section | Contents | Example |
|---|---|---|
| 1 | User commands | `man 1 passwd`: the command |
| 5 | File formats | `man 5 passwd`: the `/etc/passwd` file |
| 8 | Administration commands | `man 8 mount` |
| 2, 3 | System calls, C library functions | `man 2 open`, `man 3 printf` (package `manpages-dev`) |
| 7 | Overviews | `man 7 signal`, `man 7 hier` |

**History.** Bash keeps your commands in `~/.bash_history` (written when the shell exits). `history | grep ssh` finds an old command; `!123` re-runs entry 123. Debian's default `~/.bashrc` sets `HISTCONTROL=ignoreboth`, so a command typed with a leading space is not saved: useful for a command that contains a password, though it's better not to put passwords on command lines at all, since other users can see them in the process list (section 12).

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
