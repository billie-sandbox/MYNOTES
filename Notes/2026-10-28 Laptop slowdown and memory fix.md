---
created: 2026-10-28
topics:
  - "[[Linux]]"
  - "[[Linux Mint]]"
  - "[[thickPad]]"
---

> [!summary]
> On 2026-10-28 around 16:30 the laptop (thickPad, Linux Mint 22.1, 16 GB RAM) slowed down and hung. Cause: **RAM and swap both full**, mostly from Firefox tabs, with no protection against running out of memory. Fixed by adding **zram**, raising **swappiness**, growing the **swap file to 6 GB**, enabling **Firefox tab unloading**, installing **Auto Tab Discard**, and updating **Firefox 146 → 158.0.1**. Decided **not** to use earlyoom.

## What happened

Snapshot at 16:37, about 5 minutes into the hang:

| Metric | Value |
|---|---|
| Load average (1/5/15 min) | 21.8 / 16.7 / 9.0 |
| RAM | 14 of 15 GB used, ~700 MB available |
| Swap | 2.0 of 2.0 GB used (completely full) |
| `kswapd0` (kernel swap process) | ~75% CPU, shuffling memory |
| Memory pressure (`/proc/pressure/memory`, full avg300) | 6.6%, so all work was stalled ~6.6% of the time |
| Disk `/` | 97% full, 18 GB free |

### Main cause: Firefox tabs
- Four Firefox tab processes ("Isolated Web Co") were using **1.4–1.8 GB each**, ~6.3 GB together. Firefox overall used **8+ GB**.
- Two of those tabs were also using **150–170% CPU**.
- Chromium, Termius and t3code were open at the same time, which added to it.
- **Obsidian was not the cause.** It showed CPU spikes only because it was being swapped out.

### Why it froze instead of just slowing down
Out of the box, Linux Mint 22.1 has:
- only a **2 GB swap file**, which is small for 16 GB of RAM;
- **no zram** (compressed swap in RAM);
- **no early out-of-memory killer** (earlyoom and systemd-oomd are inactive).

So when RAM ran out, the system thrashed swap on disk instead of handling it gracefully.

### Side findings (not the cause)
- **Termius** floods the kernel log with AppArmor `ALLOWED` audit messages (mDNS `recvmsg` on port 5353), hundreds every few seconds. It's harmless noise.
- **Disk is nearly full** (97–98%).
- **System was not updated for ~10 months.** Firefox 146 was from December 2025.

## Finding the heavy tab

You can't tell from outside Firefox which site a tab process belongs to; its command line only says `tab`. Use these inside Firefox:
- `about:processes` lists memory and CPU per tab. Sort by memory and close or reload the top ones.
- `about:unloads` lets you unload background tabs by hand (frees memory, keeps the tab).

## The fix (what was applied)

> [!tip] Running commands from Claude Code
> Prefix a command with `!` in the Claude Code prompt to run it in the session. sudo asks for the password.

### 1. zram: compressed swap in RAM
Memory that would go to the slow disk gets compressed in RAM instead (zstd, usually 2–5×).

```bash
sudo apt install zram-tools
echo -e "ALGO=zstd\nPERCENT=50\nPRIORITY=100" | sudo tee /etc/default/zramswap
sudo systemctl restart zramswap
```

- `PERCENT=50`: up to half of RAM (7.7 GB) of uncompressed data.
- `PRIORITY=100`: used before the disk swap file (priority -2).
- Config file: `/etc/default/zramswap`. The service `zramswap` is enabled at boot.
- Undo: `sudo apt remove zram-tools`

### 2. Swappiness 150
The default of 60 is tuned for disk swap. With cheap zram, a high value makes the kernel compress idle memory early, before RAM runs out.

```bash
echo "vm.swappiness=150" | sudo tee /etc/sysctl.d/99-zram.conf
sudo sysctl --system
```

- Config file: `/etc/sysctl.d/99-zram.conf`
- Undo: `sudo rm /etc/sysctl.d/99-zram.conf`, then reboot.

### 3. Swap file grown from 2 GB to 6 GB
The filesystem is ext4, so `fallocate` works. `/etc/fstab` already points at `/swapfile`, so no edit was needed. Run `swapoff` only when there is free RAM or zram to take the swapped data back.

