# Setting up Medieval: Total War on a new Raspberry Pi

This repeats the working Pi 4 setup on another Pi with one script,
[`scripts/pi/setup-new-pi.sh`](../scripts/pi/setup-new-pi.sh). The result is the same as the
working Pi: Hangover 11.16 (Wine with the WowBox64 x86 emulator), the project's d3d9lazy shim,
a private patched Mesa driver, and the full-screen launcher `play.sh` with desktop shortcuts.

Budget about 45 minutes on a Pi 4, most of it the Mesa build.

## What you need

| | |
|---|---|
| Pi | Raspberry Pi 4 (tested, 4 GB). A Pi 5 should work but has never been tried; see [Pi 5](#pi-5). |
| OS | 64-bit Raspberry Pi OS (or Debian) **13 "trixie"** with the normal desktop. Hangover's package is built for trixie; the script refuses anything else. |
| Display | A monitor on HDMI and the desktop logged in. The game runs in the desktop session (labwc/Wayland with Xwayland, the Raspberry Pi OS default). |
| Disk | 10 GB free in your home folder. |
| Network | Internet on the Pi: the script downloads Hangover from its GitHub release page and Mesa from archive.mesa3d.org, and checks both against pinned SHA-256 sums. |
| Game | The GOG **offline** installer: both `setup_medieval_total_war_gog_v1_(86311).exe` and `setup_medieval_total_war_gog_v1_(86311)-1.bin`, from `Game Files/GOG Offline installer/` on this PC. |
| Account | A normal user with sudo (asked once, for apt). Any user name works. |

## Steps

### 1. Get the project onto the Pi

**From GitHub** (recommended):

```bash
git clone https://github.com/KidKlatt/TWM_Pi ~/pi-total-war
```

**Or copy it from this PC** (PowerShell, in this folder; replace `pi@newpi` with the new Pi's
user and address):

```powershell
ssh pi@newpi "mkdir -p ~/pi-total-war"
scp -r scripts config tools patches pi@newpi:pi-total-war/
```

Only `scripts`, `config`, `tools` and `patches` are needed.

### 2. Copy the game installer

The installer is 1.6 GB and is not in the repo. From this PC (PowerShell, in this folder):

```powershell
ssh pi@newpi "mkdir -p ~/gog"
scp "Game Files/GOG Offline installer/setup_medieval_total_war_gog_v1_(86311).exe" "Game Files/GOG Offline installer/setup_medieval_total_war_gog_v1_(86311)-1.bin" pi@newpi:gog/
```

A USB stick works too: put both files in one folder on the Pi.

### 3. Run the setup script

On the Pi (over SSH or in a terminal on its desktop). Use tmux so an SSH drop does not stop it:

```bash
tmux new -s setup
```

```bash
bash ~/pi-total-war/scripts/pi/setup-new-pi.sh --installer ~/gog
```

If the SSH session drops, log in again and reattach with `tmux attach -t setup`.

What it does, in order (each step is skipped on a rerun once it has finished):

1. **Checks** the Pi: 64-bit, trixie, 4 KB kernel pages, free space, sudo.
2. **Copies** `scripts`, `config`, `tools`, `patches` into `~/mtw`, where everything lives.
3. **apt packages**: innoextract, xdotool, grim, tmux, the mingw cross-compiler for the shim,
   and the Mesa build dependencies. It simulates the install first and **stops without
   installing anything if apt would upgrade or remove an existing package** (this protects a Pi
   that also runs a server). If you have checked the list and it is fine, rerun with
   `--allow-upgrades`.
4. **Hangover 11.16**: downloads the official Debian 13 release, checks its SHA-256, installs
   `hangover-wine`, `hangover-wowbox64`, `hangover-libarm64ecfex`.
5. **Game**: unpacks the installer natively with innoextract into `~/mtw/game-lazy` (no
   emulation, no Windows installer run). It warns if the installer is a different GOG build.
6. **d3d9lazy shim**: compiles `tools/d3d9lazy/d3d9lazy.c` into `~/mtw/game-lazy/d3d9.dll`.
7. **Patched Mesa 26.2.6**: downloads, checks, applies `patches/mesa-v3d-busy-range.patch` to
   the V3D driver and builds it into `~/mtw/mesa-patched`. Only the game uses it; the system Mesa
   is not touched. About 20 minutes on a Pi 4.
8. **Wine prefix** in `~/mtw/prefixes/hangover` (a few minutes; Mono/RPC messages are normal).
9. **Shortcuts** "Medieval Total War" and "Medieval Total War (FPS counter)" on the desktop
   (if `~/Desktop` exists) and in the app menu.

The full log is in `~/mtw/logs/setup-<date>.log`.

### 4. Play

Double-click **Medieval Total War** on the desktop, or run:

```bash
~/mtw/scripts/pi/play.sh
```

- `--fps` shows a frame-rate counter, `--window` runs in a 1024x768 Wine desktop window instead
  of full screen.
- The first start is slower. Battle resolution is set in the game under **Options > Video**;
  800x600 is the tested setting. Leave the options pages with the page-corner button (Escape
  does not save them) and quit the game through its menu so the setting is kept.
- Logs: `~/mtw/logs/play-*.log`.

## Known limits

- Full screen shows a white block in the left border in 4:3 menus (a Wine scaling leftover).
- Battles run at about 17–25 FPS on a Pi 4 at 800x600.
- If the Pi runs heavy background jobs (the working Pi runs an Internet-in-a-Box server), the
  game dips for a moment whenever they start. On that Pi this was eased by giving desktop
  sessions more CPU weight; it is a system change, so it is not in the script:
  `sudo systemctl set-property user.slice CPUWeight=1000` (undo: `sudo systemctl revert user.slice`).

## Pi 5

Untested. Two things to know:

- Raspberry Pi OS on a Pi 5 uses a kernel with 16 KB pages by default, which Hangover cannot run
  on. The script stops and says so. To switch to the 4 KB kernel, add the line
  `kernel=kernel8.img` to `/boot/firmware/config.txt` and reboot.
- The Mesa patch is for the V3D driver, which the Pi 5 also uses (a newer GPU generation). It
  should build and run, but its speed-up was only measured on the Pi 4.

## Troubleshooting

| Symptom | What to do |
|---|---|
| "apt would upgrade or remove the packages above" | Nothing was installed. Look at the list; rerun with `--allow-upgrades` if those changes are fine on this Pi. |
| Checksum mismatch | Delete the named file in `~/mtw/downloads` and rerun (usually a broken download). |
| A step failed | Fix the cause and rerun the same command; finished steps are skipped. To redo a finished step, delete its marker in `~/mtw/.setup/` (`apt`, `hangover`, `game`, `shim`, `mesa`, `prefix`). |
| Game window black 640x480 in `--window` mode | A full-screen run was killed hard and left a Wine setting behind. Run `WINEPREFIX=~/mtw/prefixes/hangover wine reg delete 'HKCU\Software\Wine\X11 Driver' /v EmulateModeset /f`. |
| "The game is already running" | `WINEPREFIX=~/mtw/prefixes/hangover wineserver -k` |

## What is not in the repo

The repo at https://github.com/KidKlatt/TWM_Pi holds only what the setup needs. The game is not
included: it is licensed to each owner, so everyone brings their own GOG installer. The
development record (experiment log, test harness, research scripts) is not published; a summary
is in [HOW-IT-WAS-MADE.md](HOW-IT-WAS-MADE.md).
