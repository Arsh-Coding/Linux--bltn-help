# Linux Server Learning: My Command Notebook

My personal reference while turning an old Dell into a self-hosted development and deployment server.
Every command I learn goes here with what it does and why I'd use it.
Questions and answers from my doubt sessions live in [linux-doubts.md](linux-doubts.md).

## My setup

| Machine | Role | OS |
|---|---|---|
| Acer | Development, coding, notes | Windows |
| Dell | Server (hosting, deployment, experiments) | Ubuntu Server |

| Detail | Value |
|---|---|
| Dell username | atrocity |
| Dell hostname | atrocity-old-dell |
| Dell prompt | `atrocity@atrocity-old-dell:~$` |
| Dell LAN IP | `192.168.1.8` (Wi-Fi, DHCP, so it can change until reserved in the router) |

## Roadmap progress

- [ ] Phase 1: Linux administration
  - [x] Assignment 1: Get to know the machine
  - [x] Assignment 2: Storage
    - [x] Inspect disks (`lsblk`, `df`, LVM)
    - [x] Grow `/` from 57G to 115G
    - [x] Mount the old Windows disk read-only
    - [x] Check the old disk's health (`smartctl`: healthy)
    - [x] List what's on the old disk (`find` → `~/old-disk-files.txt`, searched with `grep`). Its future: decide in Phase 5.
  - [x] Installing packages with apt (`ncdu`, `smartmontools` installed)
  - [x] Editing config files (fix the boot delay: ~2 min → ~4 s)
  - [ ] Assignment 3: Managing services ← next
  - [ ] Removing packages (`apt remove`, `purge`, `autoremove`)
  - [ ] Directories, users and permissions
- [ ] Phase 2: Networking and SSH
- [ ] Phase 3: Backend infrastructure (Git, Node.js, Nginx, PostgreSQL, MongoDB, Redis, PM2)
- [ ] Phase 4: Docker and Docker Compose
- [ ] Phase 5: Production deployment (Nginx, HTTPS, UFW, backups)
- [ ] Phase 6: DevOps and automation (GitHub Actions, cron, monitoring)

---

## Terminal survival kit

Learn these before anything else.

| Key / command | What it does |
|---|---|
| `Tab` | Auto-completes commands and file names. Press twice to see all options. |
| `↑` / `↓` | Scroll through previous commands. |
| `Ctrl + C` | Cancel / stop the running command. |
| `q` | Quit full-screen output like `top`, `less`, `man`, `systemctl`. |
| `clear` or `Ctrl + L` | Clear the screen. |
| `command \| less` | Show long output one page at a time (`Space` = next page, `q` = quit). |
| `man command` | Full manual for a command, e.g. `man ls`. |
| `command --help` | Quick help for a command. |
| `history` | List the commands you've run. |
| `sudo command` | Run a command as administrator (root). Asks for your password. |
| `exit` | Log out of the current session. |
| `sudo reboot` | Restart the machine. |
| `sudo poweroff` | Shut the machine down. |

> When you type a password in Linux, nothing appears on screen (no dots, no stars). That's normal. Type it and press Enter.

### Reading man pages

`man` = full manual. `--help` = short summary printed straight to the terminal. Check `--help` first for a quick look, and use `man` for the details.

| Key / command | What it does |
|---|---|
| `Space` / `b` | Next page / previous page |
| `/word` then `Enter` | Search for "word" |
| `n` / `N` | Next / previous search match |
| `g` / `G` | Jump to top / bottom |
| `q` | Quit |
| `man -k keyword` | Search all man pages by keyword (run this outside `man`), e.g. `man -k battery` |

### Long output: no scrolling on the Dell's screen

The Linux console has no scrollback (`Shift` + `PageUp` was removed in kernel 5.9). Send long output somewhere you can move around in. See doubt #7 in [linux-doubts.md](linux-doubts.md).

| Command | What it does |
|---|---|
| `uname -r` | Print the kernel version |
| `command \| less` | Page through output. Same keys as `man`: `/word`, `n`, `g`/`G`, `q` |
| `command 2>&1 \| less` | Same, including error messages. `1` = normal output, `2` = errors, `2>&1` merges them |
| `command \| head -n 20` | First 20 lines only |
| `command \| tail -n 20` | Last 20 lines only |
| `command \| grep word` | Only lines containing "word" |
| `command > file.txt` | Save output to a file instead of the screen (overwrites) |
| `command \| tee file.txt` | Show output on screen and save it to a file |
| `tmux` | Shell with its own scrollback. Scroll: `Ctrl` + `b`, then `[`, then `PageUp`/arrows. `q` stops scrolling, `exit` leaves tmux |

### Text size on the Dell's screen

The console has no zoom (`Ctrl` + `+` does nothing). Change the font instead. See doubt #6 in [linux-doubts.md](linux-doubts.md).

| Command | What it does |
|---|---|
| `ls /usr/share/consolefonts \| grep 32x16` | List the big console fonts available |
| `setfont Lat15-TerminusBold32x16` | Switch to a big bold font now (resets on reboot). Add `sudo` if it complains. |
| `setfont` | Back to the default font |
| `sudo dpkg-reconfigure console-setup` | Pick font and size permanently: `UTF-8` → `Guess optimal character set` → `TerminusBold` → `16x32` |

Over SSH or in Kiro, the Windows app draws the text, so `Ctrl` + `=` / `Ctrl` + `-` / `Ctrl` + `0` work as usual.

### Reading the prompt

```text
atrocity@dell:~$
   |      |   | |
   |      |   | +-- $ = normal user   (# = root user)
   |      |   +---- current folder    (~ = your home folder, /home/atrocity)
   |      +-------- hostname          (the machine's name)
   +--------------- username
```

---

## Saved commands

### Battery status

```bash
upower -i $(upower -e | grep BAT)
```

- `upower -e` lists power devices (battery, AC adapter).
- `| grep BAT` passes that list to `grep`, which keeps only lines containing "BAT".
- `$( ... )` runs the inner command first and drops its result into the outer command.
- `upower -i <device>` shows details for that one device.

Look at: `state` (charging / discharging / fully-charged), `percentage` (current charge), `capacity` (battery health compared to when it was new).

Other useful lines:
- `charge-start-threshold` / `charge-end-threshold`: the battery starts charging below the first number and stops at the second. My Dell is set to 75% / 80%. That's ideal for a laptop that stays plugged in 24/7, because sitting at 100% all the time wears out and can swell an old battery.
- `History (rate)`: power draw in watts. My Dell idles at about 5 W, which costs very little to run 24/7.

Shortcut (same output, built into newer upower versions):

```bash
upower -b
```

- `-b` / `--battery`: shows details for all battery devices without needing `-e` and `grep`. See doubt #1 in [linux-doubts.md](linux-doubts.md).

Other upower options:

| Command | What it does |
|---|---|
| `upower -e` | List every power device path (battery, AC adapter, etc.) |
| `upower -d` | Dump details for all devices, not just the battery |
| `upower -v` | Show the upower version |

If you get `command not found`:

```bash
sudo apt install upower
```

Energy lines (my Dell, on battery):

| Line | Value | Meaning |
|---|---|---|
| `energy` | 10.17 Wh | Charge left right now. |
| `energy-full` | 26.94 Wh | What a full charge holds today. |
| `energy-full-design` | 42.00 Wh | What it held when new. 26.94 ÷ 42.00 = 64% = `capacity`. |
| `energy-rate` | 5.31 W | Current power draw. |
| `time to empty` | 1.9 hours | 10.17 Wh ÷ 5.31 W ≈ 1.9 h. |

Is the charger actually connected?

```bash
upower -i $(upower -e | grep line_power)   # look for "online: yes"
ls /sys/class/power_supply/                 # lists power devices, e.g. AC  BAT0
cat /sys/class/power_supply/AC/online       # 1 = plugged in, 0 = not
```

`/sys` is a virtual folder where the kernel shows hardware info as tiny text files. Nothing in it is stored on disk.

No-install alternative (shows current charge % only):

```bash
cat /sys/class/power_supply/BAT*/capacity
```

---

## Phase 1: Linux administration

### Assignment 1: Get to know the machine

Run each command on the Dell. Read the output before moving on.

#### 1. Operating system

```bash
cat /etc/os-release
```

`cat` prints a file's contents. `/etc` is where Linux keeps system configuration. This file describes the installed OS.
Look for: `PRETTY_NAME` and `VERSION` (e.g. `24.04 LTS`). LTS = Long Term Support, which means 5 years of security updates.

#### 2. CPU

```bash
lscpu
```

Lists processor details. Output is long, so try `lscpu | less`.
Look for:
- `Architecture`: `x86_64` (Intel/AMD 64-bit)
- `Model name`: the exact CPU
- `CPU(s)`: total logical cores (threads)
- `Thread(s) per core`: 2 means Hyper-Threading is on
- `Virtualization`: `VT-x` (Intel) or `AMD-V`. Needed if you ever want to run virtual machines.

#### 3. RAM

```bash
free -h
```

