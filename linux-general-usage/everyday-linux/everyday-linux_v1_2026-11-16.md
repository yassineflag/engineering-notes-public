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

A **process** is a running program: its code, memory, open files, user and groups, current directory and environment. Each has a number, the **PID**, and a parent, the **PPID**: your shell is the parent of the commands you run, and everything descends from PID 1, systemd.

**Looking at processes:**

```
$ ps aux                                    # every process, BSD style: user, PID, %CPU, %MEM, state, command
$ ps -ef                                    # the same, System V style, with PPID
$ ps -eo pid,ppid,user,%cpu,%mem,etime,cmd --sort=-%cpu | head    # pick columns, sort by CPU
$ pstree -p                                 # the family tree with PIDs
$ pgrep -a picocom                          # PIDs and command lines matching a name
$ top                                       # live view; htop (package htop) is friendlier
```

**Expected** (`ps aux`, a few lines):

```
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  0.1  23432 13900 ?        Ss   09:41   0:02 /sbin/init
user        2817  0.2  0.0  10248  5612 pts/1    Ss   10:02   0:00 -bash
user        3140 98.5  1.2 412000 98000 pts/1    R+   10:20   2:10 cc1 -quiet ...
user        3302  0.0  0.0   6740  2100 pts/2    S+   10:25   0:00 picocom -b 115200 /dev/ttyUSB0
```

