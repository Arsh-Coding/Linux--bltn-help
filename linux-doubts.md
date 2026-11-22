# Linux Doubt Sessions

Questions I ask while learning, with the answers. Any new command from here also goes into [linux-commands.md](linux-commands.md).

| # | Date | Topic | Phase |
|---|---|---|---|
| 1 | 2026-11-22 | `upower -b` vs `upower -i $(upower -e \| grep BAT)` | Saved commands (battery) |
| 2 | 2026-11-22 | Why does Ubuntu take ~1 minute to reach the login prompt? | Phase 1 |
| 3 | 2026-11-22 | If the old partitions are mounted read-only, can I still open the files? | Phase 1 (storage) |
| 4 | 2026-11-22 | What is `du`? | Phase 1 |
| 5 | 2026-11-22 | What does `sudo ncdu -r /mnt/sda2` do? | Phase 1 (storage) |
| 6 | 2026-11-22 | How do I zoom in / out? | Phase 1 (console) |
| 7 | 2026-11-22 | I can't scroll up on the Dell, long output is lost | Phase 1 (console) |
| 6 | 2026-11-22 | ncdu shows my files but won't open them. If I close it, does it rescan? | Phase 1 (storage) |
| 7 | 2026-11-22 | Where does `sudo ncdu -o ~/old-disk.json /mnt` save the file? | Phase 1 (storage, permissions) |
| 8 | 2026-11-22 | Are you about to format or delete the old disk? | Phase 1 (storage) |
| 9 | 2026-11-22 | How do I see the folder names on each old partition? | Phase 1 (navigation) |
| 10 | 2026-11-22 | I opened my old files with `vi file.txt`. How do I list everything on the old disk? | Phase 1 (navigation, vi) |
| 11 | 2026-11-22 | What's the harm if my Wi-Fi MAC address leaks? | Phase 2 (networking, privacy) |

---

## 1. Why does `upower -b` give the same result as the long battery command?

Question: I use `upower -i $(upower -e | grep BAT)` for battery status, but `upower -b` shows the same output. What's the difference?

Answer: They end up at the same information, they just get there differently.

- `upower -b` (`--battery`) is a built-in option that prints full details for every battery device. One step, no searching.
- The long version does it manually in three steps: list all power devices (`-e`), keep only the battery line (`grep BAT`), then show details for that one device (`-i`).

| | `upower -i $(upower -e \| grep BAT)` | `upower -b` |
|---|---|---|
| How it works | You find the battery path, then ask about it | upower finds batteries itself |
| Battery not named `BAT...` (e.g. `CMB0`) | Breaks, `grep` finds nothing | Still works |
| Laptop with two batteries | `-i` expects one device path | Shows all of them |
| Availability | Works on any upower version | Newer versions only (older ones say "Unknown option") |

Verdict: use `upower -b` day to day. Keep the long version in your head anyway, because the pattern inside it is used everywhere in Linux:

- `|` (pipe) sends one command's output into another
- `grep` filters lines
- `$( ... )` runs a command and pastes its result into another command

Related options:

```bash
upower -e        # list every power device path (battery, AC adapter, etc.)
upower -d        # dump details for ALL devices, not just the battery
upower -v        # show upower version
upower --help    # list every option (this is how you discover things like -b)
```

Lesson: I found `-b` myself by reading `man upower`. When you find a long command online, check `man command` or `command --help`. There's often a built-in shortcut.