`-h` = human-readable (GB/MB instead of bytes).
Look for: `total` and `available` on the `Mem:` row. Ignore `free`; Linux uses spare RAM as cache and hands it back when needed, so `available` is the real number. `Swap:` is disk space used as emergency RAM.

#### 4. Disks

```bash
lsblk
```

Lists block devices (disks and partitions) as a tree.
Look for:
- `NAME`: `sda` (SATA disk) or `nvme0n1` (NVMe SSD). Partitions are `sda1`, `sda2`...
- `SIZE` and `TYPE` (`disk`, `part`, `lvm`)
- `MOUNTPOINTS`: where it's attached in the filesystem. `/` is the root of everything.

If you see `ubuntu--vg-ubuntu--lv` and it's much smaller than the disk, the installer left space unused. That's a common default and we'll fix it later.

Other columns and rows:
- `loop0`, `loop1`...: snap packages. Each snap is a compressed read-only image mounted as a fake disk. Not real disks, ignore them.
- `RM`: 1 = removable (USB stick). `RO`: 1 = read-only.
- `MAJ:MIN`: internal device numbers the kernel uses. Ignore.

#### 5. Network

```bash
ip addr
```

Lists network interfaces and their IP addresses.
Look for:
- `lo`: loopback (`127.0.0.1`), the machine talking to itself. Always present.
- Ethernet interface: usually `enp...` or `eno...`
- Wi-Fi interface: usually `wlp...` or `wlan0`
- `inet 192.168.x.x/24` on the real interface: the Dell's private LAN IP. Write it down.
- `state UP` means the interface is connected.

Shortcut that prints just the IP:

```bash
hostname -I
```

How to read my output:

| Part | Example from my Dell | Meaning |
|---|---|---|
| Interface name | `enp2s0`, `wlp3s0` | `en` = Ethernet, `wl` = wireless. `p2s0` = where it sits on the motherboard (PCI bus 2, slot 0). |
| `<...UP,LOWER_UP>` | `wlp3s0` | `UP` = turned on in software. `LOWER_UP` = physically connected. |
| `NO-CARRIER` / `state DOWN` | `enp2s0` | Turned on, but no cable plugged in. |
| `link/ether` | `10:5b:ad:...` | MAC address: the hardware ID of the network card. |
| `altname` | `wlx<wifi-mac>` | An alternative name for the same interface: `wlx` + the MAC address without colons. |
| `inet 192.168.1.8/24` | `wlp3s0` | IPv4 address. `/24` means the first 3 numbers are the network (`192.168.1.x`), so the LAN has up to 254 devices. Same as subnet mask `255.255.255.0`. |
| `brd 192.168.1.255` | | Broadcast address: sends to every device on the LAN at once. |
| `dynamic` + `valid_lft 83197sec` | | The router handed out this IP via DHCP. The lease lasts ~23 hours, then gets renewed. The IP can change. |
| `inet6 2401:...` `scope global` | | A public IPv6 address. Reachable from the internet unless the router or a firewall blocks it. Don't post it publicly. |
| `inet6 fe80::...` `scope link` | | Link-local IPv6. Only works on the local network. Every interface gets one. |
| `mtu 1500` | | Biggest packet size in bytes. 1500 is standard. |

My result: Wi-Fi (`wlp3s0`) is connected at `192.168.1.8`. Ethernet (`enp2s0`) has no cable.

My own MAC is low risk to share, but the router's MAC (the BSSID) can reveal where I live through Wi-Fi location databases. `iw dev wlp3s0 link` shows which network the Wi-Fi is connected to, with the router's MAC on the `Connected to` line. Hide that line before sharing. See doubt #11 in [linux-doubts.md](linux-doubts.md).

#### 6. Running services

```bash
systemctl --type=service --state=running
```

`systemctl` controls systemd, the program that starts and manages background services. Press `q` to exit.
Look for: `ssh.service` (remote login), `cron.service` (scheduled jobs), `systemd-networkd` / `NetworkManager` (networking).

What's running on my Dell (22 services):

| Service | What it does | Needed? |
|---|---|---|
| `chrony` | Keeps the clock accurate over the internet (NTP). HTTPS certificates, logs and JWT tokens all depend on correct time. | Yes |
| `cron` | Runs scheduled jobs (backups, cleanups). Used in Phase 6. | Yes |
| `dbus` | Message bus that lets system programs talk to each other. | Yes |
| `getty@tty1` | Shows the `login:` prompt on the screen. | Yes |
| `polkit` | Decides which users are allowed to do privileged actions. | Yes |
| `systemd-logind` | Manages logins and sessions. Also handles the power button and laptop lid. | Yes |
| `systemd-udevd` | Detects hardware when it appears (USB sticks, disks). | Yes |
| `user@1000` | My personal service manager. `1000` is my user ID (the first user created). | Yes |
| `systemd-journald` | Collects all logs. Read them with `journalctl`. | Yes |
| `rsyslog` | Also writes logs as text files in `/var/log/` (e.g. `/var/log/syslog`). | Yes |
| `systemd-networkd` | Manages network interfaces. Configured by netplan (`/etc/netplan/`). | Yes |
| `systemd-resolved` | DNS: turns names like `google.com` into IP addresses. | Yes |
| `networkd-dispatcher` | Runs scripts when the network changes. | Yes |
| `netplan-wpa-wlp3s0`, `wpa_supplicant` | Handle Wi-Fi login (the password handshake). | Yes, on Wi-Fi |
| `snapd` | Manages snap packages. | Yes |
| `unattended-upgrades` | Installs security updates automatically. | Yes |
| `thermald` | Intel thermal control, keeps the CPU from overheating. | Yes |
| `upower` | Battery info (what `upower -b` asks). | Yes, on a laptop |
| `udisks2` | Disk management helper. | Optional |
| `ModemManager` | Controls mobile broadband (SIM card) modems. | No, no modem |
| `multipathd` | Multiple paths to enterprise SAN storage. | No, not on a laptop |

Not in the list: `ssh.service`. On recent Ubuntu, SSH uses socket activation: `ssh.socket` listens on port 22 and only starts `ssh.service` when someone connects. So check the socket (see Assignment 3).

#### 7. Live resource usage

```bash
top
```

A live dashboard of CPU, RAM and processes. Press `q` to quit.
Useful keys while it's running:
- `P`: sort by CPU
- `M`: sort by memory
- `1`: show each CPU core separately

Header lines:
- `load average`: three numbers (1, 5, 15 minute averages). Lower than your core count = healthy.
- `Tasks`: number of processes
- `%Cpu(s)`: `us` = user programs, `sy` = system, `id` = idle
- `MiB Mem`: RAM in megabytes

How to read my `top` (idle Dell, 1 hour after boot):

| Line | My value | Meaning |
|---|---|---|
| `up 1:06` | | Time since boot (1 h 6 min). |
| `1 user` | | Logged-in sessions. |
| `load average` | `0.03, 0.04, 0.00` | Average number of processes wanting the CPU over the last 1 / 5 / 15 minutes. With 8 threads, 8.0 = fully busy. 0.03 = basically asleep. |
| `Tasks` | `178 total, 1 running, 177 sleeping, 0 zombie` | Sleeping = waiting for work, which is normal. Zombie = a finished process its parent never cleaned up. Should stay 0. |
| `%Cpu(s)` | `99.9 id` | `us` user programs, `sy` kernel, `ni` low-priority programs, `id` idle, `wa` waiting for disk (high = disk is the bottleneck), `hi`/`si` hardware/software interrupts, `st` stolen by a hypervisor (VMs only). |
| `MiB Mem` | `7319 total, 577 used, 472 buff/cache` | `buff/cache` is RAM borrowed for disk caching and given back when needed. |
| `avail Mem` | `6741.9` | The real "how much can I still use" number. |
| `MiB Swap` | `0.0 used` | Good. Swap in use means RAM is running short. |

Process columns:

| Column | Meaning |
|---|---|
| `PID` | Process ID. Unique number, used with `kill`. |
| `USER` | Who owns it. |
| `PR` / `NI` | Priority / niceness. NI goes from -20 (most important) to 19 (least). `rt` = real-time, top priority. |
| `VIRT` | Memory the process has reserved. Mostly ignore. |
| `RES` | RAM it really uses right now (KiB). This is the one to watch. |
| `SHR` | Part of `RES` shared with other processes (libraries). |
| `S` | State: `R` running, `S` sleeping, `I` idle kernel thread, `D` waiting on disk, `Z` zombie. |
| `%CPU` | Share of one CPU thread. Can go over 100% on multi-core (800% = all 8 threads). |
| `%MEM` | Share of total RAM. |
| `TIME+` | Total CPU time used since it started. |
| `COMMAND` | Program name. |

What the list showed:
- `PID 1` is `systemd`, the first process. Every other process descends from it.
- Most rows (`kworker`, `ksoftirqd`, `migration`, `cpuhp`, `idle_inject`, `rcu_...`) are kernel threads: the kernel's own workers. They show `0` memory and low PIDs. Many come in 8 copies (`/0` to `/7`), one per CPU thread.
- `top` itself shows up at 0.3% CPU. Measuring the system costs a tiny bit of it.

