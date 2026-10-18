# Omarchy — CHUWI MiniBook X N150

Omarchy customizations for the CHUWI MiniBook X N150: display and touchscreen
rotation, tablet mode, an on-screen keyboard, and touchpad gestures.

## Project status

**Stage 1: inventory.** See [INVENTORY.md](INVENTORY.md) for implemented changes,
source file locations, limitations, and requirements for the future installer.

There is no installer yet. Do not copy the author's entire configuration to
another computer: it contains user-specific paths, disk settings, and personal
preferences.

Configuration checked during the inventory (2026-10-18):

- CHUWI MiniBook X, Intel N150;
- Omarchy `4.0.5-1`;
- Hyprland `0.56.2`, Lua configuration;
- kernel `7.2.8-9-omarchy`.

Support for the N100 and other firmware versions requires separate testing.

## Origins

Sensor and tablet-mode support started from
[kyryl-bogach's gist](https://gist.github.com/kyryl-bogach/86eac430c4135ae6b200c6467da081a4).
Local additions include restoring rotation after Hyprland configuration reloads,
wvkbd integration, an fcitx5 bridge, a bar button, and gestures.

On-screen keyboard: [wvkbd](https://git.sr.ht/~proycon/wvkbd), locally built
version `0.20`. Before publishing code or binaries, review the licensing terms
of all included files and preserve required attribution and license notices.
A license for this repository has not been selected yet.
