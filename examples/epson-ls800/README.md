# Epson LS800 (Android TV 14 laser projector)

Epson's Android TV module HA90 (built by Askey, board `sti6202d101`, Amlogic), firmware
UKG3.250826.001 (`u-2.05-U-0`, June 2026). Both halves work: the debloat list and the Clean Your TV
launcher as Home. Tested on 21 October 2026.

| File | What it is |
|---|---|
| `debloat.txt` | The packages disabled on this unit, grouped and commented, with what to keep and why (`build-and-install.sh` disables the stock launcher on top) |
| `config.json` | Dock order (6 columns, Apple TV layout) and hidden tiles; push it with `tools/theme.sh` |
| `restore.sh` | Undo everything below in one go |

All commands run from the repo root, with the projector's IP in `TV_HOST`.

## Turning ADB on

Settings → Device Preferences → About → press OK on **Build** 7 times, then Developer options. This
firmware only offers **Wireless debugging** (TLS), not plain network ADB, so pair once with Google's
`adb` ([platform-tools](https://developer.android.com/tools/releases/platform-tools)):

1. Developer options → Wireless debugging → on → **Pair device with pairing code**.
2. `adb pair <ip>:<pairing port> <code>`, then `adb connect <ip>:<port shown under IP address & Port>`.
3. `adb tcpip 5555`, so `tools/tv.py` (port 5555) works. It came back by itself after a restart here.

Android switches Wireless debugging off whenever Wi-Fi drops, so pair again if the port disappears.

## Install, from scratch or again

```bash
TV_HOST=<projector-ip> tools/disable.sh examples/epson-ls800/debloat.txt
TV_HOST=<projector-ip> launcher/build-and-install.sh
TV_HOST=<projector-ip> tools/theme.sh examples/epson-ls800
```

Building needs JDK 17 and the Android SDK (`ANDROID_HOME`, platform 34). On Android 14,
`build-and-install.sh` pushes the APK and runs `pm install -r` itself, so no on-screen confirmation is
needed.

### Extras set on this unit

```bash
# Names in the Inputs menu (this firmware has no Inputs screen in Settings to rename them).
# Undo: settings delete secure tv_input_custom_labels
TV_HOST=<projector-ip> tools/tv.py "settings put secure tv_input_custom_labels 'com.droidlogic.tvinput%2F.services.Hdmi2InputService%2FHW6,ARC:com.droidlogic.tvinput%2F.services.Hdmi3InputService%2FHW7,Nintendo%20Switch'"
```

HDMI 1 names itself over CEC ("PlayStation 5"). Epson's operation-log upload is switched off in
Projector Settings → Information → Projector Operation Log Information, not by disabling
`com.epson.logcollection` (that crashes the Information screen).

## Re-enable an app

```bash
TV_HOST=<projector-ip> tools/tv.py 'pm enable com.instagram.airwave'
```

Use the package name from `debloat.txt`. To put everything back, the launcher included:

```bash
TV_HOST=<projector-ip> examples/epson-ls800/restore.sh            # add --uninstall-launcher to remove it too
```

## Change the dock or the wallpaper

Edit `dock_apps` in `config.json` (four package names, left to right; `tools/tv.py 'pm list packages'`
lists them), then push it again:

```bash
TV_HOST=<projector-ip> tools/theme.sh examples/epson-ls800
```

For a wallpaper, put a `wallpaper.jpg` (1920x1080) next to `config.json` and push the same way;
`themes/README.md` has the dim and rotation options. `tools/theme.sh --reset` goes back to the
launcher's built-in config.

## Remote buttons & projector mainboard architecture

Home, Input, Settings, the projector-menu button, brightness, screen blanking (mute), the mic, Back
and volume all work without the stock launcher.

### Projector hardware keys (Brightness, Projector Settings, Screen Off / Blank)

The Epson LS800's Android TV module (`HA90`, Amlogic `sti6202d101`) communicates with the projector
mainboard over an internal USB composite device (`Multifunction with RNDIS` -> `usb0` at `192.168.48.2/24`
plus HID/ACM interfaces) managed by the vendor HAL `vendor.epson.hardware.maimif@1.0-service` and
`com.epson/.app.EpsonService` (`MaimIfKeyListenerClient` -> `KeyHandlerC`).

- **Keep all `com.epson.*` system packages enabled** (`com.epson`, `com.epson.droidsettings`,
  `com.epson.bizmodeui`, `com.epson.logcollection`, `com.epson.setupcustomizer`,
  `com.epson.flexiblepicture`, `com.epson.installation`, `com.epson.visualization`,
  `com.epson.warningui`). Disabling any of these or force-stopping `com.epson` without a clean reboot
  can break the `IMaimIfKeyListener` binding to `maimif@1.0-service`, causing the physical
  **Brightness**, **Projector Settings**, and **Screen Blank** buttons on the remote to go unresponsive.
- **Do NOT intercept `KEYCODE_SETTINGS` (176) in an AccessibilityService (`HomeKeyService`)**: on the
  LS800, intercepting `KEYCODE_SETTINGS` in `onKeyEvent` interferes with the projector settings menu key.
- If the projector hardware buttons ever stop responding after package changes, re-enable the Epson
  packages above and **reboot the projector** (`adb reboot`) so `maimif@1.0-service` and `EpsonService`
  re-establish their listener session cleanly.

### Firmware-intercepted buttons (`YouTube` & `ALL_APPS`)

Two buttons are handled by the firmware in `PhoneWindowManager` before any app or accessibility service
sees them:

- **YouTube** sends `KEYCODE_MACRO_3` (HID usage `0x0c008d` in `/vendor/usr/keylayout/Vendor_005d_Product_0001.kl`),
  which Epson's `PhoneWindowManager.startCustomApp` hard-codes to `com.google.android.youtube.tvmusic`
  (YouTube Music); on this firmware YouTube Music forwards straight to YouTube. Keep YouTube Music
  enabled, or the button opens its Play Store page instead. It cannot be remapped to another app.
- **Apps** (`KEYCODE_ALL_APPS`, keycode `284`, HID usage `0x0c01A2`) does nothing once the stock
  launcher (`com.google.android.tvlauncher`) is disabled: Android 14's `PhoneWindowManager` consumes it
  before accessibility services and sends `android.intent.action.ALL_APPS` only to the system launcher
  (returning `ActivityTaskManager START_ABORTED code=-91` when `tvlauncher` is disabled). Pressing
  **Down** on the remote from the home screen opens the full All Apps & Inputs panel directly.

## Cosy Home TV theme & 6-column Apple TV layout

In addition to the standard `cleanyourtv` launcher, this unit was tested with the
[**Cosy Home TV**](https://github.com/oliverjwroberts/cosy-home-tv) theme and layout ported into the
launcher:

- **Clock & Dynamic Valley Backdrop (`SceneView` + `Sun`)**: Large `Outfit` thin clock and date over a
  procedural valley scene that transitions smoothly through **Dawn**, **Day**, **Dusk**, and **Night**
  based on real sunrise/sunset times for the configured coordinates (e.g. Taipei `25.03, 121.56`), with
  periodic pixel-shifting (`SHIFT_DP`) and clean dock presentation (the subtitle hint above the dock is
  hidden).
- **6-Column Dock & Grid (Apple TV style)**: Both the home screen dock (`dock`) and the All Apps grid
  (`GRID_COLUMNS = 6`) display 6 tiles per row, with Netflix (`com.netflix.ninja` /
  `com.netflix.mediaclient`), YouTube, Spotify, LINE TV, PTS+, Projector Settings, and Play Store
  configured in the dock.
- **Frosted-Glass All Apps & Inputs Panel (`GlassPanel` + `Blur`)**: Pressing **Down** from the dock
  opens a real-time blurred frosted-glass overlay with top navigation chips (`回到時鐘` / `設定` and clock),
  a 6-column scrollable app grid (`RoomyScrollView`), and a bottom frosted-glass **Inputs (`輸入來源`)**
  bar listing the projector's HDMI inputs (`HDMI 1`–`3`, including CEC labels such as PlayStation 5 /
  Google TV).
- **Hiding/Disabling Amlogic Updater (`com.droidlogic.updater`)**: To remove the built-in `Updater` tile
  from the app drawer, both disable it via ADB (`pm disable-user --user 0 com.droidlogic.updater`) and
  include `com.droidlogic.updater` in `hidden_apps` / `R.string.hidden`.

## When you are done

Switch **USB debugging** and **Wireless debugging** off in Developer options, revoke the debugging
authorisations there, restart, and check that ADB is closed:

```bash
tools/venv/bin/python tools/adb-closed.py <projector-ip>
```