Next tool: `htop` is a friendlier `top` with colours, mouse support and per-core bars. Run `htop` (if missing: `sudo apt install htop`). Inside it, `F10` or `q` quits and `Shift + K` hides kernel threads.

### My Dell's specs (from Assignment 1)

| Item | Value |
|---|---|
| Ubuntu version | 26.04.1 LTS "Resolute Raccoon" (security updates until 2031) |
| CPU model | Intel Core i5-8250U (8th gen, 15 W laptop chip), 1.6 GHz base / 3.4 GHz boost |
| Cores / threads | 4 cores / 8 threads (Hyper-Threading on) |
| Virtualization (VT-x / AMD-V) | VT-x supported (`vmx` flag present) |
| RAM total | 7.1 GiB usable (8 GB installed, some reserved for built-in graphics). 6.6 GiB available at idle. 4 GiB swap. |
| Disk 1: `sdb` | LITEON CV8-8E128, 128 GB SATA **SSD** (`ROTA 0`). Holds Ubuntu. `/` grown to 115G (7.5G used, 102G free). |
| Disk 2: `sda` | WDC WD10SPZX (WD Blue), 1 TB 2.5" 5400 RPM laptop **HDD** (`ROTA 1`). 3 old Windows NTFS partitions labelled "New Volume", holding ~390 GB of old files. SMART: `PASSED`, 0 bad sectors, 11,263 power-on hours, 28 °C. |
| Network interface (Ethernet / Wi-Fi) | Wi-Fi `wlp3s0` in use (Qualcomm Atheros card, driver `ath10k_pci`). Ethernet `enp2s0` present, no cable. |
| LAN IP | `192.168.1.8/24` (DHCP) |
| Battery health (`capacity`) | 64% of original. Charge limit set to 75–80%. Idle draw ~5 W. |

The `Vulnerabilities:` section of `lscpu` lists known CPU security bugs (Spectre, Meltdown, etc.) and how the kernel works around them. Long lists like this are normal for an 8th-gen Intel chip. Nothing to do for a home lab.

### What this hardware can handle

CPU is plenty for this roadmap. RAM (8 GB) is the real limit, so it decides what we run.

Rough RAM budget for the final server:

| Service | Typical RAM |
|---|---|
| Ubuntu itself | ~0.6 GB (575 MiB measured at idle) |
| Docker engine | 0.1–0.2 GB |
| Nginx | < 50 MB |
| Node.js API (each) | 100–300 MB |
| Redis (small data) | < 100 MB |
| PostgreSQL | 100–500 MB |
| MongoDB | 0.5–1.5 GB once capped. By default it grabs ~50% of (RAM − 1 GB) as cache, so we'll limit it. |
| Prometheus + Grafana | 0.3–1 GB |

Total: about 3–5 GB, so the full roadmap fits.

Doesn't fit well on 8 GB: self-hosted GitLab (needs ~4 GB by itself), a Proxmox lab with many VMs, Windows VMs. That's why the roadmap uses GitHub Actions for CI/CD.

### Boot time: why is startup slow?

systemd (the program that starts all services at boot) records how long each step took. See doubt #2 in [linux-doubts.md](linux-doubts.md).

| Command | What it does |
|---|---|
| `systemd-analyze` | Total boot time split into `firmware` (BIOS), `loader` (GRUB), `kernel`, `userspace` (services). |
| `systemd-analyze blame \| head -n 15` | The 15 slowest services. `head -n 15` keeps only the first 15 lines. Services start in parallel, so slow ≠ always blocking. |
| `systemd-analyze critical-chain` | The chain of units that actually delayed boot. `@` = when it was ready, `+` = how long it took. |
| `lsblk -d -o NAME,ROTA,SIZE,MODEL` | Disk info only (`-d` = no partitions). `ROTA 1` = spinning HDD, `0` = SSD. |

### Assignment 2: Storage

#### How my disks are laid out

```text
sdb (119 GB)  - the Ubuntu disk
├─ sdb1  1 GB    -> /boot/efi   EFI partition: the bootloader the BIOS starts
├─ sdb2  2 GB    -> /boot       Linux kernel files
└─ sdb3  116 GB  -> LVM
     └─ ubuntu-lv  58 GB -> /   everything else (programs, configs, /home, Docker later)
                                 ~58 GB of sdb3 is still unused

sda (931 GB)  - old Windows disk, nothing mounted
├─ sda1  16 MB   "Microsoft Reserved" partition. Windows creates this, so this disk came from Windows.
├─ sda2  288 GB  probably an old drive letter (D:, E:...). May still hold files.
├─ sda3  200 GB
└─ sda4  443 GB
```

Confirmed with `lsblk -d -o NAME,ROTA,SIZE,MODEL` and `lsblk -f`:

```text
sdb  LITEON CV8-8E128  SSD (ROTA 0)  119 GB
├─ sdb1  vfat (FAT32)   /boot/efi   1% used
├─ sdb2  ext4           /boot      10% used
└─ sdb3  LVM2_member
     └─ ubuntu-lv  ext4  /          13% used, 46.5 GB free

sda  WDC WD10SPZX      HDD (ROTA 1)  931 GB
├─ sda1  (none)         Microsoft Reserved
├─ sda2  ntfs  "New Volume"  288 GB
├─ sda3  ntfs  "New Volume"  200 GB
└─ sda4  ntfs  "New Volume"  443 GB
```

What this tells me:
- `sda` has only data partitions: no Windows system (`C:`), no EFI, no recovery partition. So Windows used to live on the SSD (now replaced by Ubuntu), and this HDD was the D:/E:/F: data drive. My old files may still be on it.
- SSD = fast: keep the OS, Docker and databases there. HDD = big but slow (spinning laptop disk): good for backups, media and archives.
- Snap loops show `squashfs` at `100%` used. That's normal: each snap is a compressed, read-only image, always "full" by design.

Rule: never format a disk until you've checked what's on it.

#### LVM in one picture

LVM (Logical Volume Manager) is a flexible layer between partitions and filesystems. It lets you resize space without repartitioning.

```text
sdb3          Physical Volume (PV)  raw space handed to LVM
  ubuntu-vg   Volume Group (VG)     a pool of space (116 GB)
    ubuntu-lv Logical Volume (LV)   a resizable "partition" cut from the pool (58 GB)
      /       mount point           where you actually use it
```

`lsblk` shows `ubuntu--vg-ubuntu--lv` with double dashes. That's the same thing as `/dev/ubuntu-vg/ubuntu-lv`. The device mapper doubles any `-` inside a name so it can use a single `-` to separate the VG name from the LV name.

#### Command reference

| Command | What it does |
|---|---|
| `lsblk -f` | `lsblk` plus filesystem info: `FSTYPE` (`ext4`, `vfat`, `ntfs`, `LVM2_member`), `LABEL`, `UUID`, free space, `FSUSE%` |
| `lsblk -d -o NAME,ROTA,SIZE,MODEL` | One line per disk (`-d` = no partitions, `-o` = pick columns). `ROTA 1` = spinning HDD, `0` = SSD |
| `df -h` | Disk free: size, used, available and `Use%` for each mounted filesystem |
| `df -hT` | Same, plus a `Type` column (`ext4`, `vfat`, `tmpfs`) |
| `df -h /` | Only the filesystem that holds `/` |
| `sudo pvs` | LVM physical volumes: which partitions feed LVM, `PFree` = unused |
| `sudo vgs` | LVM volume groups: `VSize` = total pool, `VFree` = unused space in the pool |
| `sudo lvs` | LVM logical volumes and their sizes |
| `du -sh <folder>` | Disk usage of one folder (`-s` = total only, `-h` = human-readable) |
| `sudo du -xh --max-depth=1 / \| sort -h` | Size of every top-level folder, smallest to largest |
| `du -sh *` | Size of each item in the current folder |
| `du -h --max-depth=1 ~ \| sort -h` | Each folder in home, sorted by size. `-d 1` is the short form of `--max-depth=1` |
| `du -ah <folder>` | `-a` = include individual files, not just folders |
| `du -sh <folder> 2>/dev/null` | Hide `Permission denied` errors. `2` = error output, `/dev/null` = discard it |

#### Step 1: SSD or HDD?

```bash
lsblk -d -o NAME,ROTA,SIZE,MODEL
```

`ROTA` = "rotational". `1` = spinning hard disk, `0` = SSD. `MODEL` is the drive's brand and model.
Expected: `sdb` (Ubuntu) is an SSD. `sda` (1 TB) could be either.

#### Step 2: What filesystem is on each partition?

```bash
lsblk -f
```

A filesystem is the format that organizes files on a partition. Common ones:

| `FSTYPE` | Used by |
|---|---|
| `vfat` | EFI boot partition. FAT32 is readable by every BIOS. |
| `ext4` | Linux's standard filesystem |
| `LVM2_member` | A partition handed over to LVM |
| `ntfs` | Windows drives (C:, D:...) |
| _(blank)_ | No filesystem, e.g. the 16 MB Microsoft Reserved partition |

