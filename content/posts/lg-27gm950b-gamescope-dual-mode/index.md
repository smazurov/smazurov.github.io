---
title: "LG 27GM950B Dual Mode: 5K Desktop, 330 Hz Gamescope"
date: 2026-11-15T22:09:27-07:00
draft: false
description: "A Gamescope Lua script switches the LG 27GM950B from 5K165 to 1440p330 over DDC/CI, then restores the saved GNOME display configuration on exit or crash. Tested on Fedora 45 with GNOME Wayland."
tags:
  - linux
  - gamescope
  - gnome
  - monitors
  - lua
takeaways:
  - LG 27GM950B Dual Mode uses VCP 0xB1 with explicit on/off values; changing a game's rendering resolution doesn't enable the hardware mode
  - One Lua file runs inside Gamescope at startup and outside it as a systemd user service for crash recovery
  - The installer checks prerequisites without switching modes and installs one user-local Lua file
  - The tested setup is GNOME Wayland; keep the pointer on the gaming LG during launch and reconnection
tested_on:
  - LG 27GM950B over DisplayPort, with a second 4K60 display
  - Fedora 45, Linux 7.3.3, GNOME Shell and Mutter 51.1
  - Gamescope 3.16.29, LuaJIT 2.1.1767980792
  - systemd 262, ddcutil 3.0.2, GLib/GIO 2.90.1
---

The LG 27GM950B can run at 5120×2880 and 165 Hz, or 2560×1440 and 330 Hz. Switching between them requires changing the monitor's Dual Mode setting and GNOME's display configuration. Launching a 1440p Gamescope window doesn't handle that hardware switch.

I wanted 5K on my desktop and 330 Hz in games, with my original layout back when the game closed. I also wanted to keep stock Gamescope and my existing Steam launch options. I wrote [gamescope-lg-dual-mode][1] as a startup script, with one Lua file handling both the switch and recovery.

## Switching the monitor

I used manufacturer-specific VCP feature `0xB1`, sent over DDC/CI through `ddcutil`. The [StreamController LG monitor plugin][2] documents the high-refresh values for the 45GX950A, and [lg-dual-mode][3] documents the LG Switch command format. These values work on my 27GM950B:

| Operation | VCP feature | Value |
| --- | --- | --- |
| Enable high-refresh Dual Mode | `0xB1` | `0x2100` |
| Return to normal mode | `0xB1` | `0x2200` |

I use separate on/off commands so recovery can repeat a command after an interrupted launch without toggling the monitor back.

The switch briefly blanks the monitor and changes its advertised modes. The watcher waits for Mutter to report 2560×1440 above 300 Hz, then selects it with the saved VRR preference. I use those modes as the state check rather than `B1` readback.

## Keeping my desktop layout

My gaming LG uses 2× scaling. The other LG runs at 4K60 with 1.5× scaling. When a game closes, I want both displays back at those settings and in the same positions.

