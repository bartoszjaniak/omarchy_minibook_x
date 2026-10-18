# MiniBook X N150 customization inventory

Date: **2026-10-18**.

This inventory is based on local files and OpenCode session history covering
the MiniBook, Polish input and gestures, boot configuration, and battery usage.
It distinguishes current implementations from earlier proposals and unverified
results. It is not a clean-install test or validation on a second device.

All `~` paths below refer to the source user's account.
Conversation exports are not included in the public repository.

## 1. Display and touchscreen

### 1.1. Correct orientation in laptop mode — implemented

- Native panel orientation: portrait, `1200×1920`, connector `DSI-1`.
- Default Hyprland rotation: `transform = 3`, landscape orientation.
- Interface scale: `1.25`; `GDK_SCALE = 1`.
- File: `~/.config/hypr/monitors.lua`.
- Installer consideration: the current monitor entry uses `output = ""`,
  making it a fallback rule for other displays too. The distributable version
  should target the detected internal panel specifically.

### 1.2. Automatic display and touchscreen rotation — implemented

- `minibook-autorotate` receives orientation from `monitor-sensor --accel`.
- Rotates the monitor and touchscreen together using `hyprctl eval`.
- Mapping: `normal → 0`, `left-up → 1`, `bottom-up → 2`, `right-up → 3`.
- Saves the last transform in `~/.local/state/minibook-transform`.
- Files: `~/.local/bin/minibook-autorotate`, `~/.config/hypr/autostart.lua`.
- Dependency: `iio-sensor-proxy` (locally `3.9-1`).

### 1.3. Preserving rotation across configuration reloads — fixed

- Previously, `hyprctl reload` could reset the touchscreen transform to `0`,
  misaligning touch coordinates with the display.
- Added reapplication of the saved monitor and touchscreen transform.
- File: `~/.local/state/omarchy/toggles/hypr/minibook-rotation.lua`.
- Despite its location in the state directory, this file is integration code.
  The installer must include it without copying the device's current state.

### 1.4. Automatic rotation lock — implemented

- Command: `minibook-rotation-lock [toggle|on|off|status]`.
- A state file preserves the lock across restarts.
- Includes an OSD message and a checked menu item when locked.
- Menu: `Trigger → Hardware → Rotation Lock`.
- Files: `~/.local/bin/minibook-rotation-lock`,
  `~/.config/omarchy/extensions/omarchy-menu.jsonc`.

## 2. Sensors and tablet mode

### 2.1. Enabling the second accelerometer — implemented

- Firmware enumerates the display sensor but not the base sensor.
- A udev rule instantiates an `mxc4005` device at I²C address `0x15`.
- On this unit: display on controller `0000:00:15.1`,
  base on `0000:00:15.0`.
- `ACCEL_LOCATION=display/base` tags select the correct sensor for display
  orientation.
- System file: `/etc/udev/rules.d/99-minibook-x-accel.rules`.
- Before publication: the rule assumes `iio:device0` and a particular hardware
  layout. Detection and safeguards are needed to avoid instantiating the wrong
  device on another hardware revision.

### 2.2. Detecting the folded tablet position — implemented

- A custom daemon compares gravity vectors from both accelerometers instead
  of relying on the nonfunctional ACPI tablet-mode switch.
- Polling interval: `0.5 s`; angular thresholds `45°/65°` provide hysteresis.
- Folding disables the internal keyboard and touchpad.
- Unfolding enables them again.
- State flag: `~/.local/state/minibook-tablet-mode`.
- File: `~/.local/bin/minibook-tablet-mode`; started through Hyprland autostart.
- Input lock restoration after reload:
  `~/.local/state/omarchy/toggles/hypr/minibook-tablet.lua`.
- Dependency: Python 3; input device names are currently hardcoded.

### 2.3. Turning off the keyboard backlight — conditional support only

- The daemon includes code to turn the backlight off when folded and restore
  it when unfolded.
- Uses `omarchy-brightness-keyboard` only if a `*kbd_backlight*` LED exists
  in sysfs.
- **This unit's firmware does not expose such an LED.** Automatic backlight
  control must not be advertised as working on this unit; manual Fn-key
  control remains necessary.

## 3. On-screen keyboard (OSK)

### 3.1. Local wvkbd build — completed

- Built wvkbd `0.20` from source without installing the AUR package.
- Binaries: `~/.local/bin/wvkbd-deskintl` and `wvkbd-mobintl`.
- Default layout: `deskintl`; controller-configured height: `320`.
- Source: `https://git.sr.ht/~proycon/wvkbd/archive/v0.20.tar.gz`.
- Build requirements to verify in the installer: C toolchain/make, pkg-config,
  Wayland/wayland-scanner, cairo, pango, fontconfig, and libxkbcommon;
  scdoc is needed when building documentation.
