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

There are no drive letters: one tree starts at `/` (the **root directory**), and every disk, USB stick and SD card is attached somewhere inside it (section 17). The layout follows the Filesystem Hierarchy Standard (FHS; `man 7 hier`):

| Directory | Holds | Notes |
|---|---|---|
| `/` | The top of the tree | Not the same as `/root` |
| `/usr/bin`, `/usr/sbin` | Programs installed by packages | `/bin`, `/sbin` and `/lib` are links to their `/usr` versions since Debian 12 ("merged /usr") |
| `/usr/lib` | Libraries, and default configuration installed by packages (systemd units, udev rules) | Don't edit files here; override them in `/etc` |
| `/usr/local` | Software you installed yourself, outside the package manager | Section 16 |
| `/usr/share` | Architecture-independent data: man pages, docs, icons | |
| `/etc` | System-wide configuration, plain text | Back it up; most admin work is editing files here |
| `/home/user` | Your files and your personal configuration (dotfiles) | `~` is a short name for it |
| `/root` | The root user's home | |
| `/boot` | The kernel, initramfs and bootloader files | |
| `/dev` | Device files: disks, serial ports, terminals | Created by the kernel and udev (section 18) |
| `/proc`, `/sys` | Kernel information and settings, generated on the fly | Section 1 |
| `/run` | Runtime state (PID files, sockets), cleared at each boot | In RAM |
| `/tmp` | Temporary files, anyone can write there | In RAM (tmpfs) and cleared at boot since Debian 13 |
| `/var` | Data that changes: logs (`/var/log`), caches, spools, databases | Grows over time (section 22) |
| `/opt` | Self-contained third-party software (vendor toolchains, IDEs) | |
| `/media/user`, `/mnt` | Mount points: removable media are mounted under `/media/user` by the desktop, `/mnt` is for your own temporary mounts | |
| `/srv` | Data served by this machine (a web or TFTP root) | |

**Paths.** An **absolute path** starts at `/`: `/etc/ssh/sshd_config`. A **relative path** starts from the current directory: if you are in `/etc`, `ssh/sshd_config` is the same file. Special names:

| Name | Means |
|---|---|
| `.` | The current directory: `./build.sh` runs the script here |
| `..` | The parent directory: `cd ..` goes up one level |
| `~` | Your home directory; `~alice` is alice's |
| `-` | With `cd`: the previous directory (`cd -` toggles between two) |

```
$ pwd                 # print the current directory
/home/user
$ cd /var/log         # absolute
$ cd ../lib           # relative: now in /var/lib
$ cd                  # alone: back home
```

**File names** are case-sensitive (`Notes.txt` and `notes.txt` are two files) and may contain anything except `/` and the NUL byte, including spaces, newlines and leading dashes. Extensions mean nothing to the system; `file` tells you what a file really is. Names that start with `.` are **hidden**: `ls` doesn't show them unless asked with `-a`. Your personal configuration lives in such dotfiles: `~/.bashrc`, `~/.ssh/`, `~/.config/`.

**Reading `ls -l`:**

```
$ ls -la ~/work
```

**Expected:**

```
total 24
drwxr-xr-x  4 user user 4096 Nov 16 10:12 .
drwx------ 31 user user 4096 Nov 16 09:58 ..
-rw-r--r--  1 user user  220 Nov 16 10:01 .gitignore
drwxr-xr-x  2 user user 4096 Nov 16 10:12 images
-rwxr-xr-x  1 user user  812 Nov 16 10:05 sd-write.sh
lrwxrwxrwx  1 user user   27 Nov 16 10:07 latest.img -> images/bbb-2026-11-16.img
│└───┬───┘  │ └┬─┘ └┬─┘ └┬─┘ └────┬─────┘
│  mode   links owner group size  modified
type: - file, d directory, l symbolic link, c/b character/block device
```

The mode (`rwxr-xr-x`) is section 8. `ls -lh` prints sizes as `4.0K`, `1.2G`; `ls -lt` sorts by time, newest first; `ls -ltr` reverses it, so the newest is at the bottom, next to your prompt. For one file in detail:

```
$ stat sd-write.sh       # size, mode, owner, inode, the three timestamps
$ file latest.img        # what the contents are
```

## 4. Working with files: create, copy, move, delete, link

