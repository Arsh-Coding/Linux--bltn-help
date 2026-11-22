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
| Dell username | atrocity *(update if different)* |
| Dell hostname | _fill in_ |
| Dell LAN IP | _fill in after `ip addr`_ |

## Roadmap progress

- [ ] Phase 1: Linux administration
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

#### 6. Running services

```bash
systemctl --type=service --state=running
```

`systemctl` controls systemd, the program that starts and manages background services. Press `q` to exit.
Look for: `ssh.service` (remote login), `cron.service` (scheduled jobs), `systemd-networkd` / `NetworkManager` (networking).

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

### My Dell's specs (fill in from Assignment 1)

| Item | Value |
|---|---|
| Ubuntu version | |
| CPU model | |
| Cores / threads | |
| Virtualization (VT-x / AMD-V) | |
| RAM total | |
| Disk type and size | |
| Network interface (Ethernet / Wi-Fi) | |
| LAN IP | |
| Battery health (`capacity`) | |

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

---

## Ideas for later

Inspiration from home lab setups. Revisit after Phase 4, once I know what the hardware can handle.

- Docker dashboard: Arcane or Portainer (web UI for containers)
- Monitoring: Prometheus + Grafana (Phase 6)
- Safe remote access: Tailscale (private VPN, no port forwarding needed)
- Network ad blocker: Pi-hole
- Personal cloud / media: Nextcloud, Jellyfin, Navidrome
- Virtualization: Proxmox VE (needs plenty of RAM, replaces Ubuntu as the base OS)