Also shown: `LABEL` (the name you gave a drive in Windows, like "Data"), `UUID` (a unique ID Linux uses to find a partition even if its name changes), `FSAVAIL` and `FSUSE%` (only for mounted ones).

#### Step 3: How full is each mounted filesystem?

```bash
df -hT
```

Columns: `Filesystem` (device), `Type`, `Size`, `Used`, `Avail`, `Use%`, `Mounted on`.

Rows to ignore: `tmpfs` (lives in RAM and is cleared on reboot) and `efivarfs` (BIOS settings exposed as files).
Rows that matter: `/dev/mapper/ubuntu--vg-ubuntu--lv` on `/`, `/dev/sdb2` on `/boot`, `/dev/sdb1` on `/boot/efi`.

My result before growing: `/` is `57G` size, `7.5G` used, `47G` available, `14%`.

The RAM-based (`tmpfs`) rows on my Dell:

| Mounted on | What it is |
|---|---|
| `/run` | Runtime data for running services (PID files, sockets). Recreated every boot. |
| `/dev/shm` | Shared memory that programs use to pass data to each other quickly. |
| `/tmp` | Temporary files. On this Ubuntu it lives in RAM, so everything in `/tmp` disappears on reboot. Never keep anything important there. |
| `/run/user/1000` | My own runtime folder (user ID 1000). |
| `/run/credentials/...` | Secrets systemd hands to individual services. |

Cleaner view without the RAM rows:

```bash
df -h -x tmpfs -x efivarfs      # -x = exclude this filesystem type
df -h -t ext4                   # -t = only show this filesystem type
```

Options are case-sensitive. `df -hT` (capital T) adds a Type column. `df -ht` (lowercase t) fails with "option requires an argument", because `-t` expects a type after it, like `-t ext4`.

`Size` here (~57G) is a little smaller than `lsblk` (58.1G). The filesystem keeps some room for its own bookkeeping and reserves 5% for root, so the system can still work when a normal user fills the disk.

#### Step 4: How much space is LVM holding back?

```bash
sudo pvs
sudo vgs
sudo lvs
```

Expected: `pvs` shows `/dev/sdb3` with `PFree` ~58g. `vgs` shows `ubuntu-vg` with `VSize` ~116g and `VFree` ~58g. `lvs` shows `ubuntu-lv` at ~58g.

#### Step 5: Grow `/` to use the whole SSD

Only run this if `sudo vgs` shows `VFree` around 58g.

```bash
sudo lvextend -r -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
```

- `lvextend`: make a logical volume bigger
- `-l +100%FREE`: add all the free space left in the volume group
- `-r`: also resize the filesystem inside it, so the new space is actually usable. Without `-r` the LV grows but `df` still shows the old size.
- Safe while the system is running, and no data is touched. It's one-way (shrinking `/` later is hard), which is fine here.

Expected output: `Size of logical volume ubuntu-vg/ubuntu-lv changed from <58.10 GiB ... to <116.20 GiB`, then a line from `resize2fs` (the ext4 resize tool) saying the filesystem is now bigger.

Check:

```bash
df -h /          # Size ~114G instead of ~57G
sudo vgs         # VFree now 0
lsblk            # ubuntu--vg-ubuntu--lv now ~116G
```

#### Step 6: What's using the space?

`df` answers "how full is the disk?". `du` answers "which folders are filling it?".

```bash
du -sh ~
sudo du -xh --max-depth=1 / | sort -h
```

- `du -sh ~`: total size of your home folder.
- `-x`: stay on this one filesystem. Skips `/boot`, `/proc`, `/sys` and other mounts that aren't really on this disk.
- `--max-depth=1`: one line per top-level folder instead of every file.
- `| sort -h`: sort by size, understanding `K`, `M`, `G`. The biggest ends up at the bottom.

On a fresh server, expect `/usr` (installed programs) and `/var` (logs, caches, snaps) to be the biggest. Later, Docker images will live in `/var/lib/docker` and grow fast. That's why growing `/` matters.

#### Step 7: Look inside the old Windows disk (read-only)

Mounting = attaching a partition to a folder so you can open its files. Windows gives each drive a letter (`C:`, `D:`). Linux has one tree starting at `/`, and you attach a disk onto an empty folder anywhere in it. By convention:
- `/mnt`: for disks you mount by hand
- `/media`: where removable drives (USB sticks) usually appear

Mount all three NTFS partitions read-only (`-o ro`), so nothing on them can be changed:

```bash
sudo mkdir -p /mnt/sda2 /mnt/sda3 /mnt/sda4
sudo mount -t ntfs3 -o ro /dev/sda2 /mnt/sda2
sudo mount -t ntfs3 -o ro /dev/sda3 /mnt/sda3
sudo mount -t ntfs3 -o ro /dev/sda4 /mnt/sda4
df -h /mnt/sda2 /mnt/sda3 /mnt/sda4
```

- `mkdir -p`: create the folders. `-p` = no error if they already exist (and create parent folders if needed).
- `mount -t ntfs3`: use the kernel's NTFS driver. `-o ro` = read-only.
- `df -h` on the mount folders: how full each old partition is. That's the fastest way to tell an empty partition from one full of files.

Look around:

```bash
ls /mnt/sda2
ls /mnt/sda3
ls /mnt/sda4
sudo du -sh /mnt/sda2/* | sort -h      # size of each folder. Slow on an HDD, Ctrl + C to stop.
ls "/mnt/sda2/Some Folder"             # names with spaces need quotes
```

- `$RECYCLE.BIN` and `System Volume Information` are Windows system folders. Ignore them.
- `ls` shows names with spaces or symbols inside `'single quotes'`. The quotes aren't part of the name.

#### What read-only allows

Read-only (`-o ro`) blocks changes, not access. See doubt #3 in [linux-doubts.md](linux-doubts.md).

| Action | Allowed? |
|---|---|
| Browse folders, list files | Yes |
| Open and read files | Yes |
| Copy files *from* the disk to somewhere else | Yes (copying only reads the original) |
| Edit, rename, move, delete, create | No: `Read-only file system` error, even with `sudo` |

Prove it (harmless, it just fails):

```bash
touch /mnt/sda2/test.txt      # touch = create an empty file
# touch: cannot touch '/mnt/sda2/test.txt': Read-only file system
```

Read-only is set on the mount itself, so it beats file permissions. Not even root can write to it.

Moving around and opening files:

| Command | What it does |
|---|---|
| `cd /mnt/sda2` | Go into a folder |
| `pwd` | Print the folder you're in |
| `ls -lh` | List with details: size (`-h` = KB/MB/GB), date modified |
| `ls -a` | Also show hidden files (names starting with `.`) |
| `cd "My Folder"` | Folder names with spaces need quotes. Or type `cd My` then `Tab`, and the name is filled in for you. |
| `cd ..` | Up one level |
| `cd -` | Back to the previous folder |
| `cd ~` or just `cd` | Back home |
| `file <name>` | What kind of file it is (text, JPEG image, PDF...), whatever its extension says |
| `less <name>` | Read a text file page by page (`q` to quit) |
| `head -n 20 <name>` | First 20 lines of a text file |
| `find /mnt/sda2 -iname "*.pdf"` | Search by name. `-iname` ignores upper/lower case, `*` = anything. Slow on the HDD. |
| `cp <file> ~/` | Copy a file to my home folder on the SSD |
| `cp -r <folder> ~/` | Copy a whole folder (`-r` = recursive, include everything inside) |

The server has no screen apps, so photos, videos, PDFs and Word files can't be viewed on it directly. You can identify them with `file` and list them with `ls`. Viewing them means getting them to the Acer: over the network once SSH is set up (Phase 2, e.g. `scp` or WinSCP), or on a USB stick.

The mount is temporary. After a reboot, `/mnt/sda2` is just an empty folder again until you run `mount` again. Mounting at boot needs a line in `/etc/fstab`.

#### Viewing text files with vi

`vi` (on Ubuntu, really `vim`) is the editor found on every Linux server. It has modes, which is what confuses everyone at first. See doubt #10.

| Mode | How to get there | What keys do |
|---|---|---|
| Normal (starts here) | `Esc` | Keys are commands: move, search, delete |
| Insert | `i` | Keys type text, like a normal editor |
| Command line | `:` from Normal | Type commands like `q` or `w` at the bottom, then `Enter` |

| Keys (in Normal mode) | What they do |
|---|---|
| Arrow keys (or `h` `j` `k` `l`) | Move |
| `gg` / `G` | Jump to top / bottom |
| `/word` then `Enter` | Search, `n` = next match |
| `:q` | Quit |
| `:q!` | Quit and throw away any changes. The emergency exit: `Esc`, then `:q!`, then `Enter`. |
| `:w` / `:wq` | Save / save and quit. Fails on the read-only mount (`E45: 'readonly' option is set` or "can't open file for writing"). That's the protection working. |

- For just reading, `view file.txt` opens vi in read-only mode, or use `less file.txt`.
- vi is for text only. Word, PDF, image and video files show as garbage. Check first with `file <name>`.

#### Make a list of everything on the old disk

