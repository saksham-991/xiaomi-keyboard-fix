# Xiaomi Pad 7 / 7 Pro Keyboard Fix

Hardware keyboard fixes for the **Xiaomi Pad 7 (`uke`)** and **Pad 7 Pro (`muyu`)** running custom AOSP or crDroid ROMs.

Restores typing at boot, fixes landscape arrow key rotation, enables the Caps Lock LED, adds keyboard backlight shortcuts, and resolves excessive CPU usage caused by Xiaomi's touch telemetry service.

## Quick Install — Magisk / KernelSU

1. Download **Xiaomi-Pad7-Keyboard-Fix-All-in-One.zip** from the [latest release](https://github.com/saksham-991/xiaomi-keyboard-fix/releases/tag/Magisk-Module).
2. Install the ZIP through Magisk or KernelSU.
3. Reboot your tablet.

## Problems Solved

| Problem | Fix |
| --- | --- |
| Keyboard connects but cannot type | `xiaomi_kbd_service.sh` automates HID rebinding at boot and when the keyboard connects. |
| Keyboard stops working after a few minutes | `kbd_auth.sh` and `tokenhelper` provide MiAuth tokens locally through TrustZone. |
| Arrow keys are rotated 90° in landscape | `kbd_idc.sh` generates the IDC configuration early in boot with `device.internal = 0`. |
| Caps Lock LED does not work | `kbd_leds.sh` watches input events and synchronizes the LED state. |
| Keyboard backlight cannot be adjusted | `kbd_leds.sh` adds configurable brightness shortcuts. |
| Tablet overheats or one CPU core stays at 100% | `kbd_no_toucheventcheck.sh` disables the broken MiSight touch telemetry loop. |

> The 30hz touchpad issue (choppy pointer) is also solved in the magisk module as it is a simple toggle "Disable HW Overlays" IN developer options, I added it here because the touchpad is on the same keyboard folio case, otherwise its basically a ROM issue, rather than a Keyboard issue.

### Security and Privacy

No keystrokes are logged. Authentication keys are not extracted or sent online. Authentication tokens are computed locally using the stock firmware's `devauth` trustlets.

## Manual Installation

Use this method to install the fix through `/data/adb` boot scripts without installing a Magisk module.

### Prerequisites

- A Xiaomi Pad 7 or Pad 7 Pro running a custom ROM with stock HyperOS vendor blobs.
- Root access through Magisk or KernelSU.
- ADB enabled.
- The following three proprietary HAL files extracted from the `odm.img` partition of a stock HyperOS fastboot ROM:

  ```text
  /odm/bin/midevauthd
  /odm/lib64/libmidevauth.so
  /odm/lib64/vendor.xiaomi.hardware.aidl.midevauth-V1-ndk_platform.so
  ```

- The `tokenhelper` binary from the [latest release](https://github.com/JohnnyFFM/xiaomi-keyboard-fix/releases/latest), or a manually built copy.
- The scripts from this repository's `device/` directory.

Check that the required firmware and device interface are present:

```sh
adb shell su -c 'ls /vendor/firmware_mnt/image/devauth.*; ls /dev/smcinvoke'
```

### 1. Stage the Files

Run this command on your computer from the repository directory, with the extracted HAL files and `tokenhelper` in the current directory:

```sh
adb push \
  midevauthd \
  libmidevauth.so \
  vendor.xiaomi.hardware.aidl.midevauth-V1-ndk_platform.so \
  tokenhelper \
  device/kbd_auth.sh \
  device/kbd_leds.sh \
  device/service.d-kbdauth.sh \
  device/xiaomi_kbd_service.sh \
  device/post-fs-data-kbd-idc.sh \
  device/kbd_no_toucheventcheck.sh \
  /data/local/tmp/
```

### 2. Install the Files and Set Permissions

```sh
adb shell su -c '
mkdir -p /data/adb/kbdauth /data/adb/service.d /data/adb/post-fs-data.d

cp \
  /data/local/tmp/midevauthd \
  /data/local/tmp/libmidevauth.so \
  /data/local/tmp/vendor.xiaomi.hardware.aidl.midevauth-V1-ndk_platform.so \
  /data/local/tmp/tokenhelper \
  /data/local/tmp/kbd_auth.sh \
  /data/local/tmp/kbd_leds.sh \
  /data/adb/kbdauth/

cp /data/local/tmp/service.d-kbdauth.sh \
  /data/adb/service.d/kbdauth.sh

cp /data/local/tmp/xiaomi_kbd_service.sh \
  /data/adb/service.d/xiaomi_kbd_service.sh

cp /data/local/tmp/post-fs-data-kbd-idc.sh \
  /data/adb/post-fs-data.d/kbd_idc.sh

cp /data/local/tmp/kbd_no_toucheventcheck.sh \
  /data/adb/post-fs-data.d/kbd_no_toucheventcheck.sh

chmod 755 \
  /data/adb/kbdauth/midevauthd \
  /data/adb/kbdauth/tokenhelper \
  /data/adb/kbdauth/kbd_auth.sh \
  /data/adb/kbdauth/kbd_leds.sh \
  /data/adb/service.d/kbdauth.sh \
  /data/adb/service.d/xiaomi_kbd_service.sh \
  /data/adb/post-fs-data.d/kbd_idc.sh \
  /data/adb/post-fs-data.d/kbd_no_toucheventcheck.sh
'
```

### 3. Reboot

```sh
adb reboot
```

### 4. Verify the Installation

After rebooting, check the logs:

```sh
# Check daemon status. Look for: vnd=1
adb shell su -c 'cat /data/adb/kbd/svc.log'

# Check authentication. Look for: -> STEP5 sent (real token)
adb shell su -c 'cat /data/adb/kbd/auth.log'

# Check EVDEV discovery for Caps Lock and keyboard events.
adb shell su -c 'cat /data/adb/kbd/leds.log'
```

## Backlight Shortcuts and Configuration

| Action | Shortcut |
| --- | --- |
| Increase brightness | `Ctrl` + `Alt` + `4` |
| Decrease brightness / turn off | `Ctrl` + `Alt` + `3` |

To customize the backlight, edit the tunables at the top of:

```text
/data/adb/kbdauth/kbd_leds.sh
```

| Setting | Purpose |
| --- | --- |
| `BL_STEP` | Brightness adjustment step |
| `BL_DEFAULT` | Default brightness |
| `BL_MAX` | Maximum brightness |
| `BL_NEED_ALT` | Set to `0` to use `Ctrl` + `3` / `4` without `Alt`. |

> Shortcuts are observed rather than consumed, so choose combinations that do not conflict with your apps.

## Architecture

### Installed Files

| Location | Purpose |
| --- | --- |
| `/data/adb/post-fs-data.d/kbd_idc.sh` | Writes the IDC configuration early in boot to fix orientation and external-device flags. |
| `/data/adb/post-fs-data.d/kbd_no_toucheventcheck.sh` | Stops the broken telemetry daemon responsible for excessive CPU usage. |
| `/data/adb/service.d/kbdauth.sh` | Starts `midevauthd`, `kbd_auth.sh`, and `kbd_leds.sh`. |
| `/data/adb/service.d/xiaomi_kbd_service.sh` | Rebinds the HID node when the keyboard connects to enable input. |
| `/data/adb/kbdauth/` | Stores the HAL files, `tokenhelper`, and authentication and LED scripts. |

### Technical Notes

#### Authentication

- **`midevauthd`** is Xiaomi's proprietary HAL service, copied from the stock ROM. It uses `vndbinder` with `LD_LIBRARY_PATH=/vendor/lib64:/system/lib64`.
- **`tokenhelper`** is a small, statically compiled `arm64-v8a` Binder client built with NDK r26. It calls `IMidevauthService.devauth_token_get`.
- **`kbd_auth.sh`** uses `tokenhelper` to provide the authentication tokens required to keep the keyboard working.

#### HID Rebinding

`xiaomi_kbd_service.sh` is required to prevent Android from leaving the HID device disabled. It handles rebinding at boot and when the keyboard connects.

#### CPU Usage and Touch Telemetry

`kbd_no_toucheventcheck.sh` disables Xiaomi's `/odm/bin/toucheventcheck`, a MiSight touch telemetry service.

On custom ROMs, deleting or rebinding input nodes can cause the daemon's dump thread to enter a user-space busy loop, keeping one CPU core at 100%. Disabling this service does not affect touch or keyboard input.

Logs are written to:

```text
/data/adb/kbd/notec.log
```

#### Caps Lock LED

Android emits `LED_CAPSL` only on subsequent key events, and the keyboard hardware updates LEDs on activity. `kbd_leds.sh` watches `KEY_CAPSLOCK` presses directly and uses `LED_CAPSL` events to correct state drift.

#### IDC File Precedence

Android prioritizes `/system/usr/idc` over `/data/system/devices/idc`.

The manual installation avoids mounting a competing copy under `/system`, allowing the configuration under `/data` to take effect without being shadowed by that copy.

## Building `tokenhelper`

The GitHub Actions workflow at [`.github/workflows/build.yml`](.github/workflows/build.yml) cross-compiles `tokenhelper` for `arm64-v8a` using NDK r26.

Pushing a release tag beginning with `v` automatically builds and publishes the binaries.

The `tokenhelper` binary and source are unchanged since v2.

## Uninstalling

### Magisk / KernelSU Installation

Remove the module through Magisk or KernelSU, then reboot.

### Manual Installation

Run the following on your computer:

```sh
adb shell su -c '
rm -f \
  /data/adb/service.d/kbdauth.sh \
  /data/adb/service.d/xiaomi_kbd_service.sh \
  /data/adb/post-fs-data.d/kbd_idc.sh \
  /data/adb/post-fs-data.d/kbd_no_toucheventcheck.sh \
  /data/system/devices/idc/Vendor_15d9_Product_00a3.idc

pkill -f midevauthd
pkill -f kbd_auth.sh
pkill -f kbd_leds.sh

start toucheventcheck
'
```

Optionally, remove the remaining binaries and scripts:

```sh
adb shell su -c 'rm -rf /data/adb/kbdauth'
```

Reboot to complete removal:

```sh
adb reboot
```

## Legacy Version

Version 2, which contains the authentication-only implementation, is archived on the [`v2-backup`](https://github.com/JohnnyFFM/xiaomi-keyboard-fix/tree/v2-backup) branch.

## License

This project is licensed under the [MIT License](LICENSE).

Reverse-engineered from system components for interoperability and repair purposes.
