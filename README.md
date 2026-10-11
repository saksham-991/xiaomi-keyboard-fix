# Xiaomi Pad 7 / 7 Pro Keyboard Fix (crDroid / AOSP)

Fix hardware keyboard functionality on uke (Pad 7) and muyu (Pad 7 Pro) running custom AOSP or crDroid ROMs. Restores instant typing at boot, corrects landscape arrow key rotation, enables the Caps Lock LED, adds backlight control shortcuts, and fixes CPU battery drain/overheating.

---

## Quick Install (Magisk / KernelSU Module)

If you want an automated flashable install:
1. Download Xiaomi-Pad7-Keyboard-Fix-All-in-One.zip from Releases (../../releases/latest).
2. Flash the .zip inside Magisk or KernelSU.
3. Reboot your tablet.

---

## Problems Solved

- Keyboard connected but cannot type: xiaomi_kbd_service.sh automates HID rebind loop on boot/attach.
- Stops working after a few minutes: kbd_auth.sh + tokenhelper passes MiAuth tokens locally via TrustZone.
- Arrow keys rotated 90 deg in landscape: kbd_idc.sh handles early dynamic IDC generation (device.internal = 0).
- Caps Lock LED inactive: kbd_leds.sh real-time event watcher handles LED state.
- Backlight cannot be controlled: kbd_leds.sh maps custom hardware key combinations.
- Tablet runs warm / 100% CPU on 1 core: kbd_no_toucheventcheck.sh disables broken MiSight telemetry loop.

Security & Privacy Notice: No keys are logged, extracted, or sent online. Authentication tokens are computed locally using stock firmware devauth trustlets.

---

## Manual Deployment (No Magisk Module Overhead)

For users who prefer clean /data/adb boot scripts without running a full Magisk module.

Prerequisites:
- Pad 7 / 7 Pro running a custom ROM with stock HyperOS vendor blobs (Check via: ls /vendor/firmware_mnt/image/devauth.* and ls /dev/smcinvoke).
- Root (Magisk / KernelSU) and ADB enabled.
- 3 Proprietary HAL files extracted from a stock HyperOS fastboot ROM odm.img partition:
  * /odm/bin/midevauthd
  * /odm/lib64/libmidevauth.so
  * /odm/lib64/vendor.xiaomi.hardware.aidl.midevauth-V1-ndk_platform.so
- tokenhelper binary from Releases (../../releases/latest) or built manually.

Installation Steps:

1. Stage all required binaries and scripts to /data/local/tmp via ADB:
adb push midevauthd libmidevauth.so vendor.xiaomi.hardware.aidl.midevauth-V1-ndk_platform.so tokenhelper device/kbd_auth.sh device/kbd_leds.sh device/service.d-kbdauth.sh device/xiaomi_kbd_service.sh device/post-fs-data-kbd-idc.sh device/kbd_no_toucheventcheck.sh /data/local/tmp/

2. Copy files to /data/adb/ and set proper execution permissions:
adb shell su -c 'mkdir -p /data/adb/kbdauth /data/adb/service.d /data/adb/post-fs-data.d; cp /data/local/tmp/{midevauthd,libmidevauth.so,vendor.xiaomi.hardware.aidl.midevauth-V1-ndk_platform.so,tokenhelper,kbd_auth.sh,kbd_leds.sh} /data/adb/kbdauth/; cp /data/local/tmp/service.d-kbdauth.sh /data/adb/service.d/kbdauth.sh; cp /data/local/tmp/xiaomi_kbd_service.sh /data/adb/service.d/xiaomi_kbd_service.sh; cp /data/local/tmp/post-fs-data-kbd-idc.sh /data/adb/post-fs-data.d/kbd_idc.sh; cp /data/local/tmp/kbd_no_toucheventcheck.sh /data/adb/post-fs-data.d/kbd_no_toucheventcheck.sh; chmod 755 /data/adb/kbdauth/midevauthd /data/adb/kbdauth/tokenhelper /data/adb/kbdauth/kbd_auth.sh /data/adb/kbdauth/kbd_leds.sh /data/adb/service.d/kbdauth.sh /data/adb/service.d/xiaomi_kbd_service.sh /data/adb/post-fs-data.d/kbd_idc.sh /data/adb/post-fs-data.d/kbd_no_toucheventcheck.sh'