Scanning the HDD is slow, so scan once, save the list to a file on the SSD, then search the file instantly.

```bash
find /mnt/sda2 /mnt/sda3 /mnt/sda4 -type f > ~/old-disk-files.txt
wc -l ~/old-disk-files.txt               # how many files
less ~/old-disk-files.txt                # scroll through the list (q to quit)
grep -i "resume" ~/old-disk-files.txt    # search the list, ignoring case
```

- `find <folders> -type f`: every file (not folders) under those folders, with its full path
- `>`: redirect. Send the output into a file instead of the screen. Overwrites the file if it exists. `>>` adds to the end instead.
- Nothing shows while it runs, because the output is going into the file. Wait for the prompt to come back.
- `wc -l`: word count, `-l` = count lines. One line = one file.
- `grep -i`: show only lines containing the word. `-i` = ignore upper/lower case.
- If you see `Permission denied`, put `sudo` in front of `find`. The `>` is handled by my own shell, so the list file is still owned by me, not root.

More ways to look:

```bash
find /mnt/sda2 -maxdepth 2 -type d               # folder structure, 2 levels deep
sudo apt install tree                            # optional: draws folders as a tree
tree -d -L 2 /mnt/sda2                           # -d = folders only, -L 2 = 2 levels
find /mnt -type f -iname "*.jpg" | wc -l         # how many JPG photos
find /mnt -type f -size +1G                      # files bigger than 1 GB
grep -ic "\.pdf$" ~/old-disk-files.txt           # count PDFs in the saved list (-c = count)
```

- `"\.pdf$"`: a pattern. `\.` = a real dot, `$` = end of the line, so it only matches names ending in `.pdf`.

Privacy: these lists contain my personal file and folder names. Keep them on the Dell, and never add them to the public GitHub repo.

Unmount when done:

```bash
cd ~
sudo umount /mnt/sda2 /mnt/sda3 /mnt/sda4
```

- The command is `umount` (no "n").
- `target is busy` = you (or a program) are still inside that folder. `cd ~` first.

If something goes wrong:
- `unknown filesystem type 'ntfs3'`: install the other NTFS driver with `sudo apt install ntfs-3g`, then use `-t ntfs-3g` instead of `-t ntfs3`.
- A warning that the volume is "dirty": Windows wasn't shut down cleanly (common with Fast Startup). Read-only still works for looking.

My result (mounted fine with `ntfs3`):

```text
/dev/sda2   289G  140G used  150G free  49%   /mnt/sda2
/dev/sda3   200G   97G used  104G free  49%   /mnt/sda3
/dev/sda4   444G  153G used  291G free  35%   /mnt/sda4
```

About 390 GB of old files, and this disk is probably their only copy. Don't wipe anything until I know what they are and whether the disk is healthy.

Options once I know what's there:

| Option | How | Trade-off |
|---|---|---|
| A. Keep as-is | Leave the HDD as NTFS and don't use it for the server | Safest for the files, but 1 TB sits unused. NTFS doesn't support Linux permissions, so it's a poor home for server data. |
| B. Copy off, then wipe | Copy what matters to the Acer, an external drive or the cloud. Then turn `sda` into one ext4 Linux disk. | Cleanest result. Needs somewhere to put up to 390 GB. |
| C. Shuffle and free one partition | Move `sda3`'s 97 GB into `sda4`'s 291 GB of free space. Then reformat only `sda3` (200 GB) as ext4 for the server. | No extra drive needed. But the files still have only one copy, on an old disk. |

Whatever I choose, check the disk's health first (`smartctl`, below).

Health checked: healthy (see "My HDD's result" under smartctl).

Current plan: D. Decide later. Leave the old disk untouched. The SSD's 102 GB free is enough for Phases 1–4. Revisit in Phase 5, when database backups need a home. Nothing gets formatted or deleted unless I decide to and type the commands myself (see doubt #8).

#### How Linux finds disks at boot: `/etc/fstab`

```bash
cat /etc/fstab
```

`/etc/fstab` (filesystem table) lists what gets mounted automatically at boot. One line per mount:

```text
<device>                       <mount point>  <type>  <options>  <dump>  <pass>
/dev/disk/by-uuid/2ad5d35a-... /boot          ext4    defaults   0       1
```

- Devices are listed by UUID, not by `sda2` / `sdb2`. Names like `sda` and `sdb` are handed out in the order disks are detected, and they can swap (for example after plugging in a USB drive). A UUID belongs to the filesystem itself and never changes.
- `dump`: old backup flag, always `0`. `pass`: order for the boot-time disk check (`1` = `/`, `2` = others, `0` = never).
- A wrong line in `fstab` can stop the server from booting. Always back it up before editing.

### Installing packages with apt

`apt` is Ubuntu's package manager: it downloads software from Ubuntu's servers, installs it, and keeps it updated. Like an app store for the terminal.

| Command | What it does |
|---|---|
| `sudo apt update` | Refresh the list of available packages and versions. Run it before installing. It doesn't upgrade anything. |
| `sudo apt upgrade` | Install newer versions of everything already installed. |
| `apt search <word>` | Find packages by name or description. |
| `apt show <pkg>` | Details: version, size, description, and what it depends on. |
| `sudo apt install <pkg>` | Install a package and everything it needs. |
| `sudo apt install --no-install-recommends <pkg>` | Install only what's required, skipping "recommended" extras. |
| `sudo apt remove <pkg>` | Uninstall, but keep its config files. |
| `sudo apt purge <pkg>` | Uninstall and delete its config files too. |
| `sudo apt autoremove` | Remove leftover dependencies nothing needs anymore. |
| `apt list --installed` | Everything installed. Pipe to `grep` to search, e.g. `apt list --installed \| grep ncdu`. |
| `which <command>` | Where a command's program lives, e.g. `which ncdu` → `/usr/bin/ncdu`. |

Recommends: packages can "recommend" extras that aren't strictly needed. `apt` installs them by default. `smartmontools` recommends a mail program, which pulls in a full mail server (postfix) and pops up a setup screen. `--no-install-recommends` avoids that.

#### ncdu: interactive disk usage

```bash
sudo apt update
sudo apt install ncdu
sudo ncdu -rx /             # the SSD
sudo ncdu -r /mnt/sda2      # one old Windows partition (slow, shows progress)
```

- `-x`: stay on one filesystem (same as `du -x`)
- `-r`: read-only mode, so deleting is disabled. Always use it while exploring.
- `-rr`: also blocks opening a shell from inside ncdu (`b` key).
- Why not `du`? `du` rescans every time you look at a different folder. `ncdu` scans once, shows progress, then lets you browse the results. See doubt #5 in [linux-doubts.md](linux-doubts.md).

Keys: `↑` `↓` move, `Enter` open a folder, `←` go back, `i` info about the selected item, `s` sort by size, `n` sort by name, `q` quit, `?` help. Without `-r`, `d` deletes the selected item.

ncdu measures, it doesn't open files. To look at files without losing the scan (see doubt #6):

- `b`: opens a shell inside the folder you're viewing. Use `ls -lh`, `file`, `less` there, then type `exit` to return to ncdu with the results still loaded.
- Because ncdu was started with `sudo`, that shell runs as root (prompt ends in `#`). The read-only mount protects `/mnt/sda2`, but nothing protects the rest of the system. Look, don't change, then `exit`.

Quitting (`q`) throws the scan away. Running it again rescans, but usually much faster: Linux keeps recently read folder info cached in RAM until a reboot or until the memory is needed.

Save a scan to a file, browse it any time without rescanning:

```bash
sudo ncdu -o ~/old-disk.json /mnt     # scan all three old partitions once, save to a file
ncdu -f ~/old-disk.json               # open the saved results instantly
```

- `-o <file>`: scan and save the results instead of opening the browser
- `-f <file>`: browse saved results. No scan, works even after a reboot or unmount.
- Scanning `/mnt` (no `-x`) covers `sda2`, `sda3` and `sda4` in one tree.
- It's a snapshot. Fine here, because a read-only disk can't change.