Source: [upower(1) man page](https://man.archlinux.org/man/upower.1)

---

## 2. Why does Ubuntu take ~1 minute to reach the login prompt, slower than Windows?

Question: The Dell takes a full minute to show the `login:` prompt. Why is it slower than Windows startup?

Answer: Ubuntu Server isn't slow by nature. A healthy install on this hardware should reach `login:` in roughly 10 to 30 seconds. A full minute usually means one thing is waiting for something, often on a timer.

Why Windows feels faster:

- Fast Startup: Windows "Shut down" doesn't fully shut down by default. It hibernates the core of the system to disk and resumes it next time. Ubuntu does a real cold boot every time.
- Different hardware: the Acer is likely newer and on an SSD. The Dell is older, and may have a spinning hard disk (HDD).

Usual suspects on Ubuntu Server:

| Suspect | Sign | Typical delay |
|---|---|---|
| Waiting for network (`systemd-networkd-wait-online`) | Boot screen shows "A start job is running for Wait for Network to be Configured" | Up to 2 minutes |
| GRUB menu countdown | A menu with "Ubuntu / Advanced options" sits on screen before booting | Up to 30 seconds |
| Old hard disk (HDD) | `lsblk -d -o NAME,ROTA` shows `ROTA 1` | Everything is slower |
| BIOS / POST | Dell logo stays a long time before anything else appears | Varies |

Don't guess, measure. systemd records how long every step of boot took:

```bash
systemd-analyze                        # total boot time, split into stages
systemd-analyze blame | head -n 15     # 15 slowest services
systemd-analyze critical-chain         # the chain that actually delayed the login prompt
lsblk -d -o NAME,ROTA,SIZE,MODEL       # disk type: ROTA 1 = HDD, 0 = SSD
```

Reading the output:

- `systemd-analyze` splits the time into `firmware` (BIOS), `loader` (GRUB), `kernel` and `userspace` (services). Whichever is biggest is where to dig.
- `blame` lists services from slowest to fastest. Services start in parallel, so a slow one doesn't always delay boot.
- `critical-chain` shows the path that really held up boot. `@` = when that unit was ready, `+` = how long it took itself. Look for the biggest `+`.

Perspective: servers rarely reboot, so a slow boot isn't urgent. It's still a good exercise in finding a problem with data instead of guessing.

Status: solved. Boot went from ~2 minutes to ~4 seconds after one line in netplan.

Update: `ip addr` shows the Ethernet port `enp2s0` with `NO-CARRIER` (no cable) while Wi-Fi does the work. That makes "waiting for network" the strongest suspect. `systemd-analyze blame | head -n 5` will confirm it if `systemd-networkd-wait-online.service` is at the top.

Confirmed: `systemd-networkd-wait-online.service` is at the top of `blame`. Netplan tells it to wait for Ethernet, there's no cable, so it waits until it times out. Fix: mark `enp2s0` as `optional: true` in netplan (steps in [linux-commands.md](linux-commands.md), "Fix: boot delay from the unplugged Ethernet port").

Result: added `optional: true` under `enp2s0` in `/etc/netplan/00-installer-config.yaml`, applied it, rebooted. ~2 min → ~4 s.

Lesson: the slow boot wasn't old hardware or Linux being slow. It was one service waiting for a cable that isn't there. Measuring found it in two commands, and fixing it took one line.

---

## 3. If the old partitions are mounted read-only, can I still open the files?

Question: I mounted the old Windows partitions with `-o ro`. Does read-only mean I can't open them?

Answer: You can open them. Read-only blocks changes, not access. Think of a museum: you can look at everything, but you can't touch.

| Action | Allowed? |
|---|---|
| Browse folders, list files | Yes |
| Open and read files | Yes |
| Copy files from the disk to somewhere else | Yes, because copying only reads the original |
| Edit, rename, move, delete, create | No: `Read-only file system` error, even with `sudo` |

Try it:

```bash
cd /mnt/sda2
ls -lh
touch test.txt        # fails: Read-only file system
cd ~
```

Why use it here: those ~390 GB are probably the only copy of my old files. Read-only means a typo (`rm` in the wrong folder, say) can't destroy anything while I explore.

The catch on a server: there's no screen app for photos, videos or PDFs. On the Dell I can browse, check file types with `file`, read text files with `less`, and copy files. To actually view media, I copy it to the Acer, over SSH in Phase 2 or on a USB stick.

Commands are listed in [linux-commands.md](linux-commands.md), under Assignment 2, Step 7, "What read-only allows".

---

## 4. What is `du`?

Question: What does `du` do?

Answer: `du` = disk usage. It shows how much disk space files and folders take up.

It pairs with `df`:

- `df` (disk free): how full is the whole disk? Like checking how full your phone storage is.
- `du` (disk usage): which folders are filling it? Like opening the storage screen to see which apps are big.

Plain `du` with no options lists every subfolder, one per line, and floods the screen. Use it with options:

```bash
du -sh ~                                  # total size of my home folder
du -sh *                                  # size of each item in the current folder
du -h --max-depth=1 ~ | sort -h           # each folder in home, smallest to largest
sudo du -xh --max-depth=1 / | sort -h     # each top-level folder of the whole system
```

| Option | Meaning |
|---|---|
| `-h` | Human-readable (`K`, `M`, `G` instead of raw numbers) |
| `-s` | Summarize: one total per argument instead of every subfolder |
| `--max-depth=1` (or `-d 1`) | Go only one level deep |
| `-x` | Stay on one filesystem (skip `/proc`, `/sys`, other mounted disks) |
| `-a` | Include files too, not just folders |

Why `sudo` for `/`: some folders (like `/root`) can only be read by root. Without `sudo` you get `Permission denied` lines and the totals come out too small. To hide those errors instead, add `2>/dev/null`: `2` is the error output, and `/dev/null` is a black hole that throws away anything sent to it.

`du` vs `ls -lh`: `ls` shows how big the content of a file is. `du` shows how much disk space it actually takes. Disks store data in blocks (usually 4 KB), so a 3-byte file still uses 4.0K on disk.

```bash
echo hi > test.txt     # create a tiny file (3 bytes: "hi" + newline)
ls -lh test.txt        # shows 3
du -h test.txt         # shows 4.0K
rm test.txt            # clean up
```

---

## 5. What does `sudo ncdu -r /mnt/sda2` do?

Question: What is this command for?

Answer: It opens an interactive disk usage browser on the old Windows partition, in a mode where nothing can be deleted.

| Part | Meaning |
|---|---|
| `sudo` | Run as root, so every folder can be read and the totals are complete |
| `ncdu` | NCurses Disk Usage: `du` with a full-screen, browsable interface |
| `-r` | Read-only: turns off ncdu's built-in delete key (`d`) |
| `/mnt/sda2` | Where the old Windows partition is mounted |

Why `ncdu` instead of `du` here:

- `du` prints a list and exits. To look inside a big folder you run it again, and it rescans everything. On ~390 GB of slow laptop HDD, each scan takes minutes.
- `ncdu` scans once and shows progress while it works. Then you browse the results like a file manager: biggest items at the top, `Enter` to go into a folder, `←` to come back. No rescanning.

Why `-r`: ncdu can delete files (press `d`), which makes it a cleanup tool too. While exploring a disk that holds the only copy of old files, `-r` removes that risk. `/mnt/sda2` is already mounted read-only, so this is a second safety layer. That's a good habit with disks you care about.

- `-r` = no deleting
- `-rr` = no deleting, and no opening a shell from inside ncdu (`b` key) either

Keys:

| Key | Action |
|---|---|
| `↑` `↓` | Move |
| `Enter` / `→` | Open folder |
| `←` | Go back up |
| `i` | Info about the selected item |
| `s` / `n` | Sort by size / by name |
| `?` | Help |
| `q` | Quit |

Source: [ncdu manual](https://dev.yorhel.nl/ncdu/man)

---

## 6. ncdu shows my files but won't open them. If I close it, does it rescan?

Question: `sudo ncdu -r /mnt/sda2` finished and I can see all the files, but I can't open any. If I quit, will it scan everything again?

Answer: ncdu is a map of sizes, not a file opener. It only measures, so `Enter` goes into folders but does nothing on files.

Opening files without losing the scan:

1. In ncdu, go to the folder you want and press `b`. A shell opens inside that folder.
2. Look around: `ls -lh`, `file <name>` (what type it is), `less <name>` (read text, `q` to quit).
3. Type `exit` to go back to ncdu. The scan is still there.

Careful: ncdu ran with `sudo`, so that shell is a root shell (prompt ends in `#`). `/mnt/sda2` is read-only and safe, but the rest of the system isn't. Look, don't change, then `exit`.

If I quit with `q`:

- The results are gone, and running ncdu again rescans.
- The second scan is usually much faster. Linux keeps recently read folder information cached in RAM until a reboot or until the memory is needed.

Never rescan: save the scan to a file.

```bash
sudo ncdu -o ~/old-disk.json /mnt     # scan all three partitions once and save
ncdu -f ~/old-disk.json               # browse the saved results instantly, any time
```

- `-o` = output the scan to a file instead of browsing
- `-f` = read a saved scan from a file
- Works after a reboot or unmount too. It's a snapshot, which is fine for a read-only disk.

Reminder: photos, videos, PDFs and Word files can't be viewed on the server at all. Find them with ncdu, then copy them to the Acer (SSH in Phase 2, or a USB stick) to open them.

Source: [ncdu manual](https://dev.yorhel.nl/ncdu/man)

---

## 6. How do I zoom in / out?

Question: How do I make the text bigger or smaller?

Answer: Depends on which screen.

On the Dell's own screen (the text console): `Ctrl` + `+` doesn't work there. There's no app and no zoom, just the Linux kernel drawing text with a font. To make text bigger, switch to a bigger font.

Try it for now (resets on reboot):

```bash
ls /usr/share/consolefonts | grep 32x16     # list the big fonts available
setfont Lat15-TerminusBold32x16             # big, bold, easy to read
setfont Lat15-Terminus24x12                 # medium
setfont                                     # back to the default
```

- `/usr/share/consolefonts`: the folder where console fonts live.
- `32x16`: each character is 32 pixels tall and 16 wide. Bigger numbers = bigger text.
- If `setfont` complains about the console, put `sudo` in front.

Make it permanent:

```bash
sudo dpkg-reconfigure console-setup
```

A menu opens. Arrow keys to move, `Enter` to pick: `UTF-8` → `Guess optimal character set` → `TerminusBold` → `16x32` (biggest). It writes the choice to `/etc/default/console-setup`, so it survives reboots.

`dpkg-reconfigure` reruns the setup questions for a package that's already installed. Here, the package is `console-setup`, which controls the console's font and keyboard.

Over SSH from the Acer (Phase 2) or in Kiro: the text is drawn by the Windows app, so normal zoom works.

| Where | Zoom in | Zoom out | Reset |
|---|---|---|---|
| Windows Terminal / PowerShell | `Ctrl` + `=` or `Ctrl` + scroll up | `Ctrl` + `-` or `Ctrl` + scroll down | `Ctrl` + `0` |
| Kiro (whole UI) | `Ctrl` + `=` | `Ctrl` + `-` | `Ctrl` + `0` (numpad) |

---

## 7. Where does `sudo ncdu -o ~/old-disk.json /mnt` save the file?

Question: Where does this file end up?

Answer: In my home folder, on the SSD: `/home/atrocity/old-disk.json`.

| Part | Meaning |
|---|---|
| `~` | Shortcut for my home folder, `/home/atrocity`. Check with `echo ~`. |
| `~/old-disk.json` | A file called `old-disk.json` in my home folder |
| `/mnt` | What gets scanned (the three old partitions). Nothing is written there, and it's read-only anyway. |

What's inside: only the list of folders and files with their sizes. It's a map of the old disk, not a copy, so it's small (a few MB to tens of MB). Deleting it loses nothing.

Two details about `sudo`:

- `~` still means my home, not root's (`/root`). My shell replaces `~` with `/home/atrocity` before `sudo` even starts.
- The file is owned by root, because root created it. I can still read it, so `ncdu -f ~/old-disk.json` works without `sudo`.

```bash
echo ~                                   # /home/atrocity
ls -lh ~/old-disk.json                   # owner shows as root, plus the size
sudo chown atrocity: ~/old-disk.json     # make it mine (chown = change owner)
ls -lh ~/old-disk.json                   # owner now atrocity
```

`atrocity:` with a colon and nothing after it means "user atrocity, and that user's own group".

---

## 8. Are you about to format or delete the old disk?

Question: You asked whether I have somewhere to copy my files. Are you planning to format or delete the old disk?

Answer: No. Nothing is being formatted or deleted.

- Kiro can't run anything on the Dell. It only sees what I paste and edits these notes on the Acer. Every command on the Dell is one I type myself.
- So far everything has been read-only: `lsblk`, `df`, `smartctl`, and mounting with `-o ro`. Nothing on `sda` has changed.
- The "somewhere to copy" question was only about one of the options (B: copy off, then wipe). Keeping the disk exactly as it is (option A) is a perfectly good choice.

Decision for now: leave the old disk untouched. The SSD has 102 GB free, which is plenty until Phase 5. Then we decide whether the HDD becomes backup storage.

If I ever do choose to wipe it, it'll be a separate, deliberate step: first copy what I want to keep, then double-check the device name (`sda`, not `sdb`), and only then format.

---

## 9. How do I see the folder names on each old partition?

Question: How do I list what's on each old partition?

Answer: Use `ls` on each mount folder. First check the partitions are still mounted, because mounts don't survive a reboot.

```bash
findmnt /mnt/sda2                      # prints a line if mounted, nothing if not
ls /mnt/sda2 /mnt/sda3 /mnt/sda4       # top-level folders of all three, one block each
ls -lh /mnt/sda2                       # with sizes and dates
```

If they're not mounted (after a reboot):

```bash
sudo mount -t ntfs3 -o ro /dev/sda2 /mnt/sda2
sudo mount -t ntfs3 -o ro /dev/sda3 /mnt/sda3
sudo mount -t ntfs3 -o ro /dev/sda4 /mnt/sda4
```

Going deeper:

```bash
find /mnt/sda2 -maxdepth 2 -type d | less    # folders and their subfolders, 2 levels deep
ncdu -f ~/old-disk.json                      # the saved scan: folder names plus sizes
```

- `-maxdepth 2` = only go two levels down. `-type d` = directories only, no files.
- `| less` = scroll page by page. `q` to quit.
- `ncdu -f` is the nicest view if the scan was saved: biggest folders first, with sizes.

---

## 10. I opened my old files with `vi file.txt`. How do I list everything on the old disk?

Question: I recognise a lot of my old files and even opened some with `vi file.txt`. How do I get a full list of what's there?

Answer, part 1: vi on a read-only disk is safe. You can read, scroll and search, but `:w` (save) fails because the mount is read-only.

The one thing to remember in vi:

```text
Esc   then   :q!   then   Enter      -> quit without saving, from anywhere
```

vi starts in Normal mode, where keys are commands, not typing. `i` switches to Insert mode (typing). `Esc` goes back. `:` opens the command line at the bottom (`:q` quit, `:q!` force quit, `:wq` save and quit).

For just reading, `view file.txt` (vi in read-only mode) or `less file.txt` are safer habits. vi only shows text files properly. Word, PDF and photo files look like garbage in it.

Answer, part 2: make the list once, then search it instantly.

```bash
find /mnt/sda2 /mnt/sda3 /mnt/sda4 -type f > ~/old-disk-files.txt
wc -l ~/old-disk-files.txt
less ~/old-disk-files.txt
grep -i "photo" ~/old-disk-files.txt
```

| Part | Meaning |
|---|---|
| `find ... -type f` | List every file under those folders, with full paths |
| `> ~/old-disk-files.txt` | Save the output into a file in my home folder, instead of printing it |
| `wc -l` | Count lines, which here = number of files |
| `less` | Scroll the list |
| `grep -i "photo"` | Show only lines containing "photo", ignoring case |

Why save it: walking ~390 GB on a slow laptop HDD takes minutes, but searching a text file on the SSD takes a fraction of a second.

Folder overview instead of every file:

```bash
find /mnt/sda2 -maxdepth 2 -type d
sudo apt install tree
tree -d -L 2 /mnt/sda2
```

These lists contain personal file names. Keep them on the Dell, out of the public GitHub repo.

---

## 7. I can't scroll up on the Dell, long output is lost

Question: When a command prints a lot, the top part scrolls off the screen and I can't scroll back to it.

Answer: That's expected on the Linux text console. Old guides say `Shift` + `PageUp` scrolls back, but the Linux kernel removed that feature in version 5.9 (2020) because the code was buggy and nobody maintained it. Ubuntu 24.04 runs a much newer kernel, so there's no scrollback on the Dell's own screen. Check yours with `uname -r` (prints the kernel version).

The fix is to send long output somewhere you can move around in.

1. Pipe it into `less` (the everyday fix):

```bash
systemd-analyze blame | less
lsblk -f | less
sudo apt update 2>&1 | less     # 2>&1 = include error messages too
```

`less` is the same viewer `man` uses, so the keys are the same: `↑` `↓` or `Space` / `b` to scroll, `/word` to search, `n` next match, `g` / `G` top / bottom, `q` quit.

Why `2>&1`: commands have two outputs. `1` = normal output, `2` = errors and warnings. A pipe `|` only carries `1`. `2>&1` merges errors into normal output so `less` gets both.

2. Only need part of it?

```bash
command | head -n 20       # first 20 lines
command | tail -n 20       # last 20 lines
command | grep word        # only lines containing "word"
```

3. Save it to a file and read it as often as you like:

```bash
command > output.txt          # save (overwrites the file)
command | tee output.txt      # show on screen AND save
less output.txt
rm output.txt                 # clean up when done
```

`>` sends output into a file instead of the screen. `tee` splits it like a T-shaped pipe: one copy to the screen, one to the file.

4. Get scrollback back with `tmux`:

`tmux` runs your shell inside a window that keeps its own history. Check if it's installed with `which tmux` (if not: `sudo apt install tmux`).

```bash
tmux                # start it (green bar appears at the bottom)
```

Scroll: press `Ctrl` + `b`, let go, then press `[`. Now `PageUp` / `PageDown` / arrows scroll. `q` returns to typing. `exit` leaves tmux.

5. The real long-term fix: SSH (Phase 2). Once I log in to the Dell from the Acer, Windows Terminal handles scrolling, and I can copy-paste output instead of retyping it.

Source: [Phoronix: Linux 5.9 drops soft scrollback](https://www.phoronix.com/news/Linux-5.9-Drops-Soft-Scrollback)

---

## 11. What's the harm if my Wi-Fi MAC address leaks?

Question: The Dell's Wi-Fi MAC address was in my notes (inside the `altname` `wlx...`). What could someone do with it?

Answer: Very little. A MAC address is a hardware ID, not a password.

Where a MAC travels: only on the local network. Every router on the way to the internet swaps it for its own, so websites never see it. Nobody can reach or hack the Dell over the internet just by knowing it.

The small, real risks:

| Risk | How likely / how bad |
|---|---|
| Linking the device to me | Wi-Fi sends MAC addresses unencrypted in every frame, even with WPA2, so anyone nearby with a sniffer already sees it. Publishing it next to my GitHub name lets someone match "that laptop" to "this person". Mild privacy issue. |
| Impersonating the Dell | Someone on my Wi-Fi could copy the MAC to grab its reserved IP or slip past a router MAC filter. They'd need my Wi-Fi password first, and then they could read the MAC out of the air anyway. |

Lesson: MAC filtering in a router isn't real security, because MACs are visible and easy to fake. The Wi-Fi password is what protects the network.

What's more sensitive: the router's MAC, called the BSSID. Wi-Fi location databases (Google, Apple, WiGLE) map router MACs to physical places, so a BSSID can reveal roughly where I live. Commands like `iw dev wlp3s0 link` print it on the `Connected to xx:xx:...` line. Hide that one before sharing output.

Why it was removed anyway: the notes already hid the MAC elsewhere, so the `wlx...` name was an accidental leak. The habit matters more than this one value: before posting output, check for passwords, network names, MAC addresses and public IPs.