- The local build is not updated by pacman.

### 3.2. Keyboard controller and tablet-mode integration — implemented

- `minibook-osk`: `run|start|stop|show|hide|toggle|status`.
- Tablet mode starts the OSK hidden; folding alone does not force it to appear.
- Unfolding stops the OSK; manual toggle can also start it in laptop mode.
- Uses signals for control and a visibility flag for the bar button.
- Environment overrides: `MINIBOOK_OSK_BIN`, `MINIBOOK_OSK_HEIGHT`,
  `MINIBOOK_OSK_FONT`, `MINIBOOK_OSK_THEME`.
- File: `~/.local/bin/minibook-osk`.

### 3.3. Automatic display when text fields gain focus — implemented

- Added a custom `minibook-osk-im` D-Bus bridge between fcitx5 and wvkbd.
- Reason: fcitx5 holds the input-method role required by `wvkbd --auto`.
- `NotifyIMActivated` shows the OSK in tablet mode;
  `NotifyIMDeactivated` hides it.
- Fixed disappearance after the first virtual key: ignore the standalone
  hide request and reassert OSK mode after approximately `400 ms`.
- The bridge starts at login; automatic behavior is limited to tablet mode.
- Files: `~/.local/bin/minibook-osk-im`, `~/.config/hypr/autostart.lua`.
- Dependencies: fcitx5 with its virtualkeyboard frontend, Python 3, and
  PyGObject/Gio (`python-gobject`). Coverage depends on application integration
  with the input method.
- `MINIBOOK_OSK_ALWAYS=1` bypasses the tablet-mode restriction.

### 3.4. Colors matching the Omarchy theme — implemented

- The controller reads `~/.local/state/omarchy/current/theme/colors.toml`.
- Matches the background, keys, labels, and pressed-key accent.
- Colors are loaded when the OSK process starts, not on every show request.
- Changing the theme while the OSK is running requires restarting the OSK.

### 3.5. Manual touch, shortcut, and menu controls — implemented

- Bar button: `bart.osk` plugin, visible only in tablet mode and highlighted
  when the keyboard is shown.
- Files: `~/.config/omarchy/plugins/bart.osk/{manifest.json,Osk.qml}`.
- Button registration: `~/.config/omarchy/shell.json`, left bar section.
- Shortcut `SUPER + SHIFT + K`: `~/.config/hypr/bindings.lua`.
- Menu: `Trigger → Hardware → On-screen Keyboard` in the JSONC extension.
- Publication requires a neutral plugin namespace instead of `bart.osk`.

### 3.6. Lock screen — unresolved limitation

- Omarchy uses a Quickshell session-lock screen, not hyprlock.
- As a separate layer-shell surface, wvkbd does not provide an OSK there.
- Password entry requires unfolding the device or another available method.
- Fingerprint support was only suggested, not implemented; a compatible
  reader has not been confirmed.

## 4. Touchpad, gestures, and animations

Implemented in `~/.config/hypr/input.lua`:

| Gesture / setting | Current action |
| --- | --- |
| Two-finger scrolling | Natural: `natural_scroll = true` |
| Four-finger horizontal swipe | Interactive workspace switching |
| Three-finger swipe up | `omarchy-menu toggle` — quick menu |
| Three-finger swipe down | `wtype -k Escape` — Esc in the focused context |
| Four-finger swipe up | Enable fullscreen (`set`) |
| Four-finger swipe down | Disable fullscreen (`unset`) |
| Workspace switch commit threshold | `workspace_swipe_cancel_ratio = 0.2` |

- The original three-finger workspace gesture was replaced with four fingers;
  do not install both historical variants.
- Fast-swipe detection retains its default threshold:
  `workspace_swipe_min_speed_to_force = 30`. This measures speed, not acceleration.
- Enabled the `workspaces` animation: `slide`, `easeOutQuint`, `speed = 3`.
  Smoothly completes the transition after release instead of snapping.
- The animation also applies to keyboard-driven workspace switching.
- Animation file: `~/.config/hypr/looknfeel.lua`.
- Esc gesture dependency: `wtype`.

## 5. Polish characters — implemented optional preference

- Fixed missing AltGr characters with `input.kb_layout = "pl"`
  in `~/.config/hypr/input.lua`.
