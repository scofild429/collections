---
title: "Linux"
org_id: "E679EB60-C52C-4489-8164-5892DE601136"
---
# Linux

Personal reference, mainly for Ubuntu/Debian with GNOME. Commands are examples: package names, device names, desktop shortcuts, and paths depend on the installed system. DNS notes merged on 2026-09-27; correctness and readability reviewed on 2026-09-30. Machine-specific observations are historical unless explicitly stated otherwise. Commands were reviewed without changing system settings or running installation, disk, firewall, or service operations. Package availability varies by release; check the configured repositories before installing older applications.

## Contents

- [[#Configuration]]
- [[#Install & Recovery]]
- [[#System Configuration]]
- [[#Command Reference]]
- [[#Development]]
- [[#IPC & Concurrency]]
- [[#Networking & Servers]]
- [[#Software]]
- [[#Git]]
- [[#Troubleshooting Collection]]
- [[#Git Reference & CI/CD]]
- [[#固定 GNOME 德语键盘布局：防止登录后重置为 English]]
- [[#Wayland application rendering and keyboard troubleshooting]]

## Configuration

### GNOME Key Mapping

Set XKB options for the GNOME session. Choose one `set` command; each replaces the complete options list. The `reset` command restores the schema default.

```bash
gsettings set org.gnome.desktop.input-sources xkb-options "['altwin:swap_alt_win', 'ctrl:swapcaps']"
gsettings set org.gnome.desktop.input-sources xkb-options "['altwin:swap_lalt_lwin', 'ctrl:swapcaps']"
gsettings reset org.gnome.desktop.input-sources xkb-options
```

- `altwin:swap_alt_win`: swap Alt and Win/Super on both sides.
- `altwin:swap_lalt_lwin`: swap only left Alt and left Win/Super.
- `ctrl:swapcaps`: swap Caps Lock and left Ctrl.
- On an Apple keyboard, Option normally corresponds to Alt and Command to Super; physical mappings depend on the keyboard.

## Install & Recovery

### ISO Installation

| Command | Purpose |
|---------|---------|
| `lsblk` | List block devices and mount points |
| `mkfs` | Create a filesystem (erases existing filesystem data) |
| `fdisk` | Inspect or modify a partition table |

Inspect the device's size, model, and mounted partitions before writing:

```bash
lsblk -o NAME,SIZE,MODEL,FSTYPE,MOUNTPOINTS
sudo fdisk -l
```

To write a bootable hybrid Ubuntu ISO, unmount the USB's mounted partitions, then write to the **whole USB device**, not a partition. `/dev/sdX` is a placeholder; selecting the wrong disk destroys its contents. Formatting first is unnecessary.

```bash
sudo dd if=ubuntu.iso of=/dev/sdX bs=4M status=progress conv=fsync
```

For a normal data disk instead: use `sudo fdisk /dev/sdX`; `g` creates a new GPT, `n` creates a partition (answer its prompts), `p` reviews the table, and `w` writes it. Only then create a filesystem on the intended partition, e.g. `sudo mkfs.ext4 /dev/sdX1`. Do not apply this to a disk containing data to retain.

From Windows, Rufus can write the ISO to a USB drive.

### Disk Backup & Restore

A raw partition clone copies every block, including the filesystem and its UUID. The destination partition must be at least as large as the source partition, regardless of used space. Use a live system with source and destination unmounted for a consistent offline copy. Do not format either partition before cloning or restoring.

```bash
# Destructive to the destination: replace both paths after inspecting disks.
sudo dd if=/dev/SOURCE_PARTITION of=/dev/DESTINATION_PARTITION bs=4M status=progress conv=fsync
```

For restoration, identify the backup and target explicitly and use the backup as `if=`. A larger destination does not automatically enlarge the filesystem: grow it afterward with filesystem-specific tools. There is no universal “10 GB larger” rule. A root-partition clone alone does not copy the partition table, EFI System Partition, or separate `/boot` and `/home` filesystems, and is not a complete bootable-system backup. Resolve duplicate filesystem UUIDs before using both copies in the same system.

For a file backup, mount an ext4 backup filesystem and copy an offline root filesystem mounted at `/mnt/source`. Run the preview first, inspect it, then remove `--dry-run`:

```bash
sudo mkdir -p /mnt/source /mnt/backup
sudo mount -o ro /dev/SOURCE_ROOT_PARTITION /mnt/source
sudo mount /dev/BACKUP_PARTITION /mnt/backup
sudo rsync -aAXHx --numeric-ids --dry-run \
  /mnt/source/ /mnt/backup/LinuxBackup/
```

`-x` stays on one filesystem; back up separate filesystems independently. `-A`, `-X`, and `-H` preserve ACLs, extended attributes, and hard links on a supporting destination. Add `--delete` only for an intentional mirror: it deletes destination files missing from the source. File restoration also requires bootloader and mount configuration work. [rsync manual](https://download.samba.org/pub/rsync/rsync.1)

### GRUB Repair

At the GRUB prompt, inspect partitions to find the GRUB directory. Replace `N` with the actual partition number:

```text
grub> ls
grub> ls (hd0,gptN)/boot/grub/
grub> set root=(hd0,gptN)
grub> set prefix=(hd0,gptN)/boot/grub
grub> insmod normal
grub> normal
```

With a separate `/boot` partition, the path there is usually `/grub`, not `/boot/grub`. This is a temporary recovery route and depends on readable GRUB modules and configuration.

After booting, repair according to the firmware mode. On Ubuntu, regenerate the menu with `sudo update-grub`. For legacy BIOS, installation targets a whole boot disk, e.g. `sudo grub-install /dev/nvme0n1`. **That device is not an EFI partition.** For x86-64 UEFI, the FAT EFI System Partition must be mounted at `/boot/efi`; the installation form is:

```bash
sudo grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ubuntu
sudo update-grub
```

UEFI repair must account for Secure Boot and Ubuntu's signed shim/GRUB packages; do not force an unsigned installation if the tool rejects it. Live-USB repair additionally requires mounting the installed system and setting up a chroot. Boot-Repair is an optional third-party tool, not a prerequisite for `grub-install`. [GNU GRUB installation manual](https://www.gnu.org/software/grub/manual/grub/html_node/Installing-GRUB-using-grub_002dinstall.html)

### Reinitialize a Windows Disk (diskpart)

This erases the selected disk’s partition information; it does **not** recover lost data. Verify the disk with `list disk` and `detail disk` before `clean`.

Open DiskPart with administrative privileges, then replace `N` with the intended disk number:

```text
list disk
select disk N
detail disk
clean
create partition primary
```

Stop after `detail disk` to verify the selection. Creating the partition does not format it or assign a drive letter. `clean` removes partition metadata; it is not a secure erase.

### Boot Menu Keys

My machine: `F12` during POST.

## System Configuration

### NVIDIA Drivers

Use the driver selected for the installed Ubuntu release and GPU; do not pin the historical `nvidia-430` package.

```bash
sudo apt update
ubuntu-drivers devices
sudo ubuntu-drivers install
# Reboot; complete MOK enrollment if the installation requests it.
nvidia-smi
watch -n 1 nvidia-smi
```

A graphics-driver PPA is optional, not required for the normal distribution-supported installation. [Ubuntu driver documentation](https://ubuntu.com/server/docs/how-to/graphics/install-nvidia-drivers/)

### Chinese / I18n

```bash
sudo apt install language-pack-zh-hans
locale -a | grep zh
# Only select a locale that exists; this affects the current shell and its children.
export LC_CTYPE=zh_CN.UTF-8
```

For Chinese PDF output with pandoc:

```bash
fc-list -f "%{family}\n" :lang=zh
pandoc test.org -o test.pdf --pdf-engine=xelatex -V CJKmainfont="AR PL KaitiM GB"
```

The font must be installed, and XeLaTeX plus the required CJK packages must be available. Pandoc’s `CJKmainfont` selects the CJK font through `xeCJK`; `mainfont` alone does not configure CJK-specific typesetting. See [Pandoc LaTeX variables](https://pandoc.org/MANUAL.html#variables-for-latex).

For Emacs Chinese input, configure Fcitx/Fcitx5 and the desktop input-method integration, or use an Emacs input method. `LC_CTYPE` alone does not install or enable an input method. See [Fcitx](#fcitx-chinese-input).

### Screensaver (gluqlo; X11/Xorg)

Historical X11 setup. XScreenSaver does not provide a Wayland session lock. Avoid competing screen lockers; use the desktop’s supported lock mechanism on Wayland.

- Source: <https://github.com/alexanderk23/gluqlo> (build from source)
- Add `gluqlo -root\n\` to `~/.xscreensaver` programs section

```bash
sudo apt-get install xscreensaver xscreensaver-data-extra xscreensaver-gl-extra
# Check the desktop/session integration before replacing its locker
```

- Open Screensaver settings, set Gluqlo as the only screensaver

- Add startup entry: `xscreensaver -nosplash`

- Add keyboard shortcut: `xscreensaver-command -lock`

- If needed, switch from Wayland to Xorg:

  ```bash
  # If this Ubuntu release offers an Xorg session, select it at login.
  # GDM system-wide setting, where supported: /etc/gdm3/custom.conf
  # [daemon]
  # WaylandEnable=false
  ```

### Zsh + Powerlevel10k

```bash
sudo apt install zsh
# Download MesloLGS NF font
# Clone powerlevel10k: git clone --depth=1 https://github.com/romkatv/powerlevel10k.git ~/powerlevel10k
exec zsh
```

After cloning, add `source ~/powerlevel10k/powerlevel10k.zsh-theme` to `~/.zshrc`, select MesloLGS NF in the terminal, and run `p10k configure`. `exec zsh` changes this shell; `chsh -s "$(command -v zsh)"` changes the login shell for future logins.

### Keyboard & Touchpad

For an X11 session, disable the touchpad (`xinput` does not configure native Wayland devices):

```bash
xinput list                          # find touchpad NAME and ID
xinput set-prop 'NAME' 'Device Enabled' 0
```

Add to `~/.bashrc`:

```bash
alias tpOff="xinput set-prop 'SYNA1D31:00 06CB:CD48 Touchpad' 'Device Enabled' 0"
alias tpOn="xinput set-prop 'SYNA1D31:00 06CB:CD48 Touchpad' 'Device Enabled' 1"
```

Autostart disable at graphical login (X11 only): create `~/.config/autostart/xinput.desktop` with `Exec=xinput set-prop … 0`.

Map Caps Lock to an additional Ctrl key (this is not a swap):

```bash
setxkbmap -option ctrl:nocaps   # X11
# System XKB configuration, where used: XKBOPTIONS="ctrl:nocaps"
```

Right-click on touchpad:

```bash
gsettings set org.gnome.desktop.peripherals.touchpad click-method areas
```

For a true swap use `ctrl:swapcaps`. On GNOME/Wayland, use GNOME’s XKB options as shown above. GNOME can disable the touchpad with `gsettings set org.gnome.desktop.peripherals.touchpad send-events disabled`; use `enabled` to restore it.

### Hotkeys

Personal bindings; these are not universal desktop defaults. `C` = Ctrl, `M` = Alt/Meta, `S` = Shift, `SPC` = Space.

|              |                  |
|--------------|------------------|
| C-M-t       | terminal         |
| C-M-p       | thunderbird      |
| C-M-e       | emacs            |
| C-M-f       | firefox          |
| C-M-j       | emacs window     |
| C-M-w       | emacs wörterbuch |
| Alt+Tab      | switch app       |
| Alt+Spc      | switch window    |
| Alt+Ctrl+Del | log out          |

### Autostart

Create desktop entry in `~/.config/autostart/`, e.g. for thunderbird:

```conf
[Desktop Entry]
Type=Application
Exec=/usr/bin/thunderbird
Hidden=false
NoDisplay=false
X-GNOME-Autostart-enabled=true
Name=thunderbird
```

### Firewall

Use the firewall manager configured on the system; Ubuntu may use UFW instead. The commands below affect firewalld’s default zone unless `--zone=ZONE` is supplied. Do not enable a second firewall manager without planning its interaction with the existing one. Identify the interface’s zone with `sudo firewall-cmd --get-active-zones`. The add/remove commands are alternatives; reload after the intended change.

```bash
sudo apt install firewalld
sudo systemctl enable --now firewalld
sudo firewall-cmd --add-port=80/tcp --permanent
sudo firewall-cmd --remove-port=80/tcp --permanent
sudo firewall-cmd --reload
sudo firewall-cmd --list-all

systemctl status firewalld
sudo systemctl stop firewalld       # stop now
sudo systemctl disable firewalld    # prevent startup; does not stop it
```

### Memory and Caches

Linux reclaims filesystem caches automatically. Inspect `free -h` and its `available` column before treating cache usage as a problem. For a controlled diagnostic only:

```bash
sudo sh -c 'sync; echo 3 > /proc/sys/vm/drop_caches'
```

This drops clean page cache and reclaimable dentries/inodes; it does not free application memory. It can hurt performance and is not routine maintenance. `swapoff -a` is not a cache-clearing step and may fail or cause severe memory pressure. [Kernel documentation](https://docs.kernel.org/admin-guide/sysctl/vm.html)

### Custom .desktop Entries

1.  Find executable and icon

2.  Create `~/.local/share/applications/myapp.desktop` for the current user (create the directory if needed):

    ```conf
    [Desktop Entry]
    Version=1.0
    Type=Application
    Terminal=false
    Exec=/path/to/yourapp
    Name=YourApp
    Icon=/path/to/yourapp.png
    ```

3.  Save the file. Application-menu entries normally do not need executable permission; launching a `.desktop` file directly from a file manager may additionally require marking it trusted/executable.

### Hostname

```bash
hostnamectl
sudo hostnamectl set-hostname NEWNAME
cat /etc/hostname
cat /etc/hosts
```

### System Monitor

Use `gnome-system-monitor` for a standalone monitor:

```bash
sudo apt install gnome-system-monitor
```

Panel extensions must match the installed GNOME Shell version. The old `indicator-multiload`/Clutter dependency recipe is not a general installation procedure for GNOME extensions.

### hstr (Shell History Search)

```bash
sudo apt update
sudo apt install hstr   # if available in this release’s repositories
hstr --show-configuration
```

Follow the generated configuration for your shell to bind `Ctrl-R`/`hh`. Availability and bindings depend on the shell and package version; the historical PPA is not required when the distro ships hstr.

## Command Reference

### File & Directory

```bash
find ~ -name filename                # find by name
tree -L 2                            # directory tree
xdg-open .                          # use the desktop’s default file manager
```

### Text Processing

```bash
grep -n "pattern" file.txt           # search with line numbers
sort -k3,3 file.txt                  # sort whitespace-separated field 3
sort -t, -k3,3 test.csv              # simple comma-delimited data, not quoted CSV
sort --reverse test.csv
awk '!a[$0]++' file.txt              # remove duplicate lines
printf '%s\n' "$PATH" | tr ':' '\n'               # split PATH on colon
find . -type f -name "*.md" -exec sed -i 's/foo/bar/g' {} +
```

### Links (Soft & Hard)

Soft link (symlink, shortcut):

```bash
ln -s /absolute/path/to/source linkname
```

An absolute target keeps pointing to the same path if the symlink moves; moving the target breaks it. A relative target is resolved from the symlink’s directory and is useful when moving a whole directory tree together.

Hard link:

```bash
ln filename linkname
```

Shares the same inode and file contents within one filesystem. It adds a directory entry without duplicating file data; hard links to directories are normally forbidden.

On traditional Unix filesystems, a directory’s link count is usually 2 plus its immediate subdirectories (`.` and the parent entry, plus each child’s `..`). Some filesystems and large-directory features report different counts.

### Archives (tar)

| Option | Meaning      |
|--------|--------------|
| -x     | extract      |
| -c     | create       |
| -v     | verbose      |
| -z     | gzip         |
| -f     | specify file |

```bash
tar -zvcf archive.tar.gz dir/
tar -zvxf archive.tar.gz
```

### Less Pager

|       |                |
|-------|----------------|
| j     | down           |
| k     | up             |
| Space | next page      |
| b     | previous page  |
| /     | search         |
| n     | next match     |
| N     | previous match |
| q     | quit           |

### Terminal Shortcuts

These depend on terminal bindings and shell line-editing mode. The editing keys below assume common Emacs-style bindings; application keymaps can override them.

|       |                      |
|-------|----------------------|
| S-C-c | copy from terminal   |
| S-C-v | paste to terminal    |
| C-h   | backspace            |
| C-j   | enter                |
| C-k   | cut to end of line   |
| C-u   | cut to start of line |

### Process & Port Inspection

```bash
ps aux | grep '[m]ongo'
sudo ss -ltnp 'sport = :80'          # listening TCP sockets on port 80
```

### Output Redirection

```bash
command >> file          # append stdout
command >> file 2>&1    # append stdout + stderr
cat 110.txt > 111.txt     # copy contents; truncate destination first
cat source1.c >> source2.c # append contents; source and destination must differ
```

### Misc

```bash
# Find and uninstall a package
apt list --installed | grep software
whereis software
sudo apt-get --purge remove software
sudo apt-get autoremove

# Set root password
sudo passwd

# Get public IP
curl https://ifconfig.me

# cd to directory of a binary
cd "$(dirname "$(command -v cling)")"   # only if cling is installed

# GNU sed: preview matches before an intentional in-place replacement.
# See the Text Processing section for the replacement command.
```

### sed Quick Reference

| cmd | action                      |
|-----|-----------------------------|
| a   | add line(s) after match     |
| c   | replace matched line        |
| i   | insert line(s) before match |
| s   | substitute                  |
| d   | delete                      |

Example:

```bash
sed -En '/pattern/s/old/new/gp' file.csv  # on matching lines, replace all; print changes
sed 's/^[[:blank:]]*//' file.csv          # strip leading spaces and tabs
```

## Development

### GCC Basics

```bash
gcc -c file.c -o file.o          # compile only
gcc file1.o file2.o -o app       # link
gcc -I ./include main.c -L ./lib -lmylib -o app   # with headers and library
```

### Static Libraries (.a)

1.  Compile sources to object files: `gcc *.c -c -I ../include`
2.  Archive: `ar rcs libtest.a *.o`
3.  Verify: `nm libtest.a`
4.  Consumer compiles: `gcc main.c -I ./include/ -L ./lib/ -ltest -o app`
    - `-ltest` strips `lib` prefix and `.a` suffix

**Tradeoffs:** linked code is included in the executable; library updates require relinking, and multiple executables can duplicate that code. Linking one `.a` does not make the entire executable static. `-ltest` may choose `libtest.so` if both forms exist; name the `.a` explicitly when needed.

### Dynamic / Shared Libraries (.so)

Build:

```bash
gcc -c -fPIC *.c -I ../include        # position-independent code
gcc -shared *.o -o libxxx.so
```

Consumer compiles:

```bash
gcc -I ./include main.c -L ./lib -lxxx -o app
```

Runtime — the loader must find the `.so`:

- Per-session: `export LD_LIBRARY_PATH="$PWD/lib${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}"`. This captures the current directory’s `lib` path; avoid global exports for unrelated applications.
- System installation: add the trusted absolute library directory to a file under `/etc/ld.so.conf.d/`, then run `sudo ldconfig`.
- At link time: `-Wl,-rpath=/absolute/path/to/lib`

One-shot build example:

```bash
gcc -c -fPIC add.c sub.c mult.c divi.c
gcc -shared -o libmymath.so add.o sub.o mult.o divi.o
gcc -L. -Wl,-rpath,'$ORIGIN' -Wall -o mathDemo mathDemo.c -lmymath
./mathDemo
```

Put libraries after the source/object files that reference them. `$ORIGIN` in an ELF runtime search path means the executable’s directory. [GCC linking documentation](https://gcc.gnu.org/onlinedocs/gcc/Link-Options.html)

**Go/cgo:** cgo supports linker flags through `#cgo LDFLAGS:` or `CGO_LDFLAGS`; it is not restricted to `LD_LIBRARY_PATH`. Runtime paths can be passed via linker flags, subject to the platform/linker and cgo flag validation. [cgo documentation](https://go.dev/cmd/cgo/)

### dlopen / dlsym (runtime loading)

Functions: `dlopen`, `dlclose`, `dlsym` — load a shared library at runtime and call its symbols.

### Makefile

Recipes begin with a literal TAB, not spaces. These examples track `.c` changes; add compiler-generated header dependencies for a larger project.

Format:

```makefile
target: dependencies
	command
```

V1 — simplest:

```makefile
app: main.c add.c sub.c mul.c
	gcc main.c add.c sub.c mul.c -o app
```

V2 — incremental (only recompile changed files):

```makefile
app: main.o add.o sub.o mul.o
	gcc main.o add.o sub.o mul.o -o app
%.o: %.c
	gcc -c $< -o $@
```

V3 — automatic variables and wildcard:

```makefile
src = $(wildcard ./*.c)
obj = $(patsubst %.c, %.o, $(src))
target = app

$(target): $(obj)
	gcc $^ -o $@
%.o: %.c
	gcc -c $< -o $@

.PHONY: clean
clean:
	rm -f $(obj) $(target)
```

| variable | meaning          |
|----------|------------------|
| $@      | target           |
| $<     | first dependency |
| $^      | all dependencies |

## IPC & Concurrency

### Concepts

- **Concurrency:** multiple tasks make progress over overlapping periods; they may interleave on one core or run in parallel. Tasks may be processes, threads, or asynchronous operations.
- **Parallelism:** work executes simultaneously, typically across multiple cores.
- **Synchronous / asynchronous:** whether an operation completes as part of the call or reports completion later. Related blocking/nonblocking behavior depends on the API, not simply on the number of threads.

### Multi-Process Communication

#### Message Queue (SysV)

Key API:

| function | purpose                   |
|----------|---------------------------|
| ftok()   | derive IPC key (not guaranteed unique)          |
| msgget() | create/open queue         |
| msgsnd() | send message (by type)    |
| msgrcv() | receive message (by type) |
| msgctl() | control queue; `IPC_RMID` removes it             |

A message struct must start with a positive `long mtype`. The `msgsz` argument excludes that leading `long`.

One-way: server sends type 100, client receives type 100. Two-way: fork server → parent sends type 100, child receives type 200; client does the opposite.

#### Named Pipe (FIFO)

```bash
mkfifo ./myfifo
```

One process opens `O_WRONLY`, another `O_RDONLY`. Blocking opens normally wait for the other end; `O_NONBLOCK` changes the behavior. A FIFO is a byte stream, not a message queue.

#### Unnamed Pipe

```c
int fd[2];
pipe(fd);
// fd[0] for read, fd[1] for write
// Typically used between processes after fork(); close unused ends.
// Writes can block when the buffer fills; arrange a concurrent reader.
```

#### Signals

| function | purpose                                                  |
|----------|----------------------------------------------------------|
| alarm(n) | SIGALRM after n seconds                                  |
| kill()   | send signal to a PID                                     |
| raise()  | send signal to self                                      |
| pause()  | wait until a signal terminates the process or a handler returns                           |
| signal() | register handler / SIG_IGN / SIG_DFL |

#### Semaphore & Shared Memory

| function | purpose                         |
|----------|---------------------------------|
| shmget() | create/open shared memory       |
| shmat()  | attach to process address space |
| shmdt()  | detach                          |
| shmctl() | inspect/control; `IPC_RMID` marks the segment for deletion |
| semget() | create semaphore set            |
| semctl() | control / destroy semaphore     |
| semop() | perform semaphore operations |

Inspect / clean up:

```bash
ipcs -m    # shared memory
ipcs -q    # message queues
ipcs -s    # semaphores
ipcrm -m SHMID  # Replace SHMID with the specific segment ID to remove
```

Shared memory marked with `IPC_RMID` is destroyed after its last attachment is detached; `shmdt()` only detaches the calling process. See [`shmctl`](https://man7.org/linux/man-pages/man2/shmctl.2.html).

Prefer `sigaction()` for signal handlers. Handlers may call only async-signal-safe functions; `printf()` is not one. `alarm(0)` cancels a pending alarm.

### Multi-Thread (pthreads)

The C blocks below are fragments, not standalone programs. Include `<pthread.h>` and supply an enclosing function, a declared `pthread_t tid`, a compatible `void *func(void *)` entry point, and any argument data where needed.

Compile and link with `-pthread`. Examples omit error handling for brevity; check return values (most pthread functions return an error number directly). Cancellation is a request, usually acted on at cancellation points; asynchronous cancellation can leave shared state inconsistent.

| function                           | purpose                        |
|------------------------------------|--------------------------------|
| pthread_create()         | spawn thread                   |
| pthread_join()           | wait and retrieve return value |
| pthread_detach()         | detach (cannot join)           |
| pthread_exit()           | exit thread                    |
| pthread_cancel()         | request cancellation           |
| pthread_setcancelstate() | enable/disable cancellation    |
| pthread_setcanceltype()  | deferred vs asynchronous       |
| pthread_self()           | get own thread ID              |

Mutex:

```c
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_lock(&mutex);
// critical section
pthread_mutex_unlock(&mutex);
```

Read-Write Lock:

```c
pthread_rwlock_t rwlock;
pthread_rwlock_init(&rwlock, NULL);
pthread_rwlock_wrlock(&rwlock);   // exclusive write section
// modify protected data
pthread_rwlock_unlock(&rwlock);
pthread_rwlock_rdlock(&rwlock);   // separate shared read section
// read protected data
pthread_rwlock_unlock(&rwlock);
pthread_rwlock_destroy(&rwlock);
```

Do not acquire a read lock while holding the same write lock: it may deadlock or return an error. [POSIX read-lock reference](https://www.man7.org/linux/man-pages/man3/pthread_rwlock_rdlock.3p.html)

Thread attributes:

```c
pthread_attr_t attr;
pthread_attr_init(&attr);
pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_JOINABLE); // or DETACHED
// Optional: pthread_attr_setstacksize(&attr, stack_size);
// Check the return code; stack_size must meet platform minimum/alignment rules.
pthread_create(&tid, &attr, func, arg);
pthread_attr_destroy(&attr);
```

## Networking & Servers

### DNS and Name Resolution

For how DNS relates to routes, gateways, and NAT, see [Network](Network.md).

Merged from `Linux_DNS_Resolution.org` (Org ID `C28DE385-19F7-4B66-AB36-7A12DD2CF6DA`). The original note’s “current setup” was a historical observation, not a verified description of the machine in use now.

#### Core Configuration Files

| File | Role |
|------|------|
| `/etc/resolv.conf` | Traditional DNS resolver configuration, including `nameserver IP`, search domains, and options. Often managed by NetworkManager, systemd-resolved, or resolvconf; inspect before editing. |
| `/etc/hosts` | Local address/name mappings: `IP_address canonical_hostname [aliases...]`. Useful for development and local overrides when the lookup path consults it. It is not a universal website blocker. |
| `/etc/nsswitch.conf` | For glibc NSS clients, the `hosts:` line selects lookup sources and the actions taken after each result. |
| `/etc/networks` | Legacy mapping of network names to network numbers for applications using the `networks` database; does not configure interfaces, routes, or CIDR subnets. |

Examples, not replacement recommendations:

```conf
hosts: files dns
# Local hosts file first, then the DNS resolver.

hosts: dns files
# DNS resolver first, then local hosts file if the result permits fallback.
```

NSS normally stops on success and continues on not-found/unavailable results. Bracketed rules override this: `[NOTFOUND=return]` stops after a not-found result from the preceding source. Actual configurations may include `resolve`, `myhostname`, and mDNS modules. Preserve those integrations unless intentionally changing them. [NSS manual](https://man7.org/linux/man-pages/man5/nsswitch.conf.5.html)

#### Resolution Flow

1. An application using glibc’s hostname lookup API, such as `getaddrinfo()`, follows the NSS `hosts:` configuration.
2. `files` checks `/etc/hosts`; `dns` uses the traditional resolver configuration; `resolve` contacts systemd-resolved through its native interface.
3. The source result and any NSS action rule determine whether lookup stops or continues.

With systemd-resolved, `nameserver 127.0.0.53` commonly denotes the local stub; `resolvectl status` shows upstream DNS servers. A symlink to `/run/systemd/resolve/resolv.conf` instead lets traditional DNS clients query upstream servers directly. Resolved also reads `/etc/hosts` by default and routes queries according to per-link/search domains, including VPN configuration. `.local` normally belongs to mDNS, so use `example.com` for ordinary DNS tests. [systemd-resolved manual](https://www.man7.org/linux/man-pages/man8/systemd-resolved.service.8.html)

Applications using their own resolver or DNS-over-HTTPS can bypass NSS. `dig` queries DNS rather than running the NSS source sequence, although a local DNS stub may itself serve local entries. Therefore `dig` and `getent` can legitimately differ.

#### Inspection and Troubleshooting

```bash
grep '^hosts:' /etc/nsswitch.conf
cat /etc/hosts
ls -l /etc/resolv.conf
readlink -f /etc/resolv.conf
cat /etc/resolv.conf

getent ahosts example.com       # test the system NSS lookup path
resolvectl status              # if systemd-resolved is in use
resolvectl query example.com    # query through resolved

dig example.com                # DNS query to the configured DNS server
dig @1.1.1.1 example.com        # direct query; bypasses local/VPN DNS selection
```

On Ubuntu, `dig` is provided by `dnsutils`. Change persistent DNS settings through the active network manager (e.g. a NetworkManager connection or Netplan configuration), rather than overwriting a managed `/etc/resolv.conf`. Direct public DNS queries will not resolve private VPN-only names. If resolved is running and stale cache entries are suspected, use `sudo resolvectl flush-caches`.

### SSH

```bash
ssh user@ip
# For automation, use an SSH key and agent rather than a literal password.
```

Passwordless login:

```bash
ssh-keygen -t ed25519                  # choose a file and passphrase; do not overwrite an existing key
ssh-copy-id user@ip
ssh-copy-id localhost                  # (username ≠ hostname)
```

Root login is usually unnecessary: connect as a normal user and use `sudo`. If a system explicitly requires root SSH access, `PermitRootLogin prohibit-password` permits key-based authentication while blocking password login. `AllowUsers`, when present, restricts login to its listed users; it does not grant privileges. Validate edits with `sudo sshd -t` before reloading the SSH service, and keep the existing session open while testing another connection.

### Docker

Install: <https://docs.docker.com/engine/install/ubuntu/> NVIDIA container toolkit: <https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html>

```bash
getent group docker || sudo groupadd docker
sudo usermod -aG docker "$USER"
# Log out and back in to refresh supplementary groups.

docker pull image
docker run -it --gpus all image /bin/bash

# Cleanup: review objects before deleting them.
docker container prune   # removes all stopped containers (confirmation prompt)
docker image prune       # removes dangling images (confirmation prompt)
```

Membership in the `docker` group grants root-level capabilities. Restarting the daemon does not refresh the current shell’s group membership. [Docker post-installation documentation](https://docs.docker.com/engine/install/linux-postinstall/)

### uWSGI + Nginx + Django

Example for Ubuntu, a dedicated application user `user`, Nginx group `www-data`, project `/srv/project`, and virtual environment `/srv/project/.venv`. Adapt ownership and paths to the deployment. Install uWSGI inside that environment (a C compiler/Python development headers may be required), not with `sudo pip`:

```bash
/srv/project/.venv/bin/python -m pip install uwsgi
cd /srv/project
/srv/project/.venv/bin/uwsgi --http 127.0.0.1:8000 --module project.wsgi
```

Create `/etc/uwsgi/project.ini`:

```ini
[uwsgi]
chdir = /srv/project
module = project.wsgi:application
home = /srv/project/.venv
master = true
processes = 2
socket = /run/uwsgi/project.sock
chmod-socket = 660
vacuum = true
die-on-term = true
```

Create `/etc/systemd/system/uwsgi.service`:

```ini
[Unit]
Description=Django application via uWSGI
After=network.target

[Service]
User=user
Group=www-data
WorkingDirectory=/srv/project
RuntimeDirectory=uwsgi
RuntimeDirectoryMode=0750
ExecStart=/srv/project/.venv/bin/uwsgi --ini /etc/uwsgi/project.ini
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

The application user must be able to read the project/environment and write required application data. The service creates its socket in a runtime directory shared with Nginx; do not make it world-writable. Tune the worker count for the workload.

Nginx server configuration (enable it using the distribution’s layout):

```nginx
server {
    listen 80;
    server_name example.com;

    location /static/ { alias /srv/project/static/; }
    location /media/  { alias /srv/project/media/; }
    location / {
        include /etc/nginx/uwsgi_params;
        uwsgi_pass unix:/run/uwsgi/project.sock;
    }
}
```

Set `STATIC_ROOT` to the served static directory, run `collectstatic`, configure `ALLOWED_HOSTS`, disable `DEBUG`, and provide secrets through the deployment configuration. Nginx needs read/traverse access to static/media files. Do not serve private uploads publicly.

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now uwsgi
sudo nginx -t && sudo systemctl reload nginx
journalctl -u uwsgi
```

This HTTP example needs TLS before production use. Run Django’s deployment checks and configure proxy/HTTPS settings for the actual setup. [uWSGI’s Django/Nginx guide](https://uwsgi-docs.readthedocs.io/en/latest/tutorials/Django_and_nginx.html)

### SSL / TLS Certificates

A self-signed development server certificate needs a Subject Alternative Name (SAN) matching the hostname. Keep the private key private:

```bash
(umask 077; openssl req -x509 -newkey rsa:2048 -noenc -sha256 -days 365 \
  -keyout key.pem -out cert.pem -subj "/CN=example.test" \
  -addext "subjectAltName=DNS:example.test" \
  -addext "basicConstraints=critical,CA:FALSE")
```

`-noenc` is the OpenSSL 3 spelling; older versions use `-nodes`. A self-signed leaf certificate is not automatically a certificate authority. For a private CA, create a separate CA key/certificate with CA constraints, then sign leaf CSRs with SAN and appropriate server extensions. [OpenSSL documentation](https://docs.openssl.org/master/man1/openssl-req/)

To trust an intentionally selected **CA certificate**, use the distribution’s trust store (applications may maintain separate stores):

```bash
# Ubuntu/Debian: PEM certificate with a .crt extension
sudo cp my-root-ca.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates

# Fedora/RHEL alternative
sudo cp my-root-ca.crt /etc/pki/ca-trust/source/anchors/
sudo update-ca-trust
```

For Let’s Encrypt, with Certbot installed:

```bash
sudo certbot certonly --standalone -d example.com
```

The standalone HTTP challenge needs public DNS pointing to this server and inbound port 80 reachable/free. `certonly` obtains a certificate; configure the web server to use it, ensure automatic renewal is scheduled, and arrange certificate reloads after renewal.

### GWDG Cloud Server

Historical personal workflow; activation, portal navigation, and image usernames have not been verified for the current service. These are recorded steps, not a current provisioning guide. The documentation endpoint could not be retrieved during this review; confirm the workflow through GWDG before use.

1.  Email support@gwdg.de with university email to request cloud server activation
2.  GWDG website → Cloud Server → Self-service → Create Instance
3.  Use VNC to activate account; save password
4.  Connect: `ssh cloud@ip`

<a id="95D40680-B476-44AC-90EA-0EC5243D3B36"></a>

## Software

### PDF

#### Okular

```bash
sudo apt-get install okular
```

Personal/custom bindings from the original notes (not stock Okular shortcuts). The documented default presentation shortcut is `Ctrl+Shift+P`; `F7` below is a personal binding if configured. [Okular handbook](https://docs.kde.org/trunk_kf6/en/okular/okular/menuview.html)

| key       | action            |
|-----------|-------------------|
| F2        | customize         |
| F7        | presentation mode |
| C-g g     | go to page        |
| C-n / p   | next / prev page  |
| M-n / p   | scroll down / up  |
| C-b C-b 1 | add note          |
| SPC-b     | add bookmark      |
| SPC-SPC   | rename bookmark   |

#### evince

```bash
evince file.pdf
```

#### Ghostscript (gs)

Rewrite a PDF through Ghostscript (not a guarantee that all active content is sanitized; output may lose annotations, forms, or other features):

```bash
gs -dNOPAUSE -sDEVICE=pdfwrite -sOUTPUTFILE=out.pdf -dBATCH in.pdf
```

Re-encode using the `/prepress` preset (may increase size; compare quality and size rather than assuming compression):

```bash
gs -sDEVICE=pdfwrite -dCompatibilityLevel=1.4 -dPDFSETTINGS=/prepress \
   -dNOPAUSE -dQUIET -dBATCH -sOutputFile=compressed.pdf input.pdf
```

#### xournal

```bash
sudo apt install xournal
```

### Media

#### mpv

```bash
sudo apt install mpv
```

Config at `~/.config/mpv/mpv.conf` — key options:

```conf
no-osd-bar
save-position-on-quit
no-border
hwdec=auto
autofit-larger=92%
sub-auto=fuzzy
sub-font-size=40
screenshot-format=png
```

Options depend on the installed version; see the [mpv manual](https://mpv.io/manual/stable/). For an autoload playlist script, see: <https://github.com/mpv-player/mpv/blob/master/TOOLS/lua/autoload.lua>

#### simplescreenrecorder

Primarily an X11 recorder; native Wayland screen capture requires a compatible recorder/portal workflow. The shortcut below is configuration-dependent.

```bash
sudo apt install simplescreenrecorder
alias ssr='simplescreenrecorder'
# Ctrl+Shift+Alt+V to start/pause
```

#### kmplayer

```bash
sudo apt install kmplayer
```

### Dictionary (stardict)

```bash
sudo apt install stardict sdcv
mkdir -p ~/.stardict/dic
# Inspect one downloaded archive, then extract it into the user dictionary directory.
tar -tjf dictionary.tar.bz2
tar -xjf dictionary.tar.bz2 -C ~/.stardict/dic
```

`dictionary.tar.bz2` is a placeholder for the chosen dictionary archive. The historical catalog was `http://download.huzheng.org/`; confirm the archive source and license. Do not run `bzip2` first: `tar -xjf` already decompresses bzip2. System-wide dictionaries can instead be installed under `/usr/share/stardict/dic/`.

### Email (thunderbird)

- `Alt` to open menubar
- Filter rules: `~/.thunderbird/PROFILE/ImapMail/imap.gmail.com/msgFilterRules.dat`

### GPG

| action             | command               |
|--------------------|-----------------------|
| list secret keys   | gpg --list-secret-keys |
| encrypt (terminal) | gpg -r user -e file   |
| decrypt (terminal) | gpg -d file.gpg       |
| encrypt (Emacs)    | epa-encrypt-file      |
| decrypt (Emacs)    | epa-decrypt-file      |
| verify             | gpg --verify file.sig file  |

### IPFS

Install Kubo (formerly go-ipfs). Initialize only a new repository:

```bash
ipfs init
ipfs config edit
ipfs id
ipfs daemon
```

Keep the daemon running; use another terminal for commands:

```bash
printf 'hello\n' > file.txt
file_cid=$(ipfs add -Q file.txt)
ipfs cat "$file_cid"

# Use a directory whose contents you intend to share.
directory_cid=$(ipfs add -Qr ./public-directory)
ipfs name publish "/ipfs/$directory_cid"
```

A CID identifies content; IPNS publishes a mutable name pointing to that content. Adding files does not guarantee long-term availability from other peers. Avoid publishing the whole working directory accidentally. The local Web UI is normally at `http://localhost:5001/webui` while the daemon runs. See [Kubo CLI](https://docs.ipfs.tech/reference/kubo/cli/).

### Fcitx (Chinese Input)

Choose the input-method framework supported by the desktop and distribution. An Ubuntu Fcitx5/Pinyin setup may use:

```bash
sudo apt install fcitx5 fcitx5-chinese-addons fcitx5-config-qt im-config
im-config -n fcitx5
# Log out and back in; configure Pinyin and Keyboard - German as needed.
```

Wayland integration depends on the compositor and toolkit; follow the distribution’s input-method setup. [Fcitx5 setup guide](https://fcitx-im.org/wiki/Setup_Fcitx_5) `LC_CTYPE=zh_CN.UTF-8` changes locale behavior, not the selected input method.

The older Sogou setup used Fcitx 4 and a vendor `.deb` (`sudo apt install ./sogoupinyin_VERSION_ARCH.deb`). Verify release/framework compatibility before using it; do not assume a Fcitx 4 add-on works with Fcitx5.

### Graphviz (dot)

```bash
dot -Tpng -O file.dot
```

### KDE Connect (Linux ↔ Android)

Install on both devices and pair them. Discovery usually works on the same reachable LAN, but firewall rules, VPNs, or wireless client isolation can prevent it. Received-file locations depend on the device and plugin settings; `Downloads/` is a common default.

### Photopea (Web-based image editor)

- Upload image → Image → Adjustments → Levels → click white dropper on background
- Remove background: Magic Wand → uncheck "Contiguous" → Delete

### File Manager (ranger)

```bash
sudo apt install ranger
```

### Image Viewer (eog)

```bash
eog image.png
```

### Warp Terminal

Historical personal keymap; confirm bindings in the installed Warp version. `Ctrl+f` is context-dependent and should not be assumed to move left.

|        |                                 |
|--------|---------------------------------|
| Ctrl+f | accept autocomplete (context-dependent) |
| Ctrl+n | next block                      |
| Ctrl+p | previous block                  |

## Git

### Setup

```bash
sudo apt install git
git config --global user.name "name"
git config --global user.email "email"
ssh-keygen -t ed25519 -C "email"       # add the public key to GitHub
```

### Basic Workflow

```bash
git init -b main
git remote add origin git@github.com:user/repo.git
git add -A
git commit -m "message"
git push -u origin main
```

Amend the last commit (creates a replacement commit with a new ID; rewrites history):

```bash
git commit --amend
```

### Branching

```bash
git switch -c development
# ... work, commit, push ...
git push -u origin development

# Merge to main
git switch main
git pull --ff-only origin main
git merge development
git push origin main

# Delete branch
git branch -d development
git push origin --delete development
```

Track remote branch:

```bash
git switch --track origin/ui-mockup
```

### Recovery

| Intent | Command | Effect |
|--------|---------|--------|
| Discard unstaged changes in one tracked file | `git restore -- file` | Replaces the working copy from the index; unsaved edits are lost |
| Unstage a file | `git restore --staged -- file` | Keeps working-tree edits |
| Undo a published commit | `git revert COMMIT` | Creates a new inverse commit |
| Remove the last local commit, keep changes staged | `git reset --soft HEAD~1` | Moves the branch back |
| Remove the last local commit and its tracked changes | `git reset --hard HEAD~1` | Destructive; can also overwrite obstructing untracked files |
| Recover a previous commit | `git reflog`, then `git branch recovered COMMIT` | Preserves that commit on a new branch for inspection |
| Stop tracking a file | `git rm --cached -- file` | Keeps the local file; commit the index change |

Reflog recovery is local and time-limited; it cannot recover arbitrary edits that were never recorded by Git. Add ignored local files to `.gitignore` as appropriate.

### Magit (Emacs)

`C-x g` requires an Emacs binding to `magit-status`; it is not guaranteed by a plain Emacs installation. `c` opens the commit transient; `c c` starts a normal commit.

| key     | action        |
|---------|---------------|
| C-x g   | open magit    |
| s / S   | stage         |
| c       | commit        |
| C-c C-c | finish commit |
| P p     | push          |

### GitHub

Use the file page’s **Raw** button to obtain its raw-content URL. Query parameters such as `?raw=true` are page-dependent, not a universal transformation for every GitHub URL.

## Troubleshooting Collection

### System limit for number of file watchers reached

Inspect the existing limits and the application error first:

```bash
sysctl fs.inotify.max_user_watches fs.inotify.max_user_instances
```

Do not blindly set `max_user_watches=100000`: that could lower the current limit. If watch exhaustion is confirmed, choose a larger limit appropriate for the workload; for example, only when the current value is lower:

```bash
sudo sysctl -w fs.inotify.max_user_watches=524288
```

This runtime change does not persist after reboot. For persistence, place the chosen assignment in a dedicated file under `/etc/sysctl.d/` and load that file with `sudo sysctl -p /etc/sysctl.d/99-inotify.conf`. Instance exhaustion is a separate limit; increasing watches alone will not fix it. See [inotify limits](https://man7.org/linux/man-pages/man7/inotify.7.html).

### Right-click on touchpad not working

See [[#Keyboard & Touchpad]] for the GNOME `click-method areas` setting.

### Package Manager Locks / Broken Packages

For a lock error, let the active APT/dpkg operation finish and inspect the owning process; do not delete lock files while a package manager is running. For interrupted configuration or broken dependencies:

```bash
sudo dpkg --configure -a
sudo apt --fix-broken install
```

## Git Reference & CI/CD

### Additional Git Commands

```bash
git log --oneline --graph --decorate
git config --global init.defaultBranch main
git branch -vv
git diff             # unstaged changes relative to the index
git diff HEAD        # working tree relative to HEAD
git diff --staged    # staged changes relative to HEAD
git stash push -u    # include untracked files; ignored files excluded
git stash pop        # reapply; conflicts may need resolution
```

- `git reset --soft`: moves HEAD/branch; retains index and working tree.
- `git reset --mixed` (default): also resets the index; retains working-tree edits.
- `git reset --hard`: also resets tracked working-tree content; destructive.
- `git revert HEAD~1`: reverts the commit before HEAD, not HEAD itself.
- Switching branches only requires saving/stashing changes when they would be overwritten or conflict; stashing is not always necessary.
- A fast-forward merge moves the branch pointer without a merge commit. A normal divergent merge creates a merge commit unless another mode is selected.
- Rebase replays commits on a new base, generally creating new commit IDs. Resolve conflicts, stage the resolutions, then run `git rebase --continue`; use `git rebase --abort` to return to the prior state. [Git rebase manual](https://git-scm.com/docs/git-rebase)

### SSH Agent

```bash
eval "$(ssh-agent -s)"    # only if the session does not already provide an agent
ssh-add ~/.ssh/id_ed25519
```

### CI/CD Triggers

Depending on the CI platform, workflows can be triggered by repository events, manual requests, schedules, external events, or completion of another workflow. These are CI/CD concepts, not SSH features.

## Craft import — Linux — 2026-09-27

<!-- Org properties: {"craft_id": "795D0A6D-0158-46B8-ABC0-FF714C0CC66D", "imported": "2026-09-27"} -->

Source: [Linux in Craft](craftdocs://open?spaceId=f0e27734-d8b8-47ce-be9d-9b35def0cb70&blockId=795D0A6D-0158-46B8-ABC0-FF714C0CC66D)

### 固定 GNOME 德语键盘布局：防止登录后重置为 English

**适用环境：** Ubuntu · GNOME · Wayland · Fcitx5

#### 问题与排查结论

原笔记记录：系统键盘已设为 German（`de`），但每次注销并重新登录后，GNOME 输入源仍恢复为 English（`us`）。修改 AccountsService 也无效，设置会在登录过程中再次被覆盖；最终通过系统级 dconf 策略锁定 GNOME 输入源解决。锁定能防止改写，但本身不能确认究竟是哪个组件在覆盖配置。

#### 最终方案：系统级 dconf 策略

**1 · 准备配置目录**

```bash
sudo mkdir -p /etc/dconf/profile /etc/dconf/db/local.d/locks
```

**2 · 配置 dconf profile**

编辑 `/etc/dconf/profile/user`，确保包含以下两行；如已有其他配置，保留原有条目：

```text
user-db:user
system-db:local
```

**3 · 将 GNOME 输入源设为德语**

创建或编辑 `/etc/dconf/db/local.d/00-keyboard`：

```text
[org/gnome/desktop/input-sources]
sources=[('xkb', 'de')]
```

**4 · 锁定输入源，防止登录时覆盖**

创建或编辑 `/etc/dconf/db/local.d/locks/00-keyboard`：

```text
/org/gnome/desktop/input-sources/sources
```

**5 · 应用并验证**

```bash
sudo dconf update
```

注销并重新登录，然后检查：

```bash
gsettings get org.gnome.desktop.input-sources sources
# 预期：[('xkb', 'de')]

gsettings writable org.gnome.desktop.input-sources sources
# 预期：false
```

`false` 表示系统级锁定已生效，GNOME 用户会话无法再改写该输入源。该策略适用于使用此 dconf profile 的用户。

#### 最终结构与结论

```text
Ubuntu / GNOME
└── German（de）· QWERTZ 键盘布局

Fcitx5
├── Keyboard - German
├── Pinyin
└── 其他输入法
```

**此机器上的结果：** GNOME 登录后保持德语键盘布局；Pinyin 等输入法由 Fcitx5 管理。此结果取决于会话和输入法集成，并不是所有 Wayland 桌面的通用保证。

锁定后，用户也无法在 GNOME 设置中添加或切换其他输入源。若需撤销，删除锁文件中的该键路径（保留其他策略），运行 `sudo dconf update`，然后重新登录。系统默认值文件可按需要保留或修改。参考：[GNOME dconf 锁定文档](https://help.gnome.org/system-admin-guide/dconf-lockdown.html)。


## Wayland application rendering and keyboard troubleshooting

### Recorded symptoms and scope

On a Lenovo ThinkPad running Ubuntu/GNOME Wayland, the original notes reported fuzzy text and incorrect German key mapping in Emacs and an application launched as `chatgpt`. Native GNOME applications worked correctly. Switching application backends reportedly fixed both symptoms.

This points to an application/backend interaction, but does not prove a single root cause or rule out all scaling, keyboard, and input-method configuration issues. The package behind `/usr/lib/chatgpt/ChatGPT` was not available for inspection here. Its name does not establish that it is an official OpenAI application, nor that the same instructions apply to Codex or other ChatGPT clients.

### Emacs PGTK

Where the Ubuntu release provides this package:

```bash
sudo apt update
sudo apt install emacs-pgtk
```

Quit the old Emacs process, including any daemon if that is what `emacsclient` uses, and launch the intended build. In a graphical Emacs frame, evaluate `window-system`; `pgtk` identifies the PGTK backend. Check `M-x emacs-version` as well. PGTK supports Wayland, but package installation alone does not prove an existing process changed backends. See [GNU Emacs PGTK support](https://www.gnu.org/software/emacs/manual/html_node/efaq/New-in-Emacs-29.html).

### Chromium-based application: test a fresh process

Chromium supports `--ozone-platform=wayland` when built with the Wayland backend. An Electron application can opt into single-instance behavior, so a second launch may hand off to an existing process instead of applying new startup flags. These are framework capabilities, not proof of this unidentified package's implementation. Sources: [Chromium Ozone](https://chromium.googlesource.com/chromium/src.git/+/HEAD/docs/ozone_overview.md), [Electron single-instance API](https://www.electronjs.org/docs/latest/api/app#apprequestsingleinstancelockadditionaldata).

For the specific package recorded in the original note, inspect first:

```bash
command -v chatgpt
pgrep -af '^/usr/lib/chatgpt/ChatGPT'
```

Save work and use the application's **Quit** action. If it remains running and the process name has been confirmed, the original workaround was:

```bash
pkill -x ChatGPT
pgrep -af '^/usr/lib/chatgpt/ChatGPT'
```

`pkill` sends a termination signal to matching processes; it is not a guarantee of graceful shutdown. No output from this `pgrep` only establishes that its particular path pattern has no match.

Once the old instance has exited, test the recorded command on a Wayland session:

```bash
chatgpt --ozone-platform=wayland
```

Test sharpness and German keys such as `Y/Z`, `@`, `€`, `ä`, `ö`, `ü`, and `ß`. The original note reports success on that machine; this was not reproduced in the present review.

### Persist a successful launch option

Only after the flag works, locate the actual desktop entry and make a user override. For the recorded path:

```bash
mkdir -p ~/.local/share/applications
# -i prompts before overwriting an existing personal override.
cp -i /usr/share/applications/chatgpt.desktop \
  ~/.local/share/applications/chatgpt.desktop
```

Edit the copied entry. Preserve its actual executable, quoting, existing arguments, and field codes. For an entry that originally reads `Exec=chatgpt %U`, the edited line is:

```ini
Exec=chatgpt --ozone-platform=wayland %U
```

Desktop entry `Exec` values are not general shell commands. Also inspect any desktop-action `Exec` lines and `DBusActivatable`: a D-Bus activation path may bypass the main `Exec` line. See the [Exec specification](https://specifications.freedesktop.org/desktop-entry-spec/latest/exec-variables.html) and [D-Bus activation](https://specifications.freedesktop.org/desktop-entry-spec/latest/dbus.html).

If installed, validate and refresh the desktop-entry database:

```bash
desktop-file-validate ~/.local/share/applications/chatgpt.desktop
update-desktop-database ~/.local/share/applications
```

Quit the old instance and launch from the menu. To undo the workaround, remove the added flag from the user override, or remove that override if it contains no other customizations. A copied launcher can become stale after package updates, so compare it with the vendor entry when troubleshooting later.