3. Reboot your tablet.

Verification:
Execute the following to confirm all daemons are running correctly:
su -c 'cat /data/adb/kbd/svc.log'     # Checks daemon status (vnd=1)
su -c 'cat /data/adb/kbd/auth.log'    # Expect: "-> STEP5 sent (real token)"
su -c 'cat /data/adb/kbd/leds.log'    # Checks EVDEV discovery for caps/keys

---

## Shortcuts & Configuration

Keyboard Backlight Shortcuts:
- Increase Brightness: Ctrl + Alt + 4
- Decrease / Off: Ctrl + Alt + 3
- Customization: Modify tunables at top of /data/adb/kbdauth/kbd_leds.sh (BL_STEP, BL_DEFAULT, BL_MAX). Set BL_NEED_ALT=0 to change triggers to plain Ctrl + 3/4.
Note: Shortcuts are observed rather than consumed, so select key combos your apps do not use.

---

## Architecture & Technical Overview

Executed Scripts & Locations:
/data/adb/post-fs-data.d/kbd_idc.sh -> Writes IDC early (fixes orientation/external flags)
/data/adb/post-fs-data.d/kbd_no_toucheventcheck.sh -> Kills broken telemetry daemon (fixes 100% CPU usage)
/data/adb/service.d/kbdauth.sh -> Starts midevauthd + kbd_auth.sh + kbd_leds.sh
/data/adb/service.d/xiaomi_kbd_service.sh -> Rebinds HID node on connect (enables input)
/data/adb/kbdauth/ -> Shared directory containing midevauthd, tokenhelper, kbd_auth.sh, and kbd_leds.sh

Component Breakdown & Technical Notes:
- midevauthd: Xiaomi proprietary HAL service (copied from stock ROM; uses vndbinder with LD_LIBRARY_PATH=/vendor/lib64:/system/lib64).
- tokenhelper: Small binder client compiling static arm64-v8a (NDK r26) that calls IMidevauthService.devauth_token_get.
- xiaomi_kbd_service.sh: Required fix. Prevents Android from leaving the HID device disabled.
- kbd_no_toucheventcheck.sh: Disables Xiaomi's /odm/bin/toucheventcheck (MiSight touch-telemetry service). On custom ROMs, deleting or rebinding input nodes causes this daemon's dump thread to degenerate into a user-space busy-loop, pegging a CPU core at 100%. Disabling it has zero functional impact on touch/keyboard input (logs to /data/adb/kbd/notec.log).
- Caps Lock LED: Android emits LED_CAPSL only on subsequent key events, and the hardware updates LEDs on activity. kbd_leds.sh intercepts KEY_CAPSLOCK presses directly and uses LED_CAPSL to sync state drift.
- IDC File Precedence: Android prefers /system/usr/idc over /data/system/devices/idc. Using manual root scripts avoids mounting a competing /system copy, ensuring /data overrides apply reliably.

---

## Building tokenhelper

GitHub Actions (.github/workflows/build.yml) cross-compiles it for arm64-v8a (NDK r26). Pushing a release tag starting with v* compiles and publishes binaries automatically. Binary is unchanged since v2 (same source).

---

## Uninstalling

Run the following commands in shell:
su -c 'rm /data/adb/service.d/kbdauth.sh /data/adb/service.d/xiaomi_kbd_service.sh /data/adb/post-fs-data.d/kbd_idc.sh /data/adb/post-fs-data.d/kbd_no_toucheventcheck.sh; rm -f /data/system/devices/idc/Vendor_15d9_Product_00a3.idc; pkill -f midevauthd; pkill -f kbd_auth.sh; pkill -f kbd_leds.sh; start toucheventcheck'
# Optional: su -c 'rm -rf /data/adb/kbdauth'

---

## Repository Information & License

- Legacy Branch: Version 2 (Auth-only logic) is archived on the v2-backup branch of the JohnnyFFM/xiaomi-keyboard-fix repository.
- License: MIT (see LICENSE). Reverse-engineered from system components for interoperability and repair purposes.