- The cause was fallback to `us` when system settings lacked `XKBLAYOUT`.
- This is a language preference, not a MiniBook requirement. The installer
  should preserve the existing layout or ask the user.
- Adding `XKBLAYOUT=pl` to `/etc/vconsole.conf` was suggested but must not be
  treated as a confirmed system-level change; the Lua override is confirmed.

## 6. Boot and dual boot — separate optional module

### 6.1. Boot-time orientation — implemented according to session history

- Added kernel parameter `video=DSI-1:panel_orientation=right_side_up`
  in `/etc/default/limine`; its presence in that file is confirmed.
- Set `interface_rotation: 90` in `/boot/limine.conf`; earlier verification
  reported that it survived `limine-update`.
- `/boot` currently requires elevated privileges, so this part was not
  rechecked during the inventory.
- Do not copy the complete kernel command line: it contains local encrypted
  disk identifiers and installation-specific boot settings.

### 6.2. Windows in Limine and a simplified menu — work completed

- Added Windows chainloading from a separate EFI system partition.
- Intended menu: `Omarchy`, `Windows`, and a `Snapshots` directory.
- Omarchy uses a direct UKI entry with automatic boot after `5 s`.
- Retained the managed OS group, hidden on UEFI; do not remove or disable
  snapshots when reproducing the final configuration.
- `/usr/local/bin/limine-final` exists to recreate the menu layout.
- The current menu still needs to be read as root. The script's presence
  alone does not confirm the current menu or its resilience to updates.
- Before distribution: detect ESP and UKI, check Secure Boot and the embedded
  UKI command line, create backups, and test kernel/snapshot updates.
- Intermediate versions that hid snapshots were superseded by the final
  layout retaining the Snapshots directory.

## 7. Items that are not implemented MiniBook improvements

- Battery and workload analysis: measurements and a chart were produced,
  but no persistent power optimization or process monitoring was deployed.
- Proposed 1 Hz refresh: not implemented; the panel reported only a 50 Hz mode.
- A 95% charging limit: not implemented; no driver control interface available.
- Kanban/AI agent and its widget: a separate project, unrelated to the hardware.
- Personal bar, clock, and application settings should not be copied.
- `rtc_cmos.use_acpi_alarm=1` and resume configuration already existed at the
  first diagnosis in the MiniBook session. There is no evidence to attribute
  them to these changes or justify copying a fixed hibernation offset to
  another disk.
- USB autosuspend disabling and the logind delay drop-in belong to the
  `omarchy-settings` package, not our local hardware fixes.

## 8. Requirements for a single installer

1. One entry point with selectable modules: hardware/tablet/OSK, gestures,
   language, and boot/dual boot. Do not require Windows or a Polish layout.
2. Verify DMI, Omarchy/Hyprland versions (Lua), sensors, monitors, and input
   devices. Do not claim N100 support without testing.
3. Do not modify `/usr/share/omarchy`; use user extensions and the necessary
   system rule in `/etc/udev/rules.d`.
4. Remove hardcoded `/home/bart` paths, personal plugin namespaces, and sensor
   numbering assumptions; validate parameters instead of copying blindly.
5. Safely merge Lua/JSONC/JSON files; provide backups, idempotency, a dry-run
   mode, and uninstall support.
6. Never copy secrets, session exports, browser data, disk UUIDs, resume
   offsets, or current rotation-lock/tablet/OSK flags.
7. Confirm dependency installation; build wvkbd as an unprivileged user with
   a pinned version and verified source checksum.
8. Manage daemon lifecycles without duplicates. Avoid broad `pkill -f`
   patterns, which killed the executing shell in earlier sessions.
9. Validate with `hyprctl reload` and `hyprctl configerrors`; check script
   syntax and provide an input-recovery procedure for tablet-mode failures.
10. Test a clean installation and physically test rotation, folding, OSK,
    suspend/resume, configuration reloads, and logging in again.
11. Review licensing and attribution for the gist and wvkbd before publication.

## 9. Minimum files to collect in the next stage

- Five `~/.local/bin/minibook-*` scripts.
- Two `minibook-{rotation,tablet}.lua` modules from the toggles directory.
- The `99-minibook-x-accel.rules` rule.
- OSK plugin manifest and QML, after changing its namespace.
- Extracted monitor, autostart, shortcut, gesture, animation, and menu
  configuration snippets, not entire personal configurations.
- A wvkbd build recipe and licensing information, rather than binaries alone.
- Optional, reviewed, generalized boot-orientation and Limine support.

This stage produced documentation only. Executable code has not yet been
collected, no installer has been run, and the running system's configuration
has not been changed during this inventory.