| Task | Command | Notes |
|---|---|---|
| Create a directory | `mkdir images` | `mkdir -p a/b/c` creates the parents too, and doesn't fail if it exists |
| Create an empty file, or update its time | `touch notes.txt` | |
| Copy a file | `cp a.txt b.txt` | Overwrites `b.txt` without asking; `-i` asks |
| Copy into a directory | `cp a.txt b.txt backup/` | The last argument is the destination |
| Copy a directory | `cp -r src/ dst/` | `-a` (archive) also keeps modes, owners, times and links: use it for anything that matters |
| Move or rename | `mv old.txt new.txt` | Same as rename when on the same filesystem; instant |
| Delete a file | `rm file` | **No trash, no undo** |
| Delete a directory and its contents | `rm -r dir/` | `-f` also silences errors and never asks |
| Delete an empty directory | `rmdir dir/` | Fails if it isn't empty: a safe check |
| Symbolic link | `ln -s target linkname` | A small file holding a path; breaks if the target moves |
| Hard link | `ln target linkname` | A second name for the same data; same filesystem only, not for directories |

**Links.** `latest.img -> images/bbb-2026-11-16.img` above is a symbolic link: programs that open `latest.img` get the image. The target path is stored as written; a relative target is relative to the link's own directory, not to where you were when you made it. `readlink -f latest.img` prints the final absolute path. Deleting a symbolic link (`rm latest.img`, without a trailing `/`) deletes the link, not the target.

**Wildcards (globbing).** The shell, not the command, expands patterns into the matching names before running the command:

| Pattern | Matches | Example |
|---|---|---|
| `*` | Any characters, none included (not a leading `.`) | `rm *.o` |
| `?` | One character | `ls img?.png` |
| `[abc]`, `[0-9]` | One character from the set | `ls log.[0-9]` |
| `{a,b}` | Each word in turn (brace expansion: makes names even if they don't exist) | `mkdir -p {src,doc,test}`, `cp file{,.bak}` |
| `**` | Any depth of directories, after `shopt -s globstar` | `ls **/*.dts` |

If nothing matches, Bash passes the pattern unchanged, so `rm *.tmp` in a directory without such files fails with `No such file or directory` for a file literally named `*.tmp`.

**Quoting** decides what the shell expands:

| Form | Effect | Example |
|---|---|---|
| `'...'` | Nothing expanded: the text is taken as written | `echo '$HOME *'` prints `$HOME *` |
| `"..."` | Variables and `$(...)` expanded; no word splitting, no wildcards | `cp "$file" "$HOME/My Documents/"` |
| `\` | The next character is taken literally | `ls My\ Documents` |

Always put variables in double quotes: `rm $file` with `file="my notes.txt"` deletes `my` and `notes.txt`.

**Habits that prevent disasters:**
- Run the glob with `ls` before you run it with `rm`: `ls build/*.img`, then `rm build/*.img`. Or put `echo` in front of a command to see what it would do.
- Never `rm -rf "$DIR/"` with a variable that might be empty: it becomes `rm -rf /`. Write `rm -rf "${DIR:?}/"`; Bash stops with an error if `DIR` is empty or unset.
- Prefer `mv` to a `~/trash` directory, or `gio trash file` on a desktop, for anything you might want back.
- `cp -a` and `rsync -a` (section 20) keep permissions and ownership; plain `cp -r` doesn't, which matters for root filesystem trees.

## 5. Reading and editing text

Most configuration on Linux is plain text, so reading and editing text files is the main skill.

| Command | Use |
|---|---|
| `cat file` | Print the whole file; good for short ones |
| `less file` | Page through a long file (keys below) |
| `head -n 20 file` | The first 20 lines |
| `tail -n 20 file` | The last 20 lines |
| `tail -f file` | Follow a growing file (a log) until `Ctrl+C`; `-F` keeps following when the file is rotated or recreated |
| `wc -l file` | Count lines (`-w` words, `-c` bytes) |
| `diff -u old new` | Differences in unified format, the same as `git diff` |
| `cmp a.bin b.bin` | Whether two binary files differ, and where |
| `xxd file \| head`, `hexdump -C file \| head` | Bytes in hex, for binary files and images |

**`less` keys:**

| Key | Effect |
|---|---|
| `Space` / `b` | Page down / up |
| `g` / `G` | Start / end of the file |
| `/text` / `?text` | Search forwards / backwards; `n` and `N` for next and previous |
| `&text` | Show only lines containing `text` |
| `F` | Follow the file like `tail -f`; `Ctrl+C` stops following |
| `-S` then `Enter` | Toggle wrapping of long lines |
| `q` | Quit |

**Editors.** Learn one terminal editor well enough to change a configuration file over SSH or on a board's serial console:
- **nano** is installed on Debian and Ubuntu and shows its keys at the bottom: `Ctrl+O` saves, `Ctrl+X` quits, `Ctrl+W` searches, `Alt+U` undoes. `^` in the help means `Ctrl`, `M-` means `Alt`.
- **vim** (or **vi**, which is often the only editor on a board: BusyBox has one) is modal: keys are commands until you press `i`. The minimum:

| Keys | Effect |
|---|---|
| `i` | Insert mode: type text |
| `Esc` | Back to command mode |
| `:w` `Enter` / `:q` / `:wq` / `:q!` | Save / quit / save and quit / quit without saving |
| `/text` `Enter`, then `n` | Search, then next match |
| `dd` / `u` / `Ctrl+R` | Delete the line / undo / redo |
| `gg` / `G` / `:42` | Start / end / line 42 |

Your editor for `git commit`, `crontab -e`, `visudo` and the like comes from the `EDITOR` and `VISUAL` variables (section 10), then from Debian's `editor` alternative: `sudo update-alternatives --config editor` sets it for the system, `select-editor` for your user.

**Editing system files.** Files in `/etc` belong to root. `sudoedit /etc/hosts` (or `sudo -e`) copies the file, opens your editor as **you**, and writes it back as root when you quit; your editor settings apply and the editor itself doesn't run as root. Before changing an important file, keep a copy: `sudo cp -a /etc/fstab /etc/fstab.orig`.

**Text that comes from Windows** has `\r\n` line endings, which break shell scripts with errors such as `/bin/bash^M: bad interpreter`. `file script.sh` shows `with CRLF line terminators`; `dos2unix script.sh` (package `dos2unix`) or `sed -i 's/\r$//' script.sh` fixes it.

## 6. Finding files and text

**`find` walks the tree** and tests each name; its arguments read as "where, then which tests, then what to do":

```
$ find ~/work -name '*.dtb'                       # by name (quote the pattern so the shell doesn't expand it)
$ find . -iname 'readme*'                         # case-insensitive
$ find /var/log -type f -mtime -1                 # files modified in the last 24 hours
$ find ~ -type f -size +500M                      # files over 500 MiB
$ find . -type f -newer build.stamp               # changed since that file was touched
$ find . -name '*.o' -delete                      # delete what matched (run without -delete first)
$ find . -name '*.sh' -exec chmod +x {} +         # run a command on the matches, many per call
$ find /etc -name '*.conf' 2>/dev/null            # hide "Permission denied" for directories you can't read
```

`find ... -print0 | xargs -0 cmd` passes names safely to another command, even names with spaces or newlines (section 9).

**`locate` searches a database** of every file name, so it's instant but only as fresh as the last update. On Debian it's the `plocate` package; the database is updated daily by a timer, or now with `sudo updatedb`.

```
$ locate sshd_config
```

**`grep` searches inside files:**

| Command | Finds |
|---|---|
| `grep ttyS0 /etc/inittab` | Lines that contain `ttyS0` |
| `grep -i error log.txt` | Ignoring case |
| `grep -rn 'CONFIG_USB' .` | Recursively, with file names and line numbers |
| `grep -rl 'TODO' src/` | Only the names of files that match |
| `grep -w eth0 file` | Whole words only: not `veth0` |
| `grep -v '^#' file` | Lines that **don't** match: here, without comment lines |
| `grep -E 'error\|fail'` or `grep -E '(error\|fail)'` | Extended regular expressions: `\|`, `+`, `?`, `()` without backslashes |
| `grep -r --include='*.c' 'ioctl' .` | Only in files matching a pattern |
| `grep -A 3 -B 1 'Oops' dmesg.txt` | With 3 lines after and 1 before each match |

Faster alternatives for source trees: `rg` (ripgrep, package `ripgrep`), which skips files ignored by `.gitignore`, and `fdfind` (package `fd-find`; `fd` on other distributions) for names.

**Finding programs and their packages:**

```
$ type -a python3          # what the shell will run, and every match in PATH
$ command -v picocom       # just the path; empty and status 1 if missing (good in scripts)
$ dpkg -S /usr/bin/scp    # which installed package owns a file
$ apt-file search bin/dtc # which package (installed or not) provides a file: package apt-file, then apt-file update
```

## 7. Users, groups and sudo

Every process runs as a **user** and belongs to **groups**; the kernel checks those against each file's owner and permissions (section 8). Users and groups are numbers (UID, GID); names are only a lookup in three files:

| File | One line per | Fields |
|---|---|---|
| `/etc/passwd` | User | `user:x:1000:1000:Full Name:/home/user:/bin/bash`: name, `x` (password elsewhere), UID, primary GID, comment, home, shell |
| `/etc/shadow` | User (readable by root only) | Name, password hash, password ageing |
| `/etc/group` | Group | `dialout:x:20:user`: name, `x`, GID, extra members |

UID 0 is **root**, which the kernel lets do almost anything. UIDs below 1000 are system users for services (`www-data`, `systemd-timesync`); the first human user gets 1000.

```
$ id
uid=1000(user) gid=1000(user) groups=1000(user),20(dialout),24(cdrom),27(sudo),46(plugdev),100(users),106(netdev)
$ whoami
user
$ getent passwd user      # works with LDAP and other sources too, not only /etc/passwd
```

**Becoming root, when needed.** Work as a normal user and raise privileges for one command at a time:

| Command | Effect |
|---|---|
| `sudo cmd` | Run one command as root, after your own password (remembered for 15 minutes in that terminal) |
| `sudo -u www-data cmd` | Run as another user |
| `sudo -i` | A root login shell (root's environment, in `/root`) |
| `sudo -s` | A root shell that keeps more of your environment |
| `su -` | A root login shell with **root's** password, if root has one |
| `sudo -l` | What you are allowed to run |

`sudo` works for members of the `sudo` group (`wheel` on Fedora and Arch). On Debian, if you gave a root password during installation, the installer didn't add your user to it; fix it once from a root shell, then log out and in again:

```
$ su -
# usermod -aG sudo user
# exit
```

Ubuntu has no root password by default and puts the first user in `sudo`. Since 25.10, Ubuntu's `sudo` is `sudo-rs`, a rewrite in Rust that takes the same commands and the same `sudoers` files for common uses.

**Managing users:**

| Task | Command |
|---|---|
| Add a user with a home and a password (Debian's friendly script) | `sudo adduser alice` |
| Add a user, low-level and identical everywhere | `sudo useradd -m -s /bin/bash alice`, then `sudo passwd alice` |
| Change your password | `passwd` |
| Add a user to a group | `sudo usermod -aG dialout alice` |
| Remove a user from a group | `sudo gpasswd -d alice dialout` |
| Lock / unlock an account | `sudo usermod -L alice` / `-U` |
| Delete a user and their home | `sudo deluser --remove-home alice` |
| Add a group | `sudo addgroup builders` |

**The `-a` in `usermod -aG` matters:** `usermod -G dialout alice` without it *replaces* all of alice's extra groups with `dialout`, removing `sudo` among others.

**Group changes apply at the next login.** A process gets its groups when the session starts; adding yourself to `dialout` changes nothing for shells that are already open. Log out of the desktop and back in (or reboot). To test in one terminal first: `newgrp dialout` starts a shell with the new group, or `su - $USER` starts a fresh login.

**Groups that matter on a development PC:**

| Group | Gives | Why you'd want it |
|---|---|---|
| `sudo` | `sudo` for everything | Administration |
| `dialout` | Serial ports: `/dev/ttyUSB*`, `/dev/ttyACM*`, `/dev/ttyS*` | Board consoles (section 18) |
| `plugdev` | Some USB devices whose udev rules use this group | Debug probes, SDRs, programmers |
| `adm`, `systemd-journal` | Reading all logs without `sudo` | Section 14 |
| `kvm` | `/dev/kvm` | Fast QEMU virtual machines |
| `wireshark` | Packet capture without root (if allowed at install) | |
| `docker` | The Docker daemon | **Equal to root**: anyone in it can mount `/` into a container. Rootless Podman avoids this |

**sudoers.** Rules live in `/etc/sudoers` and in files under `/etc/sudoers.d/`. Edit them only with `visudo`, which checks the syntax before saving: a broken `sudoers` file locks everyone out of `sudo`. A rule that lets the user restart one service without a password (**Illustrative**):

```
$ sudo visudo -f /etc/sudoers.d/user-tftp
user ALL=(root) NOPASSWD: /usr/bin/systemctl restart tftpd-hpa
```

Files in `/etc/sudoers.d/` are ignored if their name contains a `.` or ends in `~`, so don't name it `user.conf`.

## 8. Permissions and ownership

Every file has an **owner** (a user), a **group**, and a **mode**: three sets of three permission bits, for the owner (`u`), the group (`g`) and everyone else (`o`):

```
-rwxr-x---  1 user builders  812 Nov 16 10:05 sd-write.sh
 └┬┘└┬┘└┬┘
  │  │  └── others:  ---  nothing
  │  └───── group:   r-x  members of "builders" can read and run it
  └──────── owner:   rwx  user can read, change and run it
```

The kernel checks only **one** set: owner if you are the owner, else group if you are in the group, else others. Root skips these checks (except `x`: root can run a file only if someone has `x` on it).

**What r, w and x mean** depends on the type:

| Bit | On a file | On a directory |
|---|---|---|
| `r` | Read the contents | List the names in it (`ls`) |
| `w` | Change the contents | Create, delete and rename entries in it, **whatever the files' own permissions** |
| `x` | Run it as a program | Enter it and reach anything below (`cd`, open files inside by name) |

So deleting a file depends on the directory's `w`, not on the file's; and a file in a directory without `x` for you is unreachable even if the file is mode 777.

**Octal modes.** Each set is a digit: `r`=4, `w`=2, `x`=1, added up.

| Mode | Symbolic | Typical use |
|---|---|---|
| `644` | `rw-r--r--` | Ordinary files |
| `755` | `rwxr-xr-x` | Programs, scripts, directories |
| `600` | `rw-------` | Private files: SSH keys, files with passwords |
| `700` | `rwx------` | Private directories: `~/.ssh` |
| `664` / `775` | group can write | Files and directories shared in a group |

**Changing modes and owners:**

```
$ chmod +x sd-write.sh            # add x for everyone (minus the umask, below)
$ chmod u+x,go-w file             # add x for the owner, remove w for group and others
$ chmod 600 ~/.ssh/id_ed25519     # octal
$ chmod -R u+rwX,go+rX shared/    # recursive; capital X adds x only to directories and already-executable files
$ sudo chown user:user file       # owner and group
$ sudo chown -R user: ~/work      # owner, and group = the owner's primary group
$ chgrp builders shared/          # group only (you must be a member)
```

`chmod -R 755` on a tree makes every file executable; `chmod -R 777` is never the fix for a permission problem. Find which component denies access instead: `namei -l /path/to/file` lists the owner and mode of every directory on the way.

**The umask** removes bits from new files' modes. With the usual `022`, new files are `644` and new directories `755`; with `002`, the group can write too. `umask` prints yours.

**Special bits:**

| Bit | Shown as | On a file | On a directory | Example |
|---|---|---|---|---|
| setuid (4000) | `s` in the owner's `x` | Runs as the file's owner | (no effect on Linux) | `/usr/bin/passwd` runs as root to write `/etc/shadow` |
| setgid (2000) | `s` in the group's `x` | Runs with the file's group | New files inherit the directory's group | A shared project directory: `chmod 2775 shared/` |
| sticky (1000) | `t` in others' `x` | | Only a file's owner can delete it, even if everyone can write the directory | `/tmp` is `1777` |

**ACLs** give rights to more than one user or group when owner/group/others isn't enough:

```
$ setfacl -m u:alice:rw report.txt     # alice may read and write
$ setfacl -d -m g:builders:rwX shared/ # default ACL: inherited by new files in shared/
$ getfacl report.txt
```

A `+` after the mode in `ls -l` (`-rw-rw-r--+`) means the file has an ACL.

**Capabilities** split root's powers into pieces (`man 7 capabilities`), so a program can get one of them without being setuid root: `sudo setcap cap_net_raw+ep ./mytool` lets it open raw sockets, `getcap ./mytool` shows it.

**Other reasons for `Permission denied`** when the mode looks right:
- The filesystem is mounted `noexec` (a USB stick, sometimes `/tmp`): run the script with `bash script.sh`, or copy it elsewhere.
- The filesystem is FAT or exFAT (SD card boot partitions, USB sticks), which has no Linux owners or modes: the mount options decide for every file (section 17).
- You were added to the group after logging in (section 7).
- AppArmor, enabled on Debian and Ubuntu, confines some programs (CUPS, some browsers, snaps): `sudo journalctl -k | grep -i apparmor` shows refusals.

## 9. Pipes, redirection and text processing

Every process starts with three open files, numbered:

| Number | Name | Default |
|---|---|---|
| 0 | stdin, standard input | The keyboard |
| 1 | stdout, standard output | The terminal |
| 2 | stderr, standard error (messages, errors) | The terminal |

The shell can point each one somewhere else before the command starts; the command doesn't know or care.

**Redirection:**

| Syntax | Effect |
|---|---|
| `cmd > file` | stdout to `file`, replacing it |
| `cmd >> file` | stdout appended to `file` |
| `cmd 2> errors.txt` | stderr to a file |
| `cmd > out.txt 2>&1` | stdout to the file, then stderr to where stdout goes (order matters) |
| `cmd &> all.txt` | Both, in Bash |
| `cmd 2>/dev/null` | Discard errors; `/dev/null` swallows everything |
| `cmd < file` | stdin from a file |
| `cmd <<'EOF'` ... `EOF` | stdin from the following lines (a here-document) |
| `cmd <<< "text"` | stdin from a string |

`> file` empties the file before the command runs, so `sort data.txt > data.txt` leaves an empty file. Write to a new file and rename it, or use `sponge` (package `moreutils`).

**`sudo` and redirection.** `sudo echo "192.168.0.50 bbb" >> /etc/hosts` fails with `Permission denied`: the redirection is done by *your* shell before `sudo` starts. Let a root process open the file instead:

```
$ echo "192.168.0.50 bbb" | sudo tee -a /etc/hosts      # -a appends; without it, tee replaces
$ sudo tee /etc/modules-load.d/i2c.conf > /dev/null <<'EOF'
i2c-dev
EOF
```

**Pipes.** `cmd1 | cmd2` connects stdout of the first to stdin of the second; both run at the same time. Most text tools read stdin when no file is given, so they chain:

| Tool | Does | Example |
|---|---|---|
| `sort` | Sort lines; `-n` numeric, `-h` human sizes (`2K`, `1G`), `-r` reverse, `-k2` by field 2, `-u` unique | `du -sh * \| sort -h` |
| `uniq` | Collapse **adjacent** duplicate lines; `-c` counts. Sort first | `sort \| uniq -c \| sort -rn` |
| `cut` | Pick fields or columns | `cut -d: -f1,7 /etc/passwd` |
| `tr` | Translate or delete characters | `tr 'a-z' 'A-Z'`, `tr -d '\r'` |
| `wc` | Count lines, words, bytes | `ls \| wc -l` |
| `head`, `tail` | First, last lines | `... \| head -n 10` |
| `tee` | Copy stdin to a file **and** stdout | `make 2>&1 \| tee build.log` |
| `column -t` | Align columns | `mount \| column -t` |
| `sed` | Edit a stream (below) | |
| `awk` | Process fields, compute (below) | |
| `xargs` | Turn input lines into arguments | below |

**sed, the minimum:**

```
$ sed 's/old/new/' file           # replace the first match on each line, print the result
$ sed 's/old/new/g' file          # every match
$ sed -n '10,20p' file            # print lines 10 to 20 only
$ sed '/^#/d' file                # delete comment lines
$ sed -i.bak 's/^#\?PasswordAuthentication .*/PasswordAuthentication no/' sshd_config
                                  # edit in place, keeping sshd_config.bak
```

**awk, the minimum.** awk splits each line into fields `$1`, `$2` ... (on spaces by default, or on `-F`):

```
$ awk -F: '$7 ~ /bash$/ {print $1}' /etc/passwd     # users whose shell is bash
$ ls -l | awk '{sum += $5} END {print sum}'         # total size of the files listed
$ df -h | awk 'NR > 1 && $5+0 > 80'                 # filesystems over 80 % full (NR = line number)
```

**xargs** reads names and runs a command with them as arguments, avoiding `Argument list too long`:

```
$ find . -name '*.log' -print0 | xargs -0 gzip      # -print0 / -0: names separated by NUL, safe with spaces
$ cat hosts.txt | xargs -n 1 -P 4 ping -c 1         # one name per command, 4 in parallel
```

**Exit status and chaining.** Every command returns a number: 0 for success, anything else for failure. `$?` holds the last one.

| Syntax | Runs the second command |
|---|---|
| `a ; b` | Always |
| `a && b` | Only if `a` succeeded: `make && ./app` |
| `a \|\| b` | Only if `a` failed: `ping -c1 bbb \|\| echo "board down"` |

**Substitution:**
- `$(cmd)` is replaced by the command's output: `echo "Kernel $(uname -r)"`, `cd "$(dirname "$script")"`. The old form with backquotes does the same but doesn't nest.
- `<(cmd)` is replaced by a file name that reads the command's output: `diff <(ssh bbb cat /etc/fstab) fstab.new`.

**Worked examples:**

```
$ du -h --max-depth=1 ~ 2>/dev/null | sort -h | tail -n 10        # the 10 biggest directories in home
$ journalctl -b -p err --no-pager | awk '{sub(/\[.*/, "", $5); print $5}' | sort | uniq -c | sort -rn | head
                                                                    # which programs logged the most errors this boot
$ grep -rh '^CONFIG_' .config | cut -d= -f1 | sort > opts.txt      # the option names of a kernel config
$ history | awk '{print $2}' | sort | uniq -c | sort -rn | head    # your most used commands
```

## 10. Variables, the environment and shell startup files

**Shell variables** live in one shell; **environment variables** are passed to every program it starts:

```
$ BOARD=bbb                    # a shell variable: no spaces around =
$ echo "$BOARD"
bbb
$ export BOARD                 # now also in the environment of child processes
$ export CROSS=arm-linux-gnueabihf-    # set and export at once
$ printenv BOARD               # read one from the environment; printenv or env alone lists them all
$ unset BOARD
$ LANG=C ls /nonexistent       # set a variable for one command only
```

A child process gets a **copy**: a script can't change its parent's variables. That's why a settings file is *sourced* (`source ./settings.sh`, or `. ./settings.sh`), which runs it in the current shell, rather than executed.

**Variables you'll meet:**

| Variable | Holds |
|---|---|
| `PATH` | Directories searched for commands, separated by `:`, in order |
| `HOME`, `USER`, `SHELL` | Your home, name and login shell |
| `PWD` | The current directory |
| `EDITOR`, `VISUAL` | The editor other programs open |
| `LANG`, `LC_*` | Language and formats (section 22). `LC_ALL=C` gives English messages and byte sorting: use it when you paste errors into a search engine |
| `PS1` | The prompt |
| `TERM` | The terminal type, for colours and keys |

**PATH.** When you type `picocom`, the shell tries each `PATH` directory in turn and runs the first match; it never looks in the current directory, which is why a script there is run as `./build.sh`. To add a directory for your tools:

```
$ export PATH="$HOME/.local/bin:$PATH"       # first in the list: found before system programs
$ export PATH="$PATH:/opt/arm-gnu-toolchain/bin"   # last: system programs win on name clashes
$ hash -r                                    # forget remembered locations after installing or moving a program
```

Debian's default `~/.profile` already adds `~/bin` and `~/.local/bin` if they exist when you log in.

**Startup files.** Which ones Bash reads depends on how it starts:

| File | Read by | Put here |
|---|---|---|
| `/etc/profile`, `/etc/profile.d/*.sh` | Login shells, for all users | System-wide environment |
| `~/.profile` | Login shells (desktop login, SSH, `su -`) if there is no `~/.bash_profile` | Environment: `PATH`, `EDITOR`, `export`s |
| `~/.bash_profile` | Login shells, **instead of** `~/.profile` if it exists | Avoid creating it on Debian, or make it source `~/.profile` |
| `/etc/bash.bashrc` | Interactive shells, all users | |
| `~/.bashrc` | Interactive non-login shells (each new terminal tab); Debian's `~/.profile` also sources it | Aliases, functions, prompt, shell options |
| `/etc/environment` | PAM at login, for every program (not a shell script: `NAME=value` lines only) | Proxy settings, system-wide variables |
| `~/.config/environment.d/*.conf` | The systemd user manager: desktop apps and user services | Variables for graphical programs |

After editing `~/.bashrc`, `source ~/.bashrc` applies it to the current shell. Changes to `~/.profile` apply at the next login.

**Aliases and functions** go in `~/.bashrc`:

```bash
alias ll='ls -alF'
alias serial='picocom -b 115200 /dev/serial/by-id/usb-FTDI*'

# a function takes arguments; an alias doesn't
mkcd() { mkdir -p -- "$1" && cd -- "$1"; }
```

`\ls` or `command ls` runs the real command, bypassing an alias.

## 11. Small shell scripts

A script is a file of shell commands. When a task takes more than one command and you'll do it again, write it down as a script.

**The skeleton:**

```bash
#!/usr/bin/env bash
# What this script does, in one line.
set -euo pipefail

main() {
    echo "Hello from $0 with $# arguments: $*"
}

main "$@"
```

- The first line, the **shebang**, names the interpreter. Without it, or with `#!/bin/sh`, the script may run under `dash` (Debian's `/bin/sh`), which lacks Bash features such as `[[ ]]` and arrays.
- `chmod +x script.sh`, then `./script.sh`. Or `bash script.sh` without the `x` bit.
- `set -e` stops at the first failing command, `-u` treats an unset variable as an error, `-o pipefail` makes a pipe fail if any part fails. They catch many mistakes but have exceptions (a failing command inside `if` or before `||` doesn't stop the script); they don't replace checking what matters.
- `"$@"` passes the arguments on unchanged, each one quoted. `$1`, `$2` ... are the arguments, `$#` their number, `$0` the script's name.

**The building blocks:**

```bash
# condition: [[ ]] is Bash's test; spaces inside the brackets are required
if [[ -f "$file" && -r "$file" ]]; then
    echo "readable file"
elif [[ -d "$file" ]]; then
    echo "directory"
else
    echo "missing" >&2      # messages for the user go to stderr
    exit 1
fi

# loops
for img in images/*.img; do
    sha256sum "$img"
done

while read -r host; do
    ping -c 1 -W 1 "$host" > /dev/null && echo "$host up"
done < hosts.txt

# a command's success as a condition
if ! command -v bmaptool > /dev/null; then
    echo "install bmap-tools" >&2
    exit 1
fi

# default value for an argument
dev=${1:-/dev/sdX}

# clean up a temporary directory whatever happens
tmp=$(mktemp -d)
trap 'rm -rf "$tmp"' EXIT
```

| Test | True when |
|---|---|
| `-e path` / `-f` / `-d` / `-L` | It exists / is a regular file / a directory / a symbolic link |
| `-r` / `-w` / `-x` | You can read / write / run it |
| `-b path` | It is a block device (a disk) |
| `-z "$s"` / `-n "$s"` | The string is empty / not empty |
| `"$a" == "$b"`, `"$a" != "$b"` | Strings equal / different; `==` with an unquoted right side matches a pattern: `[[ $f == *.img ]]` |
| `$a -eq $b`, `-lt`, `-gt` ... | Numbers compared; or `(( a < b ))` |

**An example with safety checks** (**Illustrative**): write a board image to an SD card, refusing anything that isn't a removable, unmounted disk. Section 17 explains the commands.

```bash
#!/usr/bin/env bash
# sd-write.sh IMAGE DEVICE: write a disk image to an SD card, with checks.
set -euo pipefail

img=${1:?usage: $0 IMAGE DEVICE}
dev=${2:?usage: $0 IMAGE DEVICE}

[[ -f "$img" ]] || { echo "no such image: $img" >&2; exit 1; }
[[ -b "$dev" ]] || { echo "not a block device: $dev" >&2; exit 1; }
[[ "$(lsblk -dno TYPE "$dev")" == disk ]] || { echo "$dev is a partition, give the whole disk" >&2; exit 1; }

# Card readers usually report removable (RM=1); some built-in readers don't.
if [[ "$(lsblk -dno RM "$dev" | tr -d ' ')" != 1 ]]; then
    echo "$dev is not removable; refusing" >&2
    exit 1
fi

if lsblk -no MOUNTPOINTS "$dev" | grep -q .; then
    echo "$dev has mounted partitions; unmount them first:" >&2
    lsblk -o NAME,MOUNTPOINTS "$dev" >&2
    exit 1
fi

lsblk -o NAME,SIZE,MODEL,TRAN "$dev"
read -r -p "Write $img to $dev, erasing it? Type yes: " answer
[[ "$answer" == yes ]] || exit 1

case "$img" in
    *.xz) xzcat "$img" | sudo dd of="$dev" bs=4M conv=fsync status=progress ;;
    *.gz) zcat "$img" | sudo dd of="$dev" bs=4M conv=fsync status=progress ;;
    *)    sudo dd if="$img" of="$dev" bs=4M conv=fsync status=progress ;;
esac
sync
echo "done; remove the card"
```

**Check every script with ShellCheck** (package `shellcheck`): `shellcheck sd-write.sh` finds missing quotes, unused variables and the classic Bash traps, with a wiki page for each warning. When a script grows past a page, or needs data structures, error handling or parsing, Python is usually the better tool. The [Bash folder](../../software/bash/) is the place for the language in depth.

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