I use [Mutter's DisplayConfig interface][4] to read and restore that state through `GetCurrentState` and `ApplyMonitorsConfig`. Before the first hardware write, the watcher verifies that Mutter accepts the configuration and saves a recovery snapshot. That includes modes, scaling, positions, transforms, primary display and color-mode settings. There's no fixed desktop preset to maintain.

The [Lua implementation][5] calls GLib/GIO through LuaJIT FFI and stores the snapshot as typed GVariant data, without extra Lua D-Bus or JSON libraries. An exclusive lock prevents two managed Gamescope sessions from changing the monitor at once.

## Surviving a Gamescope crash

I couldn't rely on cleanup inside Gamescope. After `SIGKILL`, its Lua interpreter is gone too.

The startup script runs the same file under the host LuaJIT interpreter in a transient systemd user service. That watcher owns the saved state and waits on Gamescope's pidfd, without polling DDC during gameplay.

When Gamescope exits, the watcher switches Dual Mode off if it enabled it, waits for the saved mode to return, and restores the desktop. If the watcher itself is killed, systemd runs the file through `ExecStopPost` in recovery mode. A failed restore keeps the snapshot for another attempt.

The service exists for the managed session, not as a permanent daemon or login service. Recovery still needs the user session and runtime snapshot; I haven't tested it across logout, reboot or power loss.

## Installing it

My setup is GNOME Wayland with the 27GM950B. The script needs Gamescope with Lua scripting, a separate LuaJIT interpreter, `ddcutil` with DDC access, and systemd 254 or newer. The [README][1] lists the full dependencies.

The installer checks the monitor and enabled layout, Gamescope scripting, systemd user services, pidfds, Mutter access and a DDC brightness read. It refuses mirrored LG layouts. Preflight doesn't install packages or change monitor settings.

These commands use the [v0.1.0 release][7]:

```sh
# Read-only checks; installs nothing.
curl -fsSL https://github.com/smazurov/gamescope-lg-dual-mode/releases/download/v0.1.0/install.sh | bash -s -- --check

# Run preflight, then install one Lua file.
curl -fsSL https://github.com/smazurov/gamescope-lg-dual-mode/releases/download/v0.1.0/install.sh | bash
```

Run it as your desktop user, without `sudo`. The installer fetches both files from the same pinned release before running them and removes the temporary downloads afterward.

That leaves one file at `~/.config/gamescope/scripts/90-lg-dual-mode.lua`, or the equivalent under `XDG_CONFIG_HOME`. [Gamescope's loader][6] reads it at startup. The fullscreen 1440p launch command stays the same:

```sh
gamescope --backend wayland -w 2560 -h 1440 -W 2560 -H 1440 -f -- %command%
```

The script only manages matching fullscreen 1440p launches; it skips windowed launches and other resolutions. Put `GAMESCOPE_LG_DUAL_MODE=0` before `gamescope` to bypass it for one launch.

I hit one multi-monitor problem: the gaming LG switched modes, but the cube opened on the 4K LG. GNOME still chooses where the window goes.

**Leaving the pointer on the gaming LG during launch and reconnection fixed placement in my tests.** The script doesn't force an output.

## What I tested

I tested the 27GM950B over DisplayPort on Fedora 45, GNOME/Mutter 51.1 and Gamescope 3.16.29. The [validation report][8] has the full setup.

I checked Wayland output names against presentation feedback to verify that Gamescope presented on `DP-1` at 1440p330 with VRR selected. I also confirmed that the rotating cube was visible on the gaming LG.

| Exit condition | Saved desktop restored |
| --- | --- |
| Normal Gamescope exit | Yes |
| Gamescope killed with `SIGKILL` | Yes |
| Watcher killed with `SIGKILL`, followed by systemd cleanup | Yes |

The restored modes, refresh-rate mode, layout, scaling, transforms, primary display and color settings matched the snapshot in all three cases.

The 67 hardware-free tests and linters also passed on Fedora 45 and stock Ubuntu 24.04. They cover rollback, locking, command failures and recovery snapshots. The Ubuntu checks are hardware-free; my desktop testing is on Fedora.

I haven't live-tested launches starting in Dual Mode, multiple supported LGs or physical hotplug. Those paths have hardware-free coverage only. Gamescope's [documentation][9] calls scripting "currently experimental and subject to change massively."

## Recovery and uninstall

If automatic restoration fails, I can reapply the saved monitor and display settings with the same file. Close managed Gamescope sessions before running it:

```sh
luajit -E "${XDG_CONFIG_HOME:-$HOME/.config}/gamescope/scripts/90-lg-dual-mode.lua" --recover
```

The snapshot stays at `$XDG_RUNTIME_DIR/gamescope-lg-dual-mode/session.gv` if restoration fails. If the connected monitors have changed, the script refuses to replace the layout instead of silently removing a newly connected display.

The watcher logs to my user journal:

```sh
journalctl --user -u 'gamescope-lg-lua-*' -n 100
```

To uninstall:

```sh
curl -fsSL https://github.com/smazurov/gamescope-lg-dual-mode/releases/download/v0.1.0/install.sh | bash -s -- --uninstall
```

Uninstall removes only its own inactive copy. It refuses symlinks and unrelated files, and keeps recovery snapshots.

## FAQ

**Will this work on any 5K LG monitor?**

I've only tested the 27GM950B, and that's the only enabled profile. The script matches `GSM / 0x5cf8`, not a generic LG name or my serial. Other models need their own profiles and hardware tests for their modes and firmware.

**Does it preserve HDR?**

I haven't tested HDR rendering. The script saves and restores GNOME's color-mode settings.

[1]: https://github.com/smazurov/gamescope-lg-dual-mode
[2]: https://github.com/tyvsmith/streamcontroller-lg-monitor-control
[3]: https://github.com/dftpnd/lg-dual-mode
[4]: https://github.com/GNOME/mutter/blob/main/data/dbus-interfaces/org.gnome.Mutter.DisplayConfig.xml
[5]: https://github.com/smazurov/gamescope-lg-dual-mode/blob/v0.1.0/gamescope-lg-dual-mode.lua
[6]: https://github.com/ValveSoftware/gamescope/blob/master/src/Script/Script.cpp
[7]: https://github.com/smazurov/gamescope-lg-dual-mode/releases/tag/v0.1.0
[8]: https://github.com/smazurov/gamescope-lg-dual-mode/blob/v0.1.0/docs/VALIDATION.md
[9]: https://github.com/ValveSoftware/gamescope/blob/master/scripts/README.md