Where the file goes (see doubt #7):

- `~` = my home folder, so the file is `/home/atrocity/old-disk.json`, on the SSD. Check with `echo ~`.
- With `sudo`, my own shell expands `~` before `sudo` runs, so it's still my home, not root's (`/root`).
- It holds only names, sizes and dates. It's a map of the old disk, not a copy of the files, so it's small (a few MB to tens of MB).
- `sudo` created it, so root owns it. Check with `ls -lh ~/old-disk.json`. I can still read it, which is why `ncdu -f` works without `sudo`.
- Make it mine: `sudo chown atrocity: ~/old-disk.json` (`chown` = change owner, `atrocity:` = my user and my group).

#### smartctl: disk health

SMART is a self-monitoring system built into every drive. It counts errors and wear over the drive's life.

```bash
sudo apt install --no-install-recommends smartmontools
sudo smartctl -H /dev/sda           # overall verdict: PASSED / FAILED
sudo smartctl -A /dev/sda           # the detailed counters
sudo smartctl -a /dev/sda | less    # everything (model, age, error log)
```

`PASSED` alone isn't enough, since drives often "pass" while already wearing out. In `-A`, check the `RAW_VALUE` column:

| ID | Attribute | Healthy value | Meaning |
|---|---|---|---|
| 5 | `Reallocated_Sector_Ct` | 0 | Bad spots the drive has already swapped out |
| 197 | `Current_Pending_Sector` | 0 | Spots it's struggling to read right now |
| 198 | `Offline_Uncorrectable` | 0 | Spots it couldn't read at all |
| 9 | `Power_On_Hours` | any | Age: total hours it has been running |
| 193 | `Load_Cycle_Count` | under a few hundred thousand | How often the heads parked. Laptop drives do this a lot. |

Any non-zero value in 5, 197 or 198 means: copy anything important off this disk first, and don't trust it with backups.

Reading the `-A` table:

| Column | Meaning |
|---|---|
| `VALUE` | Current health score on the maker's scale (usually 1–200 or 1–253). Higher = better. |
| `WORST` | Lowest `VALUE` ever recorded |
| `THRESH` | If `VALUE` drops to this or below, the attribute has failed. `000` = never fails, informational only. |
| `TYPE` | `Pre-fail`: failing predicts the drive is about to die. `Old_age`: normal wear. |
| `WHEN_FAILED` | `-` = never failed |
| `RAW_VALUE` | The actual count (hours, sectors, degrees). The most useful column. |

#### My HDD's result (`/dev/sda`, WD Blue WD10SPZX)

Verdict: healthy. `PASSED`, and every error counter is 0.

| ID | Attribute | Raw | What it means for me |
|---|---|---|---|
| 5, 196 | Reallocated sectors / events | 0 | No bad spots ever swapped out |
| 197, 198 | Pending / uncorrectable sectors | 0 | Nothing it struggles to read |
| 1, 7, 200 | Read / seek / write error rates | 0 | Clean |
| 199 | `UDMA_CRC_Error_Count` | 0 | Cable and connection are fine |
| 194 | Temperature | 28 °C | Cool |
| 9 | `Power_On_Hours` | 11,263 | ~470 days of total running time |
| 240 | `Head_Flying_Hours` | 4,102 | Heads actually reading/writing. Parked the rest of the time. |
| 193 | `Load_Cycle_Count` | 246,249 | Head parks. WD rates this model for ~600,000, so ~40% used. Watch it on a 24/7 server. |
| 4 | `Start_Stop_Count` | 41,480 | Spin-downs and spin-ups, typical for a laptop drive |
| 191 | `G-Sense_Error_Rate` | 31,345 | Shocks or bumps the drive felt while running (carried around as a laptop). Not damage by itself. Sitting still as a server, this should stop growing. |
| 192 | `Power-Off_Retract_Count` | 22 | Emergency head parks from sudden power loss. Low. |
| 241 / 242 | Total LBAs written / read | 7.30 / 12.06 billion | × 512 bytes ≈ 3.7 TB written, 6.2 TB read over its life |

Healthy today doesn't mean it can't die tomorrow. Important files should always exist in at least two places (more on backups in Phase 5).

Check again every few months, and compare the numbers that matter (5, 193, 197, 198):

```bash
sudo smartctl -A /dev/sda
```

Source for the load-cycle rating: [WD10SPZX spec](https://www.zones.com/site/product/index.html?id=105133120&page_name=product_tech_spec).

### Assignment 3: Managing services

Practise on `ModemManager`. The Dell has no SIM modem, so it's safe to stop, and turning it off is a real cleanup.

| Command | What it does |
|---|---|
| `systemctl status <name>` | Is it running, since when, how much memory, plus its last log lines. No `sudo` needed to look. |
| `sudo systemctl stop <name>` | Stop it now. Starts again at next boot if enabled. |
| `sudo systemctl start <name>` | Start it now. |
| `sudo systemctl restart <name>` | Stop then start. Use after changing a config file. |
| `sudo systemctl reload <name>` | Re-read config without stopping (only some services support it, e.g. Nginx). |
| `sudo systemctl enable <name>` | Start automatically at boot. Doesn't start it now. |
| `sudo systemctl disable <name>` | Don't start at boot. Doesn't stop it now. |
| `sudo systemctl disable --now <name>` | Disable and stop in one go. (`enable --now` = enable and start.) |
| `sudo systemctl mask <name>` | Block it completely, so nothing can start it. Undo with `unmask`. |
| `systemctl is-active <name>` | Prints just `active` or `inactive`. |
| `systemctl is-enabled <name>` | Prints just `enabled` or `disabled`. |
| `systemctl list-unit-files --type=service` | Every installed service and whether it starts at boot. |
| `systemctl --failed` | Services that crashed or failed to start. |
| `journalctl -u <name> -n 20` | Last 20 log lines for one service (`-u` = unit). |
| `journalctl -u <name> -f` | Follow the log live (`Ctrl + C` to stop). |
| `journalctl -b -p err` | Only errors (`-p err`) since this boot (`-b`). |

Reading `systemctl status`:

```text
● ModemManager.service - Modem Manager
     Loaded: loaded (/usr/lib/systemd/system/ModemManager.service; enabled; preset: enabled)
     Active: active (running) since ...
   Main PID: 812 (ModemManager)
     Memory: 4.2M
```

- `Loaded`: where the service's config file lives, and `enabled` / `disabled` = starts at boot or not.
- `Active`: `active (running)`, `inactive (dead)` = stopped, `failed` = crashed.
- `Main PID`: its process ID. The same number shows in `top` and `ps`.
- `Memory`: how much RAM it uses right now.
- The lines at the bottom are its most recent log entries.

Practice run:

```bash
systemctl status ModemManager           # look: active (running), enabled
sudo systemctl stop ModemManager
systemctl status ModemManager           # now: inactive (dead), but still enabled
sudo systemctl start ModemManager
sudo systemctl restart ModemManager
journalctl -u ModemManager -n 20        # its log, including the stops/starts you just did
sudo systemctl disable --now ModemManager
systemctl is-enabled ModemManager       # disabled
systemctl is-active ModemManager        # inactive
```

Check whether the SSH server is installed (we set it up properly in Phase 2):

```bash
systemctl status ssh.socket
```

- `active (listening)`: installed and waiting for connections on port 22.
- `Unit ssh.socket could not be found`: the OpenSSH server isn't installed yet.

### Kernel messages: the ath10k AER spam

This line keeps printing on the Dell's screen:

```text
ath10k_pci 0000:03:00.0: AER:   Error of this Agent is reported first
```

| Part | Meaning |
|---|---|
| `ath10k_pci` | The kernel driver for the Qualcomm Atheros Wi-Fi card (the card behind `wlp3s0`). |
| `0000:03:00.0` | The card's PCI address: bus `03`, slot `00`, function `0`. Same numbers as `p3s0` in `wlp3s0`. |
| `AER` | Advanced Error Reporting. PCIe (the fast link between the card and the motherboard) checks every packet and reports damaged ones. |
| `Error of this Agent is reported first` | Two devices reported an error at the same moment (usually the card and the motherboard port it sits in). This one came first. |

The full report is 4–5 lines. The screen probably shows only this one because it's logged at `err` priority, and the rest at `warn`, which the console hides. Every line is in the kernel log.

The `severity=` line decides how worried to be:
- `Correctable` (older kernels say `Corrected`): the hardware spotted a damaged packet and resent it. Nothing lost. Common on this card, usually triggered by ASPM (Active State Power Management: the link dozes off and wakes up many times a second to save power). Harmless, but it floods the log and buries real errors.
- `Uncorrectable (Non-Fatal)` / `Fatal`: data was lost or the card needs a reset. Expect Wi-Fi drops.

| Command | What it does |
|---|---|
| `sudo dmesg` | The kernel's message buffer: hardware and driver messages since boot. Ubuntu needs `sudo` for it. |
| `sudo dmesg -T` | Same, with real clock times instead of seconds since boot. |
| `sudo dmesg -l err,warn` | Only `err` and `warn` priority lines (`-l` = level). |
| `sudo dmesg -n 3` | The screen only shows messages more serious than `err`. Stops the spam on screen, the log still gets everything. Resets at reboot. |
| `journalctl -k -b` | Kernel messages (`-k`) from this boot (`-b`), read from the saved journal. `-b -1` = the previous boot. |
| `grep -E "aer\|ath10k"` | `-E` = extended patterns, so `\|` means OR. |
| `grep -c "AER"` | Count matching lines instead of printing them. |
| `grep -A 3 "severity"` | Print each match plus 3 lines After it (`-B` = Before, `-C` = both). Good for multi-line reports. |
| `lspci` | List PCI devices: Wi-Fi, Ethernet, graphics, USB controllers. |
| `lspci -nn -s 03:00.0` | One device (`-s` = slot), with its `[vendor:device]` ID numbers. |
| `sudo lspci -vv -s 03:00.0` | Very verbose details. The `LnkCtl: ASPM ...` line shows whether link power saving is on. |
| `cat /sys/module/pcie_aspm/parameters/policy` | The ASPM policy. The one in `[brackets]` is active. |
| `cat /proc/cmdline` | The boot options (kernel parameters) the running kernel started with. |
| `sudo update-grub` | Rebuild GRUB's boot menu from `/etc/default/grub`. Needed after every edit to that file. |
| `journalctl --list-boots` | Every boot the journal remembers. `IDX` 0 = this boot, -1 = the one before, and so on. `-b`, `-b 0` and `-b -0` all mean this boot. |
| `journalctl -k -b -3 \| grep -c "AER"` | Count AER lines in an older boot (here 3 boots ago). Proves whether a problem existed before a change. |

The number in brackets, like `[ 430.745498]`, is seconds since boot (430 s ≈ 7 minutes). Gaps between lines show how often it repeats.

Is it caused by my netplan edit? No. Netplan works at the network-config layer (IP addresses, Wi-Fi login). AER happens two layers below, on the PCIe wire between the Wi-Fi card and the motherboard. The edit was also on Ethernet `enp2s0`, while the errors come from the Wi-Fi card at `03:00.0`. Proof from the logs: count AER lines in a boot from before the edit with `journalctl --list-boots`, then `journalctl -k -b -N | grep -c "AER"` (N = that boot's IDX number).

#### Step 1: Diagnose

```bash
sudo dmesg -T | grep -i -E "aer|ath10k" | tail -n 15
journalctl -k -b | grep -c "AER"
lspci -nn -s 03:00.0
sudo lspci -vv -s 03:00.0 | grep -i aspm
cat /sys/module/pcie_aspm/parameters/policy
```

Look for: `severity=`, how many AER lines this boot, the exact card model, and `ASPM ... Enabled` or `Disabled`. If the error shows a different address than `03:00.0`, use that one.

#### Step 2: Test the ASPM theory live (no reboot)

```bash
journalctl -k -b | grep -c "AER"                                   # note the number
echo performance | sudo tee /sys/module/pcie_aspm/parameters/policy
# wait ~5 minutes, then count again. Stopped growing = ASPM was the cause.
journalctl -k -b | grep -c "AER"
echo default | sudo tee /sys/module/pcie_aspm/parameters/policy    # undo
```

- `performance` = ASPM off on every PCIe link. Slightly more power, no doze/wake glitches.
- `sudo echo performance > file` fails: the `>` is handled by my own shell, which isn't root. `sudo tee file` writes the file as root.
- `Operation not permitted` = the BIOS keeps ASPM control for itself, so Linux can't change it. Go to option B.

#### Step 3: Make it permanent with a boot option

- Option A (Step 2 worked): `pcie_aspm.policy=performance`. Fixes the cause.
- Option B (it didn't): `pci=noaer`. Turns off AER reporting. The hardware still corrects the errors, but real PCIe errors are hidden too.

```bash
sudo cp /etc/default/grub /etc/default/grub.bak
sudo nano /etc/default/grub
#   GRUB_CMDLINE_LINUX_DEFAULT=""  ->  GRUB_CMDLINE_LINUX_DEFAULT="pcie_aspm.policy=performance"
#   keep anything already inside the quotes, separate options with a space
sudo update-grub
sudo reboot
cat /proc/cmdline                      # the new option should be listed
journalctl -k -b | grep -c "AER"       # 0, or no longer growing
```

Undo: `sudo cp /etc/default/grub.bak /etc/default/grub`, `sudo update-grub`, `sudo reboot`.

---

## Phase 2: Networking and SSH

Not started. Things already found that we'll handle here:

- [ ] Install / check the SSH server and log in from the Acer: `ssh atrocity@192.168.1.8`
- [ ] Give the Dell a fixed IP: reserve `192.168.1.8` for its Wi-Fi MAC address in the router (DHCP reservation)
- [ ] Stop the laptop sleeping when the lid closes. By default `systemd-logind` suspends on lid close, which takes the server offline. Until then, keep the lid open.
- [x] Fix the boot delay from the unplugged Ethernet port (netplan `optional: true`). Done early: boot ~2 min → ~4 s.
- [ ] The Wi-Fi password is saved in plain text in `/etc/netplan/*.yaml`. Never copy that file into this public repo, a chat or a screenshot. Show it with `sudo grep -v password` instead.
- [ ] Ethernet `enp2s0` has no `dhcp4: true` in netplan. Add it before plugging in a cable.

### Fix: boot delay from the unplugged Ethernet port

Cause (confirmed): `systemd-networkd-wait-online.service` is the slowest unit in `systemd-analyze blame`. Netplan tells it to wait for Ethernet `enp2s0`, but no cable is plugged in, so it waits until it times out.

Fix: mark `enp2s0` as `optional: true`. Boot then only waits for Wi-Fi. If a cable is plugged in later, Ethernet still works.

#### Editing files with nano

`nano` is a simple terminal text editor. The shortcuts are listed at the bottom of its screen (`^` = `Ctrl`).

| Key | What it does |
|---|---|
| Arrow keys | Move the cursor |
| `Ctrl + O`, then `Enter` | Save ("write Out") |
| `Ctrl + X` | Exit. Asks to save if there are unsaved changes. |
| `Ctrl + K` | Cut the current line |
| `Ctrl + U` | Paste |
| `Ctrl + W` | Search |
| `Alt + U` | Undo |

#### Steps

The file name differs between installs (`50-cloud-init.yaml`, `00-installer-config.yaml`, `90-...yaml`). Find mine first, then use that name in place of `<file>`.
On my Dell: `00-installer-config.yaml`. Wi-Fi may be in a separate file such as `00-installer-config-wifi.yaml`. Netplan reads every `.yaml` in the folder and merges them in alphabetical order.

```bash
ls -l /etc/netplan/                          # list the config files
sudo grep -l enp2s0 /etc/netplan/*.yaml      # which file mentions the Ethernet port
sudo cp /etc/netplan/<file>.yaml /etc/netplan/<file>.yaml.bak
sudo nano /etc/netplan/<file>.yaml
sudo netplan try
```

- `ls -l` shows the files as `-rw-------  root`: only root can read them, because they hold the Wi-Fi password. That's why `cat` and `grep` need `sudo`.
- `grep -l`: print only the names of files that contain the word, not the matching lines. So the Wi-Fi password never appears on screen.
- `*.yaml`: a wildcard, meaning every file ending in `.yaml` in that folder.
- Type paths with `Tab`: `sudo nano /etc/netplan/` then `Tab` fills in or lists the real file names, so no typos.
- `cp ... .bak` makes a backup first. Netplan only reads files ending in `.yaml`, so the `.bak` copy is ignored.
- In nano, add one line under `enp2s0`, lined up exactly with `dhcp4`:

```yaml
network:
  version: 2
  ethernets:
    enp2s0:
      dhcp4: true
      optional: true        # <- add this line
  wifis:
    wlp3s0:
      dhcp4: true
      access-points:
        "<wifi-name>":
          password: "<hidden>"
```

- YAML rules: spaces only (never `Tab`), and indentation decides the structure. One wrong space breaks the file.
- `sudo netplan generate`: checks the YAML and builds the config without applying it. No output = valid. An error names the line that's wrong.
- `sudo netplan try` applies the change and waits 120 seconds. Press `Enter` to keep it. If something breaks and you don't press Enter, it rolls back by itself.

The installer's version of my file (`00-installer-config.yaml`) puts `version: 2` at the bottom. The order of keys doesn't matter in YAML, only the indentation:

```yaml
# This is the network config written by 'subiquity'
network:
  ethernets:
    enp2s0:
      dhcp4: true
      optional: true        # <- add this line
  version: 2
```

My actual file after the edit (name, MAC and password hidden):

```yaml
# This is the network config written by 'subiquity'
network:
  ethernets:
    enp2s0:
      match:
        macaddress: <ethernet-mac>
      set-name: enp2s0
      optional: true
  version: 2
  wifis:
    wlp3s0:
      access-points:
        <wifi-name>:
          password: <hidden>
      dhcp4: true
```

- `match: macaddress:` applies this block to the card with that MAC address, whatever the kernel calls it.
- `set-name: enp2s0` then names that card `enp2s0`, so the name never changes.
- No `dhcp4: true` under `enp2s0`: if a cable is plugged in, the port comes up but never asks the router for an IP. Add `dhcp4: true` (lined up with `optional`) before using Ethernet.

Compare the backup with the current file, and view the file without showing the password:

```bash
sudo diff /etc/netplan/00-installer-config.yaml.bak /etc/netplan/00-installer-config.yaml
sudo grep -v password /etc/netplan/00-installer-config.yaml
```

- `diff old new`: prints only the lines that differ. `<` = only in the old file, `>` = only in the new one. No output = identical. Lines like `3c3` say which line numbers changed (`a` added, `d` deleted, `c` changed).
- `grep -v word`: print every line *except* those containing the word (`-v` = invert). Use it before pasting or screenshotting this file.

Check that it worked:

```bash
sudo reboot
systemd-analyze                         # total should be much lower
systemd-analyze blame | head -n 5       # wait-online should drop off the top
```

Undo, if needed: `sudo cp /etc/netplan/50-cloud-init.yaml.bak /etc/netplan/50-cloud-init.yaml`, then `sudo netplan apply`.

---

## Phase 3: Backend infrastructure

### Git basics: pushing this notebook to GitHub

Repo: https://github.com/Arsh-Coding/Linux--bltn-help

First-time setup (run once per repo):

```bash
git init
git add README.md linux-commands.md linux-doubts.md
git commit -m "first commit"
git branch -M main
git remote add origin https://github.com/Arsh-Coding/Linux--bltn-help.git
git push -u origin main
```

| Command | What it does |
|---|---|
| `git init` | Turns the current folder into a Git repository (creates a hidden `.git` folder that stores history). |
| `git add <files>` | Stages files: marks them to be included in the next commit. Naming files avoids committing things by accident. |
| `git commit -m "msg"` | Saves a snapshot of the staged files with a message. Local only, nothing is uploaded yet. |
| `git branch -M main` | Renames the current branch to `main` (GitHub's default name). |
| `git remote add origin <url>` | Saves the GitHub repo URL under the nickname `origin`. |
| `git push -u origin main` | Uploads commits to GitHub. `-u` remembers `origin main`, so next time plain `git push` is enough. |

Every update after that:

```bash
git status                                   # what changed?
git add linux-commands.md linux-doubts.md    # stage the changed files
git commit -m "Add notes on <topic>"         # snapshot
git push                                     # upload
```

One-time identity setup, if `git commit` says "Author identity unknown":

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

---

## Troubleshooting log

### Screen is black after installing Ubuntu Server

Ubuntu Server has no desktop. A black screen with white text and a `login:` prompt is the normal working state.

- See `dell login:`? Type your username, Enter, then password (invisible), Enter.
- Boot messages covering the prompt? Press `Enter` to get a fresh `login:` line.
- Completely blank? Press any key (the console may have blanked). Then try `Ctrl + Alt + F2` to open a fresh terminal.

Status: solved. Logged in as `atrocity@atrocity-old-dell`.

### Root filesystem only uses half the SSD

- Problem: `lsblk` shows `sdb3` is 116 GB but `ubuntu-lv` (mounted at `/`) is only 58 GB.
- Cause: the Ubuntu installer only gives the root LV part of the volume group by default.
- Fix: `sudo lvextend -r -l +100%FREE /dev/ubuntu-vg/ubuntu-lv` (see Assignment 2: Storage).
- Status: solved. `sudo vgs` shows `VFree 0`. `df -h /` went from `57G size, 47G avail` to `115G size, 102G avail`.

### `du` seems to run forever

- Cause 1: `| sort -h` can't sort until it has every line, so it prints nothing until `du` completely finishes. The screen looks frozen even while `du` is working.
- Cause 2: the old disk holds ~390 GB on a slow laptop HDD, through the NTFS driver. Adding up every file there can take many minutes.
- Cause 3: without `-x`, `du /` also walks into `/mnt/sda2..4` and counts all the old Windows files.
- Fix: `Ctrl + C` to stop. Drop `| sort -h` to see each folder as it finishes. Keep `-x` when scanning `/`. For big disks, use `ncdu`, which shows progress while it scans and lets you browse the results.

### Battery shows `discharging`

- Seen: `upower` showed 47% and `discharging`. About an hour later: 38%, draining at 5.3 W, `time to empty: 1.9 hours`.
- Likely cause: charger not plugged in. With the 75–80% charge limit it should charge whenever it's below 75%.
- Check: plug in, run `upower -b`, and `state` should change to `charging`. Confirm the adapter with `cat /sys/class/power_supply/AC/online` (1 = connected).
- If it's plugged in but `online` is 0: try another socket, reseat the cable, or check the adapter. Dell laptops can refuse non-Dell or damaged adapters.
- Why it matters: when the battery hits 0 the server just loses power, like pulling the plug. Running services and unsaved files can be damaged.
- Cause: the charger wasn't plugged in.
- Status: solved. Plugged in, `state` now shows `charging`. It will stop at 80% because of the charge limit.

### `cp: cannot stat '/etc/netplan/50-cloud-init.yaml': No such file or directory`

- Meaning: `stat` = look up a file's details. `cp` couldn't find the source file, so nothing was copied and nothing changed.
- Cause: the netplan file on my Dell has a different name. I copied the name from the notes instead of checking with `ls` first.
- Fix: `ls -l /etc/netplan/` to see the real name, and `sudo grep -l enp2s0 /etc/netplan/*.yaml` to find the file with the Ethernet config.
- Lesson: look before you act. Use `Tab` to complete paths, so you only ever type names that really exist.
- Status: solved. My Ethernet config is in `/etc/netplan/00-installer-config.yaml` (the name Ubuntu's server installer, "subiquity", uses).

### `df -ht`: "option requires an argument -- 't'"

- Cause: options are case-sensitive. Lowercase `-t` means "only show this filesystem type" and needs a value after it. Capital `-T` adds the Type column.
- Fix: `df -hT`, or `df -h -t ext4` to filter.
- Lesson: when an option misbehaves, run `command --help` and check the exact letter and case.

### Boot takes ~1 minute to reach `login:`

Status: solved. Diagnosed with `systemd-analyze`, `systemd-analyze blame` and `systemd-analyze critical-chain` (see "Boot time" in Phase 1 and doubt #2).

- Clue: `ip addr` shows Ethernet `enp2s0` with `NO-CARRIER` (no cable).
- Confirmed: `systemd-analyze blame | head -n 5` shows `systemd-networkd-wait-online.service` at the top.
- Cause: netplan has `enp2s0` configured, so boot waits for a network that will never come up until the wait times out.
- Fix: add `optional: true` under `enp2s0` in `/etc/netplan/50-cloud-init.yaml`, then `sudo netplan try`. Full steps in "Fix: boot delay from the unplugged Ethernet port" (Phase 2).
- Result: boot ~2 min → ~4 s. My netplan file turned out to be `/etc/netplan/00-installer-config.yaml` (found with `sudo grep -l enp2s0 /etc/netplan/*.yaml`), not `50-cloud-init.yaml`.
- Lesson: measure first (`systemd-analyze blame`), back up before editing (`cp file file.bak`), apply network changes with `netplan try` so a mistake rolls itself back.

### `ath10k_pci ... AER: Error of this Agent is reported first` keeps printing

- Seen: this line repeats on the Dell's screen, over and over.
- Meaning: the Wi-Fi card (driver `ath10k_pci`) is reporting errors on its PCIe link. See "Kernel messages: the ath10k AER spam" in Phase 1.
- Likely cause: `Correctable` errors from ASPM power saving. Very common with Qualcomm Atheros cards. Harmless, but floods the log.
- Quick relief: `sudo dmesg -n 3` hides it from the screen until the next reboot.
- Repeats every few seconds to ~30 s (seen at 430 s, 430.8 s, 457 s, 480 s after boot).
- Not caused by the netplan edit: netplan only sets IPs and Wi-Fi login, the edit was on Ethernet, and AER is a hardware link error on the Wi-Fi card.
- `sudo diff` of the backup and the current file: only `7a8 > optional: true`. Nothing else changed.
- `sudo dmesg -n 3` worked: the lines stopped appearing on screen.
- AER lines per boot (`journalctl -k -b -N | grep -c "AER"`):

| Boot | When | Length | AER lines |
|---|---|---|---|
| -5 | Fri 20 Nov | 23 min | 508 |
| -4 | Sat 21 → Sun 22 Nov | ~22 h | 4 |
| -3 | Sun 22 Nov | 4.5 h | 4 |
| -2 | Sun 22 Nov | 2 h | 4 |
| -1 | Sun 22 Nov | 2.5 min | 217 |
| 0 | Sun 22 Nov | 19 min | 2,889 |

- Reading it: 4 is probably the normal startup lines (`AER: enabled with IRQ ...`, one per PCIe port), meaning zero errors. Some boots are clean and some are flooded, so whatever the timing of the edit, the config can't be the trigger. The flood also existed on Friday.
- ASPM policy is `[default]`: Linux keeps whatever ASPM settings the BIOS chose. So the ASPM test (Step 2) is worth running, while the errors are still flowing.
- Status: diagnosing. Next: check the severity and ASPM state (Step 1), test ASPM live (Step 2), then add a GRUB boot option (Step 3).

---

## Ideas for later

Inspiration from home lab setups. Revisit after Phase 4, once I know what the hardware can handle.

- Docker dashboard: Arcane or Portainer (web UI for containers)
- Monitoring: Prometheus + Grafana (Phase 6)
- Safe remote access: Tailscale (private VPN, no port forwarding needed)
- Network ad blocker: Pi-hole
- Personal cloud / media: Nextcloud, Jellyfin, Navidrome
- Virtualization: Proxmox VE (needs plenty of RAM, replaces Ubuntu as the base OS)
- RAM upgrade: the i5-8250U supports up to 32 GB DDR4. Check how many slots the Dell has and what's in them with `sudo dmidecode -t memory`. Going to 16 GB would make VMs and heavier stacks realistic.
- The 1 TB `sda` disk: once its old contents are checked, use it for backups, Docker volumes or media.