```bash
sudo swapoff /swapfile
sudo fallocate -l 6G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

Cost: ~4 GB of disk space (free space went from 18 GB to 14 GB).

### 4. Firefox tab unloading on low memory
In `about:config`:

| Pref | Value | Meaning |
|---|---|---|
| `browser.tabs.unloadOnLowMemory` | `true` (already default) | unload least-recently-used background tabs when memory is low |
| `browser.low_commit_space_threshold_percent` | **5 → 10** | on Linux, "low" = available RAM below this % of total; 10% ≈ 1.5 GB, so Firefox acts before the system thrashes (the freeze happened at ~700 MB) |
| `browser.low_commit_space_threshold_mb` | `200` (left alone) | as far as I know, used on Windows/macOS only |

Unloading is not killing: the tab stays in the tab bar and reloads when clicked. The active tab is never unloaded.

### 5. Auto Tab Discard extension
- https://addons.mozilla.org/firefox/addon/auto-tab-discard/
- Settings: discard inactive tabs after **30 minutes**; don't discard tabs playing audio or pinned tabs; whitelist sites that must stay live (music, chat).

### 6. Firefox updated 146.0 → 158.0.1
Firefox comes from Linux Mint's own repository (`packages.linuxmint.com`, priority 700). Ubuntu's `firefox 1snap1` package (priority 500) is only a placeholder that switches Firefox to the Snap version. Mint's priority means the normal Mint package is chosen.

```bash
sudo apt update
sudo apt install --only-upgrade firefox
firefox --version
```

Or: Update Manager → Refresh → tick firefox → Install Updates.

## Decided against: earlyoom

earlyoom kills the process with the highest OOM score when free RAM < X% **and** free swap < Y%. It was skipped as too aggressive and risky, because:
- earlyoom 1.7 (the version in Mint 22.1) has **no "browsers only" mode**. `--prefer`/`--avoid` only shift scores by ±300.
- It **can kill the tab you are using** if that tab is the biggest memory user.
- Killed tabs show "Gah. Your tab just crashed" (they can be reloaded).

Firefox's own scores for reference: background tabs have `oom_score_adj` 167, the active tab 100, the main `firefox-bin` 0, and Chromium tab processes 300.

> [!warning] If earlyoom is ever reconsidered
> The earlyoom service splits its settings on spaces, so `Isolated Web Co` must be written as `Isolated.Web.Co`. A safer config draft:
> ```bash
> EARLYOOM_ARGS="-m 5 -s 10 -r 3600 --prefer ^(Isolated.Web.Co|Web.Content|chromium|chrome)$ --avoid ^(firefox-bin|cinnamon|Xorg|obsidian|termius-app|t3code|gnome-terminal-|systemd|claude)$"
> ```
> in `/etc/default/earlyoom`. See what it killed: `journalctl -u earlyoom | grep -i sending`

Without earlyoom, the kernel's own OOM killer remains the last resort. It acts much later, so there may be a long freeze first.

## Condition after the fix

Snapshot at 17:08:

| | During freeze (16:37) | After fix (17:08) |
|---|---|---|
| Load average | 21.8 | **1.2** |
| RAM used / available | 14 GB / 0.7 GB | **6.6 GB / 8.7 GB** |
| Swap | 2.0 of 2.0 GB (full) | **0.57 of 13.7 GB** |
| Memory stall (avg10) | ~8.6% | **0%** |
| Largest Firefox tab | 1.8 GB, ~170% CPU | **556 MB** |
| `kswapd0` | 75% CPU | idle |

- zram: 567 MB of data stored in 117 MB of real RAM (**~4.8× compression**). The disk swap file is untouched.
- Firefox 158.0.1 is running, with Auto Tab Discard active.
- Temperatures: 42–62 °C (normal).
- Disk: 14 GB free (98% full).

Final swap layout:

```
NAME       TYPE      SIZE  PRIO
/dev/zram0 partition 7.7G  100   ← used first (fast, compressed)
/swapfile  file      6G    -2    ← fallback on disk
```

## Health check commands

```bash
uptime                          # load average
free -h                         # RAM and swap
swapon --show                   # swap devices and priority
zramctl                         # zram data vs compressed size
cat /proc/sys/vm/swappiness     # should be 150
cat /proc/pressure/memory       # memory stall %; "full" near 0 is healthy
ps -eo pid,%cpu,rss,comm --sort=-rss | head -12   # biggest memory users
df -h /                         # disk space
journalctl --since "15 min ago" -p warning --no-pager | tail -25
```

When it slows down: open `about:processes` in Firefox first.

## To do later
- [ ] Free disk space (98% full, 14 GB free). Find the big folders first, e.g. `du -sh ~/* | sort -h`
- [ ] Run all pending system updates in Update Manager (security fixes)
- [ ] Optionally quiet the Termius AppArmor log spam
- [ ] Optionally consider the distro question below

## Distro notes
- **Fedora Workstation** ships zram and systemd-oomd enabled by default. It's the best of these out of the box for this problem.
- **Pop!_OS** also enables zram by default.
- **Decision: stay on Mint.** The steps above give Mint the same protection. With ~6+ GB of browser tabs, 16 GB runs out on any distro; the difference is only whether it degrades gracefully or freezes.
