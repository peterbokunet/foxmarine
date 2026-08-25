# FoxMarine APK — Migrating a Play Store App Between Devices

## The Problem

When you install an app from the Google Play Store, Google doesn't deliver a single APK. Instead, it delivers an **App Bundle** — the app split into pieces optimized for your specific device:

- **base.apk** — the application code and core resources
- **split_config.xxxhdpi.apk** — display density resources matched to your screen
- **split_config.en.apk** — language resources (English, Spanish, etc.)

The base APK contains a flag (`requiredSplitTypes="base__density"`) that tells Android: *"I won't run without my companion density split."* If you extract just the base APK and try to install it on another phone, Android sees the flag, can't find the companion splits, and refuses to run the app.

This is what happens when you use a backup tool or APK extractor — you get the base APK, but not the splits. The app installs but immediately crashes or won't launch on the new device.

## The Solution

Pull **all** split APKs from the source device and install them together on the target device using `adb install-multiple`. This delivers the complete set Android expects — no APK modification needed.

## App Details

| Field | Value |
|-------|-------|
| Package | `com.foxmarine.android.bluetoothlegatt` |
| Version | 1.34 (versionCode 34) |
| Min SDK | 21 (Android 5.0) |
| Target SDK | 35 (Android 15) |
| Architecture | arm64-v8a |
| Function | Bluetooth LE GATT communication with FoxMarine hardware |

## Prerequisites

Install these tools via Homebrew:

```bash
brew install android-platform-tools
```

This provides `adb` (Android Debug Bridge). You also need:

- USB debugging enabled on both devices (Settings → Developer Options → USB Debugging)
- Both devices connected via USB
- The app still installed on the source device

## Steps

### 1. Connect the Source Device and List APK Splits

```bash
adb devices -l

adb shell pm path com.foxmarine.android.bluetoothlegatt
```

This outputs something like:

```
package:/data/app/~~<hash>==/com.foxmarine.android.bluetoothlegatt-<hash>==/base.apk
package:/data/app/~~<hash>==/com.foxmarine.android.bluetoothlegatt-<hash>==/split_config.en.apk
package:/data/app/~~<hash>==/com.foxmarine.android.bluetoothlegatt-<hash>==/split_config.es.apk
package:/data/app/~~<hash>==/com.foxmarine.android.bluetoothlegatt-<hash>==/split_config.xxxhdpi.apk
```

### 2. Pull All Splits

```bash
mkdir -p splits

# Pull each path listed above
adb pull /data/app/~~<hash>==/com.foxmarine.android.bluetoothlegatt-<hash>==/base.apk splits/
adb pull /data/app/~~<hash>==/com.foxmarine.android.bluetoothlegatt-<hash>==/split_config.en.apk splits/
adb pull /data/app/~~<hash>==/com.foxmarine.android.bluetoothlegatt-<hash>==/split_config.es.apk splits/
adb pull /data/app/~~<hash>==/com.foxmarine.android.bluetoothlegatt-<hash>==/split_config.xxxhdpi.apk splits/
```

### 3. Connect the Target Device and Install

If both devices are connected simultaneously, use `-s <serial>` to target the right one:

```bash
adb devices -l

adb -s <TARGET_SERIAL> install-multiple \
  splits/base.apk \
  splits/split_config.en.apk \
  splits/split_config.es.apk \
  splits/split_config.xxxhdpi.apk
```

If only the target device is connected:

```bash
adb install-multiple \
  splits/base.apk \
  splits/split_config.en.apk \
  splits/split_config.es.apk \
  splits/split_config.xxxhdpi.apk
```

### 4. Launch

```bash
adb shell am start -n com.foxmarine.android.bluetoothlegatt/.DeviceScanActivity
```

## Split APKs in This Repo

| File | Size | Purpose |
|------|------|---------|
| `splits/base.apk` | 8.7 MB | Application code and core resources |
| `splits/split_config.en.apk` | 36 KB | English language resources |
| `splits/split_config.es.apk` | 24 KB | Spanish language resources |
| `splits/split_config.xxxhdpi.apk` | 86 KB | High-density display resources |

## What Didn't Work

- **Installing just `base.apk`** — Android refuses to run it due to the `requiredSplitTypes` flag
- **Rebuilding as a single APK** (via apktool) — stripping the split requirements and merging resources failed due to unresolved resource references during recompilation
- **Decompiling with JD-GUI** — incompatible with modern JDK (uses deprecated `com.apple.eawt` APIs)

## Notes

- `adb install-multiple` is the same mechanism the Play Store uses to install App Bundles
- The splits from one device work on another as long as the target meets the minSdk requirement
- The xxxhdpi density split works for any high-density screen (both Pixel 6 Pro and Pixel 10 Pro qualify)
- If the app is ever removed from the Play Store, these splits serve as the installable backup
