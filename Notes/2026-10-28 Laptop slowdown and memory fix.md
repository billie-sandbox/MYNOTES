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

## Early warning: psi-notify (planned, not installed yet)

psi-notify only shows desktop notifications when the system is under pressure; it **never kills anything**. It's in the Mint/Ubuntu repositories (1.3.1) and runs as a user service.

It reads the kernel's pressure counters (`/proc/pressure/*`): the % of time programs are stalled waiting on CPU, memory or disk. That's a better early signal than "RAM % used". During the freeze, memory pressure hit ~12%, so an 8% threshold would have warned in time.

```bash
sudo apt install psi-notify
mkdir -p ~/.config/psi-notify
printf 'update 5\nthreshold cpu some avg10 50.00\nthreshold memory some avg10 8.00\nthreshold io full avg10 15.00\n' > ~/.config/psi-notify/config
systemctl --user enable --now psi-notify
systemctl --user status psi-notify --no-pager | head -8   # check: active (running)
```

| Config line | Alerts when… |
|---|---|
| `update 5` | (how often it checks: every 5 s) |
| `threshold memory some avg10 8.00` | programs waited on memory >8% of the last 10 s (default is 10) |
| `threshold cpu some avg10 50.00` | programs waited for CPU >50% of the time |
| `threshold io full avg10 15.00` | all work was blocked on disk >15% of the time |