`RSS` is the memory really in use, in KiB. `STAT` is the state: `R` running, `S` sleeping (waiting for something, the normal state), `D` uninterruptible sleep (waiting for a disk or a device; can't be killed until it returns), `T` stopped, `Z` zombie (finished, waiting for its parent to collect its exit status). `+` means in the foreground of a terminal, `s` a session leader.

In `top` and `htop`: `P` sorts by CPU, `M` by memory, `k` sends a signal, `q` quits; in `htop`, `F4` filters and `F5` shows the tree.

**Load and memory.** `uptime` shows the **load average** over 1, 5 and 15 minutes: the average number of processes running or waiting for a CPU (or in `D` state). Compare it to `nproc`: a load of 8 on an 8-core machine is full, on a 2-core machine it's overloaded.

```
$ free -h
```

**Expected:**

```
               total        used        free      shared  buff/cache   available
Mem:            31Gi       9.2Gi       1.1Gi       410Mi        21Gi        21Gi
Swap:          2.0Gi          0B       2.0Gi
```

Read **available**, not **free**: the kernel uses spare memory as a disk cache (`buff/cache`) and gives it back when programs need it. When even that runs out, the kernel's **OOM killer** kills the process using the most memory; `journalctl -k | grep -i 'out of memory'` shows it. A parallel build (`make -j$(nproc)`) of a big C++ project can need 1 to 2 GB per job; lower `-j` if the OOM killer strikes.

**Signals** are short messages the kernel delivers to a process (`man 7 signal`):

| Signal | Number | Sent by | Default effect |
|---|---|---|---|
| `SIGTERM` | 15 | `kill PID` (the default), `systemctl stop` | Ask to terminate; the program can clean up first |
| `SIGINT` | 2 | `Ctrl+C` | Interrupt |
| `SIGKILL` | 9 | `kill -9 PID` | Kill at once; can't be caught, no cleanup |
| `SIGHUP` | 1 | Closing the terminal | Terminate; many daemons reload their configuration on it instead |
| `SIGTSTP` | 20 | `Ctrl+Z` | Stop (suspend) |
| `SIGSTOP` / `SIGCONT` | 19 / 18 | `kill -STOP` / `kill -CONT` | Stop / continue; `SIGSTOP` can't be caught |

```
$ kill 3302                  # polite: SIGTERM
$ kill -9 3302               # only if SIGTERM didn't work after a few seconds
$ pkill -f 'http.server'     # by pattern on the full command line (check first with pgrep -af)
$ killall picocom            # by exact program name
```

`kill -9` first is a bad habit: the program can't flush files, remove lock files or restore the terminal. A process in `D` state or a zombie ignores even `SIGKILL`; a zombie disappears when its parent collects it or exits.

**Jobs: foreground and background.** A command normally runs in the **foreground**: the shell waits for it.

| Action | Command |
|---|---|
| Start in the background | `long-command &` |
| Suspend the foreground job | `Ctrl+Z` |
| List the jobs of this shell | `jobs` |
| Resume a job in the background / foreground | `bg %1` / `fg %1` |
| Keep a job running after the shell exits | `disown %1`, or start it with `nohup cmd > out.log 2>&1 &` |

For anything long (a build, a download, a serial log) over SSH, `tmux` (section 20) is better than `nohup`: you can reattach and see the output. For a one-off command that should run as a proper service, `systemd-run --user --unit=mylog cmd` starts it under systemd, with its output in the journal.

**Priority.** `nice -n 10 make -j8` starts a command with a lower CPU priority (19 is the lowest); `renice -n 10 -p PID` changes a running one. `ionice -c3 cmd` gives it disk access only when no one else needs it. Both are useful to keep a desktop responsive during a long build.

**Open files and busy resources:**

```
$ ls -l /proc/3302/fd          # the files a process has open (/proc/PID/ holds everything about it)
$ sudo lsof /dev/ttyUSB0       # who has the serial port open
$ sudo lsof -i :8000           # who uses TCP or UDP port 8000
$ sudo fuser -vm /media/user/BOOT   # who is using a mounted filesystem (section 17)
```

## 13. Services with systemd

**systemd** is PID 1 on Debian, Ubuntu, Fedora and Arch. It starts the system, keeps services running, and handles logins, logs, timers, mounts and devices. Everything it manages is a **unit**, described by a small text file:

| Unit type | Describes | Example |
|---|---|---|
| `.service` | A program to run and keep running | `ssh.service`, `NetworkManager.service` |
| `.timer` | When to start a service: like cron | `apt-daily.timer` |
| `.socket` | A socket that starts its service on the first connection | `cups.socket` |
| `.target` | A group of units, a state to reach | `multi-user.target`, `graphical.target` |
| `.mount` | A mounted filesystem, often generated from `/etc/fstab` | `home.mount` |
| `.device` | A device the kernel announced | `dev-ttyUSB0.device` |

**Everyday commands:**

| Task | Command |
|---|---|
| State, recent log lines, PID, memory | `systemctl status ssh` |
| Start / stop / restart now | `sudo systemctl start ssh` / `stop` / `restart` |
| Reload the configuration without restarting (if the service supports it) | `sudo systemctl reload ssh` |
| Start at every boot / stop doing so | `sudo systemctl enable ssh` / `disable` |
| Both at once: enable and start | `sudo systemctl enable --now ssh` |
| Make a unit impossible to start, even by a dependency | `sudo systemctl mask ModemManager` (`unmask` to undo) |
| What has failed | `systemctl --failed` |
| All services and their state | `systemctl list-units --type=service` |
| All unit files and whether they're enabled | `systemctl list-unit-files` |
| Show a unit's file and its overrides | `systemctl cat ssh` |
| Reread unit files after editing them | `sudo systemctl daemon-reload` |

**Expected** (`systemctl status ssh`):

```
● ssh.service - OpenBSD Secure Shell server
     Loaded: loaded (/usr/lib/systemd/system/ssh.service; enabled; preset: enabled)
     Active: active (running) since Sun 2026-11-16 09:41:12 CET; 1h 2min ago
 Invocation: 5f0c...
       Docs: man:sshd(8)
             man:sshd_config(5)
   Main PID: 812 (sshd)
      Tasks: 1 (limit: 38123)
     Memory: 4.1M
        CPU: 52ms
     CGroup: /system.slice/ssh.service
             └─812 "sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups"

Nov 16 09:41:12 pc systemd[1]: Starting ssh.service - OpenBSD Secure Shell server...
Nov 16 09:41:12 pc sshd[812]: Server listening on 0.0.0.0 port 22.
Nov 16 09:41:12 pc systemd[1]: Started ssh.service - OpenBSD Secure Shell server.
```

**Loaded** says where the unit file is and whether it's **enabled** (started at boot); **Active** whether it runs **now**. They are independent: a service can be enabled but stopped, or running but disabled.

**Where unit files live:**

| Directory | Contains | Edit? |
|---|---|---|
| `/usr/lib/systemd/system/` | Units installed by packages | No: an upgrade overwrites them |
| `/etc/systemd/system/` | Your units, and overrides | Yes |
| `/etc/systemd/system/<unit>.d/*.conf` | Drop-ins: a few lines that change a packaged unit | Yes, with `systemctl edit` |
| `~/.config/systemd/user/` | Your user units (`systemctl --user`) | Yes |

To change a packaged service, `sudo systemctl edit ssh` opens an empty drop-in; what you write there overrides the matching settings of the original file, and the drop-in survives upgrades. `sudo systemctl edit --full ssh` copies the whole file to `/etc` instead.

**Writing a service.** A unit that serves the build output directory over HTTP, so a board can fetch images with `wget` (**Illustrative**):

```ini
# /etc/systemd/system/image-server.service
[Unit]
Description=HTTP server for board images
After=network-online.target
Wants=network-online.target

[Service]
User=user
ExecStart=/usr/bin/python3 -m http.server 8000 --directory /srv/images
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```
$ sudo systemctl daemon-reload
$ sudo systemctl enable --now image-server
$ systemctl status image-server
$ journalctl -u image-server -f        # its output goes to the journal (section 14)
```

- `ExecStart=` needs an absolute path, and isn't run by a shell: no `|`, `>`, `&&` or `~`. For those, point it at a script, or use `/bin/sh -c '...'`.
- The service runs with an almost empty environment and in `/`, not your home. A command that works in your terminal and fails as a service is usually missing a variable (`Environment=NAME=value`) or a working directory (`WorkingDirectory=`).
- `Restart=on-failure` restarts it if it crashes; `Type=` defaults to `simple`, right for a program that stays in the foreground.
- `WantedBy=multi-user.target` is what `enable` hooks it into: the normal boot.

**Timers** replace cron jobs. A timer that runs a backup service every night, and catches up after the PC was off (**Illustrative**; the backup script is in section 21):

```ini
# /etc/systemd/system/backup.service
[Unit]
Description=Back up /home to the external disk

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/backup-home.sh
```

```ini
# /etc/systemd/system/backup.timer
[Unit]
Description=Nightly backup

[Timer]
OnCalendar=*-*-* 02:30
Persistent=true

[Install]
WantedBy=timers.target
```

```
$ sudo systemctl enable --now backup.timer     # enable the timer, not the service
$ systemctl list-timers                        # next and last run of every timer
$ systemd-analyze calendar '*-*-* 02:30'       # check what an OnCalendar expression means
$ sudo systemctl start backup.service          # run it once now, to test
```

`crontab -e` still works if the `cron` package is installed; timers have the advantage of logging to the journal and of `Persistent=true`.

**User services** run as you, without `sudo`, while you're logged in: put the unit in `~/.config/systemd/user/` and use `systemctl --user enable --now unit`. `sudo loginctl enable-linger user` keeps them running when you're logged out.

**Boot, targets and power:**

```
$ systemctl get-default                        # graphical.target on a desktop
$ sudo systemctl set-default multi-user.target # boot to a text console next time
$ systemd-analyze blame | head                 # which units took longest at boot
$ systemctl reboot                             # or poweroff, suspend
```

## 14. Logs: the journal, /var/log and dmesg

Two sources hold most of what you need when something fails: the **journal** of `systemd-journald`, which collects the output of every service, the kernel's messages and what programs send to syslog; and a few classic text files under `/var/log`.

**journalctl:**

| Command | Shows |
|---|---|
| `journalctl -b` | Everything since this boot |
| `journalctl -b -1` | The previous boot (after a crash or a bad shutdown); `--list-boots` lists them |
| `journalctl -u ssh` | One unit; repeat `-u` for several |
| `journalctl -f` | Follow new messages, like `tail -f` |
| `journalctl -e` | Jump to the end |
| `journalctl -p err -b` | Only errors and worse this boot (`emerg`, `alert`, `crit`, `err`, `warning`, `notice`, `info`, `debug`) |
| `journalctl -k` | Kernel messages only (like `dmesg`) |
| `journalctl --since "1 hour ago"`, `--since 09:00 --until 09:30` | A time window |
| `journalctl _COMM=sshd` | By program name |
| `journalctl -o short-iso` / `-o cat` | Full timestamps / message text only |
| `journalctl --disk-usage` | Space used by the journal |
| `sudo journalctl --vacuum-size=500M` | Delete old entries down to 500 MB |

Members of `adm` or `systemd-journal` (section 7) see every message; others see only their own user's. Since Debian 12, the journal is kept on disk in `/var/log/journal/` across reboots and `rsyslog` isn't installed by default, so `/var/log/syslog` may not exist: use `journalctl`. To limit its size permanently, set `SystemMaxUse=500M` in a drop-in such as `/etc/systemd/journald.conf.d/size.conf`, then `sudo systemctl restart systemd-journald`.

**Text logs still under `/var/log`:**

| File | Holds |
|---|---|
| `/var/log/apt/history.log`, `/var/log/dpkg.log` | What was installed, upgraded and removed, and when (section 15) |
| `/var/log/apt/term.log` | The full output of every APT run |
| `/var/log/unattended-upgrades/` | Automatic upgrades |
| `/var/log/Xorg.0.log`, `~/.local/share/xorg/` | X11 graphics, on desktops that still use it |
| Application logs | Web servers, databases: `/var/log/<name>/` |

`logrotate` compresses and deletes old text logs (`.1`, `.2.gz` ...) on a timer; its rules are in `/etc/logrotate.d/`.

**Kernel messages: `dmesg`.** The kernel's ring buffer holds driver messages: devices plugged in, disk errors, USB problems, the OOM killer. On Debian, reading it needs root:

```
$ sudo dmesg -T | tail -n 30     # -T: human-readable times
$ sudo dmesg -w                  # follow: run it, then plug the device in
$ journalctl -k -f               # the same through the journal, without sudo if you're in adm
```

**Expected** (plugging in an FTDI USB serial adapter):

```
[Sun Nov 16 10:31:02 2026] usb 1-2: new full-speed USB device number 7 using xhci_hcd
[Sun Nov 16 10:31:02 2026] usb 1-2: New USB device found, idVendor=0403, idProduct=6001, bcdDevice= 6.00
[Sun Nov 16 10:31:02 2026] usb 1-2: Product: FT232R USB UART
[Sun Nov 16 10:31:02 2026] usb 1-2: SerialNumber: A10KXXXX
[Sun Nov 16 10:31:02 2026] ftdi_sio 1-2:1.0: FTDI USB Serial Device converter detected
[Sun Nov 16 10:31:02 2026] usb 1-2: FTDI USB Serial Device converter now attached to ttyUSB0
```

This is the first thing to check when a device "doesn't appear": whether the kernel saw it, with which IDs, which driver took it, and which `/dev` name it got (section 18).

**Writing to the log** from your own scripts: `logger -t backup "finished, 12 GB"` sends a message to the journal, and `journalctl -t backup` finds it. A systemd service needs nothing: its stdout and stderr already go to the journal.

## 15. Packages with APT and dpkg

A **package** is an archive of files plus a description: its name, version, dependencies and scripts to run on install and removal. Debian packages are `.deb` files. **dpkg** installs a `.deb` and records it; **APT** sits on top: it downloads packages from the **repositories** you configured, verifies their signatures and resolves dependencies.

**Everyday commands:**

| Task | Command |
|---|---|
| Refresh the list of available packages | `sudo apt update` |
| Upgrade everything that can be, without removing anything | `sudo apt upgrade` |
| Upgrade, removing or adding packages if needed (point releases, kernel changes) | `sudo apt full-upgrade` |
| Install | `sudo apt install picocom git build-essential` |
| Install a `.deb` file, with its dependencies | `sudo apt install ./vendor-tool_1.2_amd64.deb` (the `./` matters) |
| Remove / remove with its system configuration | `sudo apt remove pkg` / `sudo apt purge pkg` |
| Remove dependencies nothing needs any more | `sudo apt autoremove` |
| Search names and descriptions | `apt search serial terminal` |
| Describe a package | `apt show picocom` |
| Which version would be installed, and from where | `apt policy picocom` |
| What's installed / upgradable | `apt list --installed`, `apt list --upgradable` |
| Files of an installed package | `dpkg -L picocom` |
| Which package owns a file | `dpkg -S /usr/bin/picocom` |
| Installed packages matching a pattern | `dpkg -l 'linux-image*'` |
| What changed recently | `less /var/log/apt/history.log` |

`apt` is meant for people: progress bars, colours, output that may change between versions. In scripts use `apt-get` and `apt-cache`, whose output is stable: `sudo apt-get install -y pkg`.

**Repositories** are configured in `/etc/apt/sources.list.d/`. Debian 13 uses the **deb822** format, one `.sources` file per repository (**Expected**, for a standard installation):

```
# /etc/apt/sources.list.d/debian.sources
Types: deb
URIs: http://deb.debian.org/debian
Suites: trixie trixie-updates
Components: main contrib non-free non-free-firmware
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg

Types: deb
URIs: http://security.debian.org/debian-security
Suites: trixie-security
Components: main contrib non-free non-free-firmware
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg
```

The older one-line format in `/etc/apt/sources.list` (`deb http://deb.debian.org/debian trixie main`) still works; `sudo apt modernize-sources` converts it. **Components:** `main` is free software, `non-free-firmware` holds firmware for Wi-Fi cards and GPUs, `contrib` and `non-free` hold packages that depend on or are non-free software.

**Third-party repositories** (Docker, VS Code, a vendor's tools): put the vendor's signing key in `/etc/apt/keyrings/` and name it in `Signed-By:`, so it's trusted only for that repository (**Illustrative**):

```
$ curl -fsSL https://example.com/repo/key.asc | sudo gpg --dearmor -o /etc/apt/keyrings/example.gpg
$ sudo tee /etc/apt/sources.list.d/example.sources > /dev/null <<'EOF'
Types: deb
URIs: https://example.com/repo/debian
Suites: trixie
Components: main
Signed-By: /etc/apt/keyrings/example.gpg
EOF
$ sudo apt update
```

`apt-key`, which old instructions still mention, is gone from APT 3.0; and a key trusted for every repository could sign fake Debian packages. Every third-party repository can replace any package on your system: add as few as possible.

**Newer versions without leaving Debian stable.** **Backports** are packages from the next release rebuilt for this one. Add the suite `trixie-backports` to a `.sources` file, then ask for it explicitly; nothing is taken from backports otherwise:

```
$ sudo apt install -t trixie-backports linux-image-amd64
```

**Holding a package** at its version: `sudo apt-mark hold linux-image-amd64`, `apt-mark showhold`, `sudo apt-mark unhold ...`. **Automatic security updates:** `sudo apt install unattended-upgrades`, then `sudo dpkg-reconfigure unattended-upgrades`. After upgrades, `needrestart` (installed by default on Ubuntu) lists the services still running old libraries.

**Fixing a stuck package system:**

| Message | Meaning and fix |
|---|---|
| `Could not get lock /var/lib/dpkg/lock-frontend` | Another APT is running: often `unattended-upgrades` or a software centre. Wait; `ps aux \| grep -E 'apt\|dpkg'` shows it. Never delete the lock files |
| `dpkg was interrupted, you must manually run 'sudo dpkg --configure -a'` | An install was cut off: run exactly that |
| `Unmet dependencies` / `Try 'apt --fix-broken install'` | `sudo apt --fix-broken install` |
| `The following packages have been kept back` | They need new dependencies or removals: `sudo apt full-upgrade`, after reading what it will remove |
| `NO_PUBKEY ...` or `is not signed` | A third-party key is missing or expired: fetch the vendor's current key as above |
| `Release file ... is not valid yet` | The PC's clock is wrong (section 22) |

**Other distributions,** for when you're on someone else's machine:

| Task | Debian, Ubuntu | Fedora | Arch |
|---|---|---|---|
| Refresh and upgrade | `apt update && apt full-upgrade` | `dnf upgrade` | `pacman -Syu` |
| Install / remove | `apt install` / `apt remove` | `dnf install` / `dnf remove` | `pacman -S` / `pacman -R` |
| Search | `apt search` | `dnf search` | `pacman -Ss` |
| Owner of a file | `dpkg -S` | `rpm -qf` | `pacman -Qo` |
| Files of a package | `dpkg -L` | `rpm -ql` | `pacman -Ql` |

## 16. Software outside APT

Not everything is packaged, or packaged in the version you need. In rough order of preference:

| Source | Use for | Where it goes | Updated by |
|---|---|---|---|
| Debian package | Everything that's there | `/usr` | `apt` |
| Backports | A newer version of a few packages | `/usr` | `apt` |
| Vendor APT repository | Tools whose vendor maintains one (Docker, VS Code, KiCad PPA on Ubuntu) | `/usr` or `/opt` | `apt` |
| Flatpak | Desktop applications (`flatpak install flathub org.kicad.KiCad`) | `/var/lib/flatpak`, sandboxed | `flatpak update` |
| Snap (Ubuntu) | Desktop and some command-line apps; Ubuntu installs Firefox this way | `/snap`, sandboxed | Automatic |
| Language tools: `pipx`, `cargo`, `npm` | Command-line tools written in Python, Rust, JavaScript | `~/.local`, `~/.cargo` | The tool |
| A vendor tarball | Toolchains, SDKs, IDEs (the Arm GNU Toolchain of the embedded notes) | `/opt/<name>` | You, by hand |
| Built from source | Anything else | `/usr/local` or `~/.local` | You, by hand |
| AppImage | Single-file desktop apps | `~/Applications` | You, or the app |

**Python.** Debian protects its own Python packages: `pip install --user something` fails with `error: externally-managed-environment`. Don't force it with `--break-system-packages`. Instead:

```
$ sudo apt install python3-serial                # if Debian packages the library
$ sudo apt install pipx && pipx install esptool  # a command-line tool, in its own environment, on PATH
$ python3 -m venv .venv                          # a project: its own environment in the project directory
$ . .venv/bin/activate
(.venv) $ pip install -r requirements.txt
(.venv) $ deactivate
```

`uv` (`pipx install uv`) does the same as `venv` and `pip`, much faster.

**Building from source** usually follows one of these, after installing the build dependencies (`sudo apt install build-essential cmake pkg-config`, plus the `-dev` packages the README lists; `sudo apt build-dep <package>` installs all of them for a program Debian also packages, if `deb-src` lines are enabled):

```
$ ./configure --prefix=/usr/local && make -j"$(nproc)" && sudo make install          # Autotools
$ cmake -B build -DCMAKE_INSTALL_PREFIX=/usr/local && cmake --build build -j && sudo cmake --install build
$ meson setup build --prefix=/usr/local && ninja -C build && sudo ninja -C build install
```

- `/usr/local` is yours: `apt` never writes there, and `/usr/local/bin` comes before `/usr/bin` in `PATH`, so your build wins over the packaged one. That's the point, and also a trap when you forget it's there: `type -a tool` shows both.
- With `--prefix="$HOME/.local"`, no `sudo` is needed at all.
- There is no uninstall: keep the build directory (`sudo make uninstall` works only if the project provides it; CMake writes `build/install_manifest.txt`, the list of installed files).
- New shared libraries in `/usr/local/lib` need `sudo ldconfig` once, or programs fail with `error while loading shared libraries`.

**Several versions of one tool.** Debian's `update-alternatives` switches what a generic name points to (`editor`, `java`, `cc`, `x-terminal-emulator`): `update-alternatives --display editor`. For a toolchain in `/opt`, put its `bin` directory in `PATH` only in the shells that need it, through a settings file you source (section 10), rather than globally.

**Containers for build environments.** When a tool needs an older or different distribution (a vendor SDK that wants Ubuntu 22.04, a Yocto release that doesn't support your host), run it in a container instead of changing the PC: `podman run --rm -it -v "$PWD":/work -w /work ubuntu:22.04 bash`. Podman runs rootless (no daemon, no root-equivalent group); Docker is the same idea with a daemon. The toolchain and Buildroot notes of the [embedded Linux notes](../../embedded-linux/) suggest the same for build hosts.

## 17. Disks, partitions, filesystems and SD cards

**Names.** Disks appear as block devices in `/dev`, and their partitions as numbered devices:

| Device | What | Partitions |
|---|---|---|
| `/dev/sda`, `/dev/sdb` ... | SATA and USB disks, USB sticks, most USB card readers | `/dev/sda1`, `/dev/sda2` |
| `/dev/nvme0n1` | NVMe SSD | `/dev/nvme0n1p1` |
| `/dev/mmcblk0` | SD card in a built-in reader (and the eMMC or SD card on a board) | `/dev/mmcblk0p1` |
| `/dev/loop0` | A file used as a disk (images, below) | `/dev/loop0p1` |

**The letters are not stable**: the USB stick that is `sdb` today can be `sdc` tomorrow, and the order changes with what's plugged in. Identify a disk by its size, model and contents every time, and in configuration use UUIDs or labels, never `sdX`.

**Seeing what's there:**

```
$ lsblk -o NAME,SIZE,TYPE,FSTYPE,LABEL,MOUNTPOINTS,MODEL,TRAN,RM
```

**Expected** (with a 32 GB SD card in a USB reader):

```
NAME          SIZE TYPE FSTYPE LABEL  MOUNTPOINTS          MODEL         TRAN  RM
sda          29.7G disk                                    SD Card Reader usb    1
├─sda1         64M part vfat   BOOT   /media/user/BOOT
└─sda2       29.6G part ext4   rootfs /media/user/rootfs
nvme0n1     476.9G disk                                    Samsung SSD    nvme   0
├─nvme0n1p1   512M part vfat          /boot/efi
├─nvme0n1p2 475.4G part ext4          /
└─nvme0n1p3   977M part swap          [SWAP]
```

| Command | Shows |
|---|---|
| `lsblk -f` | Filesystems, labels, UUIDs and use, as a tree |
| `sudo blkid` | UUID, label and type of every filesystem |
| `df -h` | Size and free space of every mounted filesystem |
| `df -i` | Free inodes (a filesystem can be "full" with free space, if it holds millions of small files) |
| `du -sh dir` | Space used by a directory |
| `du -h --max-depth=1 / 2>/dev/null \| sort -h` | Which top-level directory is big; `ncdu` (package `ncdu`) is the interactive version |
| `findmnt` | Every mount, as a tree, with options |
| `sudo fdisk -l /dev/sda` | The partition table |

**Mounting** attaches a filesystem to a directory, the **mount point**; its files then appear there. On a desktop, removable media are mounted for you under `/media/user/`. By hand:

```
$ sudo mount /dev/sda1 /mnt                 # the type is detected
$ ls /mnt
$ sudo umount /mnt                          # umount, not unmount; the device or the directory
$ udisksctl mount -b /dev/sda1              # as your user, like the desktop does (mounts under /media/user)
$ udisksctl power-off -b /dev/sda           # unmount everything and power the device off before unplugging
```

`umount: /mnt: target is busy` means a process has a file open there, or a shell has its current directory there (yours included): `sudo fuser -vm /mnt` or `sudo lsof +f -- /mnt` shows which one.

**Permanent mounts** go in `/etc/fstab`, one line per filesystem, by UUID (**Illustrative**, for a data disk):

```
# <device>                                  <mount point>  <type>  <options>                  <dump> <pass>
UUID=3f0e6a2c-xxxx-xxxx-xxxx-xxxxxxxxxxxx   /data          ext4    defaults,nofail            0      2
```

- `nofail` lets the PC boot even if the disk is missing. Without it, a missing disk in `fstab` stops the boot in an emergency shell.
- Test before rebooting: `sudo findmnt --verify`, then `sudo systemctl daemon-reload && sudo mount -a`. No error and the disk mounted means the line is right.
- For a FAT or exFAT disk, add `uid=1000,gid=1000,umask=022` (or `fmask`/`dmask`): FAT has no owners, so the options decide them (section 8).

**Partitioning and formatting** destroys what's on the disk; check the device name with `lsblk` twice:

```
$ sudo fdisk /dev/sdX            # interactive: p print, g new GPT table, o new MBR table, n new, t type, w write, q quit
$ sudo parted /dev/sdX print     # parted and gparted (graphical) do the same
$ sudo mkfs.ext4 -L data /dev/sdX1
$ sudo mkfs.vfat -F 32 -n BOOT /dev/sdX1      # FAT32, needed for many boot partitions (package dosfstools)
$ sudo mkfs.exfat -n STICK /dev/sdX1          # for sticks shared with Windows and macOS (package exfatprogs)
```

| Filesystem | Use |
|---|---|
| ext4 | Linux system and data disks; the default |
| btrfs, xfs | Linux, with snapshots (btrfs) or for large files (xfs) |
| vfat (FAT32) | Boot partitions (UEFI, the BeagleBone's and Raspberry Pi's first partition), small sticks; 4 GiB file limit, no permissions |
| exFAT | Sticks and cards shared with other systems; no file limit, no permissions |
| NTFS | Windows disks: read and write with the `ntfs3` kernel driver |

**Writing an image to an SD card.** Board images (`.img`, often compressed as `.img.xz`) are a byte-for-byte copy of a whole card, partition table included. They're written to the **whole disk** (`/dev/sdX`), not to a partition (`/dev/sdX1`):

1. Run `lsblk` without the card, insert it, run `lsblk` again: the new disk is the card. Check its size.
2. Unmount its partitions if the desktop mounted them: `udisksctl unmount -b /dev/sdX1` (and each other partition).
3. Write:

```
$ xzcat bbb-image.img.xz | sudo dd of=/dev/sdX bs=4M conv=fsync status=progress
$ sudo dd if=sdcard.img of=/dev/sdX bs=4M conv=fsync status=progress     # uncompressed
$ sudo bmaptool copy bbb-image.img.xz /dev/sdX     # faster when a .bmap file exists: skips empty blocks (package bmap-tools)
$ sync
```

4. `udisksctl power-off -b /dev/sdX`, then remove the card.

`dd` writes wherever `of=` points, without asking, including your system disk: that's why it's nicknamed "disk destroyer". The script in section 11 adds checks. Graphical tools (GNOME Disks' "Restore Disk Image", balenaEtcher, Raspberry Pi Imager) only offer removable disks.

**Images as disks: loop devices.** To look inside an image without writing it to a card:

```
$ sudo losetup -fP --show sdcard.img        # attach; -P creates a device per partition; prints /dev/loop0
$ lsblk /dev/loop0
$ sudo mount /dev/loop0p2 /mnt
$ ls /mnt/etc
$ sudo umount /mnt && sudo losetup -d /dev/loop0
```

`udisksctl loop-setup -f sdcard.img` does the same as your user, and the desktop then mounts the partitions.

**Checking and swap:**
- `sudo fsck /dev/sdX2` checks and repairs a filesystem. **Only on an unmounted filesystem**: running it on a mounted one corrupts it. Cards pulled out without unmounting are the usual reason to need it.
- `swapon --show` lists swap; Debian creates a swap partition by default. A swap file: `sudo fallocate -l 4G /swapfile && sudo chmod 600 /swapfile && sudo mkswap /swapfile && sudo swapon /swapfile`, plus `/swapfile none swap sw 0 0` in `fstab`.
- `sudo smartctl -a /dev/nvme0n1` (package `smartmontools`) shows a disk's health and wear.

## 18. Hardware, devices and the serial console

**What hardware is there:**

| Command | Shows |
|---|---|
| `lscpu`, `nproc` | CPU model, cores, threads, flags; the number of usable CPUs |
| `free -h` | Memory (section 12) |
| `lsusb` | USB devices with vendor:product IDs; `lsusb -t` as a tree with drivers and speeds |
| `lspci -k` | PCI devices (network, graphics, Wi-Fi) and the driver each one uses |
| `lsblk` | Disks (section 17) |
| `ip -br link` | Network interfaces (section 19) |
| `sensors` | Temperatures and fans (package `lm-sensors`) |
| `sudo dmidecode -t system` | Maker, model and serial number of the PC |
| `inxi -Fz` | A summary of everything (package `inxi`; `-z` hides serials and addresses for posting in a forum) |
| `lsmod`, `modinfo ftdi_sio` | Loaded kernel modules; information on one |
| `sudo modprobe usbserial`, `sudo modprobe -r ...` | Load / unload a module and its dependencies |

**From device to `/dev` name: udev.** When the kernel finds a device, it announces it; **udev** (`systemd-udevd`) receives the announcement, loads the driver if needed, creates the `/dev` file with the right owner, group and mode, and adds stable names. Rules in `/usr/lib/udev/rules.d/` (from packages) and `/etc/udev/rules.d/` (yours) decide.

```
$ udevadm monitor --udev --property     # watch events while plugging a device in; Ctrl+C to stop
$ udevadm info /dev/ttyUSB0             # the properties of one device
$ udevadm info -a -n /dev/ttyUSB0       # the attributes of it and its parents, to write rules with
```

**Stable names.** `/dev/ttyUSB0` and `/dev/ttyUSB1` are numbered in the order adapters appear. udev also creates names that don't change:

```
$ ls -l /dev/serial/by-id/
```

**Expected:**

```
lrwxrwxrwx 1 root root 13 Nov 16 10:31 usb-FTDI_FT232R_USB_UART_A10KXXXX-if00-port0 -> ../../ttyUSB0
```

Use that path in scripts and aliases (section 10). `/dev/disk/by-id/` and `/dev/disk/by-uuid/` do the same for disks.

**Your own udev rule.** A rule that gives a board's adapter a short name and the right group (**Illustrative**; take the IDs and serial number from `udevadm info -a`):

```
# /etc/udev/rules.d/99-bbb-console.rules
SUBSYSTEM=="tty", ATTRS{idVendor}=="0403", ATTRS{idProduct}=="6001", ATTRS{serial}=="A10KXXXX", \
    SYMLINK+="bbb-console", GROUP="dialout", MODE="0660", ENV{ID_MM_DEVICE_IGNORE}="1"
```

```
$ sudo udevadm control --reload
$ sudo udevadm trigger --subsystem-match=tty     # or unplug and replug
$ ls -l /dev/bbb-console
```

`==` compares, `=` and `+=` assign. `ATTRS` (plural) matches the device or any of its parents, which is where USB IDs live; all the `ATTRS` of one rule must come from the same parent. `ID_MM_DEVICE_IGNORE` tells ModemManager to leave the port alone (below). The same kind of rule, with `MODE="0660", GROUP="plugdev"`, gives you access to debug probes, SDRs and programmers whose packages don't ship a rule.

**The serial console.** A board's console is a UART; a USB serial adapter makes it a `/dev/ttyUSB*` (FTDI, CP210x, CH340, PL2303) or `/dev/ttyACM*` (CDC ACM devices: many development boards with built-in debuggers). To talk to it you need to be in `dialout` (section 7), and a terminal program:

```
$ picocom -b 115200 /dev/ttyUSB0                       # exit: Ctrl+A then Ctrl+X
$ picocom -b 115200 --logfile bbb-boot.log /dev/bbb-console
$ tio /dev/ttyUSB0                                     # 115200 by default; exit Ctrl+T then Q; reconnects automatically
$ screen /dev/ttyUSB0 115200                           # exit: Ctrl+A then K, then y
$ minicom -D /dev/ttyUSB0 -b 115200                    # exit: Ctrl+A then X; turn off hardware flow control in its setup (Ctrl+A O)
```

Settings for almost every board console, including the BeagleBone Black's: **115200 baud, 8 data bits, no parity, 1 stop bit (8N1), no flow control**. Wiring: the adapter's TX goes to the board's RX and RX to TX, plus ground; the voltage must match (3.3 V for the BeagleBone Black and Raspberry Pi; a 5 V adapter can damage them).

**When the serial port misbehaves:**

| Symptom | Likely cause | Fix |
|---|---|---|
| `/dev/ttyUSB0` doesn't exist | Driver not loaded, cable charge-only, adapter dead | `sudo dmesg -w` while plugging in (section 14); try another cable |
| `Permission denied` | Not in `dialout`, or not logged in again since | `id`; section 7 |
| `Device or resource busy` | Another program has it open: a second terminal, ModemManager probing it | `sudo lsof /dev/ttyUSB0`; the udev rule above, or `sudo systemctl mask ModemManager` if you have no modem |
| The device appears, then disappears a second later (Ubuntu, CH340 adapters) | `brltty`, the Braille display service, claims the adapter's IDs | `sudo apt remove brltty` if you don't use a Braille display |
| Garbage characters | Wrong baud rate | 115200 for most boards; some boot ROMs print at other rates |
| Nothing at all | TX and RX not crossed, no ground, board not powered, wrong voltage | Check wiring; type `Enter` once the board has booted |
| Output but typing does nothing | Board's RX not connected, or hardware flow control on | Check the wire on the board's RX; disable flow control |

## 19. The network from the user's side

## 20. SSH, file transfer and tmux

## 21. Archives, checksums and backups

## 22. Housekeeping: time, names, updates and disk space

## 23. Troubleshooting by symptom

## 24. Not covered here

## References

---

Version 1 · 2026-11-16