- Tune the numbers in `~/.config/psi-notify/config`, then run `systemctl --user restart psi-notify`.
- Undo: `systemctl --user disable --now psi-notify && sudo apt remove psi-notify`
- When a warning pops up: open `about:processes` in Firefox, close the heavy tabs, then close unused apps.
- Optional live graph: Cinnamon **System Monitor** applet (right-click panel → Applets → Download → "System Monitor").
- Links: [psi-notify GitHub](https://github.com/cdown/psi-notify), [PSI explained](https://www.google.com/search?q=linux+PSI+pressure+stall+information+explained)

## Hardware and performance settings

Checked 2026-10-28, on AC power.

**Machine:** Lenovo ThinkPad T480 (`20L6S0VD00`)

### CPU: Intel Core i7-8550U
4 cores / 8 threads, 1.8 GHz base, up to 4.0 GHz turbo.

| Setting | Value | Meaning |
|---|---|---|
| Driver | `intel_pstate` | Intel's own frequency control (correct) |
| Governor | `powersave` | normal for intel_pstate; still boosts on demand, so the name is misleading |
| Power profile | **balanced** | Mint's default (power-profiles-daemon) |
| Energy preference (EPP) | `balance_performance` | leans towards performance |
| Turbo | enabled | ✅ |
| thermald | active | manages heat |
| Observed speed | ~2.4–2.6 GHz on all cores | boosting normally |

The CPU is not held back, just not at maximum. To go all out while plugged in (more heat and fan noise):

```bash
powerprofilesctl list
powerprofilesctl set performance   # back to normal: powerprofilesctl set balanced
```

You can also use the power/battery applet in the Cinnamon panel. Use `balanced` on battery.

### GPU: hybrid graphics
- **Intel UHD Graphics 620**: built into the CPU, driver `i915`, low power.
- **NVIDIA GeForce MX150**: separate GPU, driver `nvidia`.
- **PRIME mode: `nvidia`** (`prime-select query`). The whole desktop and all apps render on the MX150. At the time of checking it was at 13% load, in power state P0.

| Mode | Desktop runs on | Good for | Cost |
|---|---|---|---|
| `nvidia` (current) | MX150 | OBS recording (NVENC hardware encoder), Steam games | more power and heat, worse battery |
| `on-demand` | Intel | battery life, cooler; pick NVIDIA per app (right-click → "Run with NVIDIA GPU") | must choose NVIDIA per app |

Switch with `sudo prime-select on-demand` (or `nvidia`) and log out/in, or use **NVIDIA Settings → PRIME Profiles**.

**Recommendation:** mostly plugged in, recording or gaming → keep `nvidia`. Mostly on battery → `on-demand`. GPU and CPU settings were **not** the cause of the 2026-10-28 freeze; that was memory.

### T480 throttling (optional, advanced)
The T480 is known to throttle the CPU under sustained load: Lenovo's firmware caps power, and the CPU can drop to around 2 GHz. The community tool **`throttled`** (erpalma) fixes this. Research it before installing.
- [ThinkPad T480 throttling fix Linux](https://www.google.com/search?q=thinkpad+t480+throttling+linux+throttled+erpalma)
- [prime-select on-demand vs nvidia Linux Mint](https://www.google.com/search?q=prime-select+on-demand+vs+nvidia+linux+mint)

Check commands:

```bash
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
cat /sys/devices/system/cpu/cpu0/cpufreq/energy_performance_preference
cat /sys/devices/system/cpu/cpufreq/policy*/scaling_cur_freq   # current kHz per core
powerprofilesctl get
prime-select query
glxinfo -B | grep renderer     # which GPU renders the desktop
nvidia-smi                     # NVIDIA load, power state, memory
```

## Overheating finding (2026-10-29)

The new Conky strip showed **CPU 97 °C at only 17% load**, while on battery.

| Sensor | Value |
|---|---|
| Package | 97 °C |
| Cores | 79 / 84 / 97 / 97 °C |
| Fan | ~3,560 RPM |
| CPU limit (Tjmax) | 100 °C, so it was throttling |

- **Cause at that moment:** one Firefox tab (an "Isolated Web Co" process) was using 100% of one core. After it calmed down, the temperature dropped to 72 °C.
- **Why it's still a problem:** ~97 °C from a *single* busy core is too hot. A healthy T480 should be around 70–80 °C under that load. The laptop is ~8 years old, so the likely causes are a **dusty fan and heatsink** or **dried-out thermal paste**. Cleaning the fan and replacing the paste usually brings temperatures down 10–20 °C.
- [ThinkPad T480 repaste and fan cleaning](https://www.google.com/search?q=thinkpad+t480+repaste+fan+cleaning+high+temperature)

Check temperatures and fan:

```bash
cat /sys/class/hwmon/hwmon*/name    # find "coretemp"
cat /sys/class/hwmon/hwmonN/temp*_input   # millidegrees; N = the coretemp one
cat /proc/acpi/ibm/fan             # ThinkPad fan speed and level
```

## Conky resource strip (installed 2026-10-29)

A compact, always-on-top line of system stats attached to the top of the bottom panel, at the bottom-right of the screen:

```
CPU 5% 72° · RAM 55% · ZRAM 512K/7.7G · SWAP 0K/6.0G · PSI 0.0% · GPU 5%
```

- Normal values are grey and turn **yellow/red only when high**, so it doesn't distract.
- Extra items appear only when needed: the **heaviest process** when RAM ≥ 80%, and **DISK** when ≥ 95% full.
- The background is solid `#191919`, the same as the panel, so it's readable over any page.

| Item | Meaning | Yellow / red |
|---|---|---|
| CPU | total CPU % | 80 / 95 % |
| ° | CPU package temperature (`coretemp`) | 80 / 90 °C |
| RAM | RAM used % | 80 / 90 % |
| ZRAM | compressed swap in RAM, used/size; fills first, normal to use | 50 / 80 % |
| SWAP | disk swap file, used/size; only used once zram is full, **so filling = slowdown coming** | 10 / 50 % |
| PSI | memory pressure, % of the last 10 s programs waited on memory | 3 / 8 % |
| GPU | MX150 load (`off` in on-demand mode) | — |

### Files
- `~/.config/conky/conky.conf`: the compact layout (active). Thresholds are in the comments at the top. Conky reloads when the file is saved.
- `~/.config/conky/conky-full.conf`: the detailed multi-section layout (CPU per core, graphs, top 5 processes, GPU temp/VRAM, disk, Wi-Fi, batteries). Use it with `conky -c ~/.config/conky/conky-full.conf -d`.
- `~/.config/conky/stats.sh`: helper script used by both layouts. Modes: `psi`, `zram`, `swap`, `gpu`, `gpu-short`. It sets `LC_ALL=C` so decimals use `.` regardless of locale.
- `~/.config/autostart/conky.desktop`: starts Conky 10 s after login (toggle it in **Startup Applications**).

### Setup steps (done)
1. Install in a normal terminal (`sudo` can't ask for a password through Claude Code's `!`): `sudo apt install conky-all`
2. Start: `conky -c ~/.config/conky/conky.conf -d`. Stop: `pkill conky`
3. Autostart:
   ```bash
   printf '[Desktop Entry]\nType=Application\nName=Conky\nExec=sh -c "sleep 10 && conky -c %s/.config/conky/conky.conf -d"\nX-GNOME-Autostart-enabled=true\n' "$HOME" > ~/.config/autostart/conky.desktop
   ```

### Customizing (`conky.conf`)
| Change | Setting |
|---|---|
| Corner | `alignment` (`bottom_right`, `top_left`, …) |
| Offset from edge | `gap_x`, `gap_y` (`gap_y = 40` = the panel height, so it sits right on top of it) |
| Background | `own_window_colour` |
| Text / warning colors | `default_color`, `color3` (yellow), `color4` (red) |
| Font | `font = 'DejaVu Sans Mono:size=8'` |
| Refresh | `update_interval` (s); helper values refresh every 5 s |

### What didn't work
Placing it **inside** the Cinnamon panel isn't possible:
- `own_window_type = 'override'` crashes Conky 1.19 here (X error, BadMatch), with or without a transparent background.
- `'dock'` runs, but Cinnamon draws the panel on top of it.

So `'normal'` with `above` hints, sitting just above the panel, is the most compact working option. For stats truly inside the panel, a Cinnamon applet like *Bash Sensors* could run `stats.sh` instead.

Undo: `pkill conky; rm ~/.config/autostart/conky.desktop; sudo apt remove conky-all` (the config stays in `~/.config/conky/`).

## To do later
- [ ] Clean the fan/heatsink and replace thermal paste (CPU hit 97 °C on one busy core, see "Overheating finding")
- [ ] Install and configure psi-notify (see "Early warning" above)
- [ ] Decide CPU power profile: `balanced` vs `performance` when plugged in
- [ ] Decide GPU mode: keep `nvidia` or switch to `on-demand`
- [ ] Optionally research the T480 `throttled` fix
- [ ] Free disk space (98% full, 14 GB free). Find the big folders first, e.g. `du -sh ~/* | sort -h`
- [ ] Run all pending system updates in Update Manager (security fixes)
- [ ] Optionally quiet the Termius AppArmor log spam
- [ ] Optionally consider the distro question below

## Distro notes
- **Fedora Workstation** ships zram and systemd-oomd enabled by default. It's the best of these out of the box for this problem.
- **Pop!_OS** also enables zram by default.
- **Decision: stay on Mint.** The steps above give Mint the same protection. With ~6+ GB of browser tabs, 16 GB runs out on any distro; the difference is only whether it degrades gracefully or freezes.
