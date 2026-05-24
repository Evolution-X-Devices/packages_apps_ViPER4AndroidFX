# ViPER4AndroidFX AOSP Integration Guide

This repository is an Android platform/ROM integration module for ViPER4AndroidFX. It is not a standalone Android Studio or Gradle project. It packages a prebuilt, presigned APK and prebuilt native audio effect libraries into an AOSP-style build tree.

## What this module provides

The repository installs the following components:

- `ViPER4AndroidFX.apk` as a presigned system app
- `libv4a_re.so` as a vendor audio effect library
- 32-bit and 64-bit versions of the native sound effect library

Important files:

- `Android.bp`: Soong definition for the prebuilt native library
- `Android.mk`: includes subdirectory makefiles
- `config.mk`: product makefile fragment used by your device/product makefile
- `system/app/ViPER4AndroidFX/ViPER4AndroidFX.apk`: prebuilt application
- `vendor/lib/soundfx/libv4a_re.so`: 32-bit audio effect library
- `vendor/lib64/soundfx/libv4a_re.so`: 64-bit audio effect library

## Important note about the inherit-product line

This line is not a shell command:

```makefile
$(call inherit-product, packages/apps/ViPER4AndroidFX/config.mk)
```

Do not run it in PowerShell, Bash, CMD, or any terminal. It must be added to an Android product makefile such as:

- `device/<vendor>/<device>/device.mk`
- `device/<vendor>/<device>/<product>.mk`
- another product makefile inherited by your target

If you run it directly in PowerShell, you will get an error similar to:

```text
call : The term 'call' is not recognized as the name of a cmdlet...
```

That error is expected because `$(call inherit-product, ...)` is Android Make/Kati syntax, not a terminal command.

## Requirements

You need a full Android ROM build tree, such as AOSP, LineageOS, Evolution X, or another AOSP-derived ROM tree.

You also need:

- A configured Android build environment
- A valid device tree and product target
- Soong/Kati/Make/Ninja from the Android build system
- The `libviperaidl` module available somewhere in the ROM tree, or an intentional adjustment to remove that requirement
- Device SELinux policy access
- Device audio effect configuration access

On Windows, Android platform builds are normally done through WSL2, a Linux VM, a container, or a native Linux machine. The Android platform build system is not intended to run directly in PowerShell.

## Step 1: Place the repository in the Android source tree

Clone or copy this repository into:

```text
packages/apps/ViPER4AndroidFX
```

Example from the root of the Android source tree:

```bash
git clone https://github.com/Evolution-X-Devices/packages_apps_ViPER4AndroidFX.git packages/apps/ViPER4AndroidFX
```

The included `config.mk` assumes this path:

```makefile
BUILD_PATH := packages/apps/ViPER4AndroidFX
```

If you place the repository somewhere else, update `config.mk` accordingly.

## Step 2: Include the module from your product makefile

Edit your device or product makefile. Common locations include:

```text
device/<vendor>/<device>/device.mk
device/<vendor>/<device>/<product>.mk
```

Add:

```makefile
$(call inherit-product, packages/apps/ViPER4AndroidFX/config.mk)
```

The included `config.mk` adds the Soong namespace and product packages:

```makefile
BUILD_PATH := packages/apps/ViPER4AndroidFX

PRODUCT_SOONG_NAMESPACES += \
    packages/apps/ViPER4AndroidFX

PRODUCT_PACKAGES += \
    ViPER4AndroidFX \
    libv4a_re

RELAX_USES_LIBRARY_CHECK := true
```

This causes the build to include:

- `ViPER4AndroidFX`
- `libv4a_re`

## Step 3: Confirm the prebuilt files exist

Confirm the repository contains:

```text
packages/apps/ViPER4AndroidFX/system/app/ViPER4AndroidFX/ViPER4AndroidFX.apk
packages/apps/ViPER4AndroidFX/vendor/lib/soundfx/libv4a_re.so
packages/apps/ViPER4AndroidFX/vendor/lib64/soundfx/libv4a_re.so
```

The APK is installed through `system/app/Android.mk` as a prebuilt app:

```makefile
LOCAL_MODULE := ViPER4AndroidFX
LOCAL_MODULE_TAGS := optional
LOCAL_MODULE_CLASS := APPS
LOCAL_CERTIFICATE := PRESIGNED
LOCAL_SRC_FILES := ViPER4AndroidFX/ViPER4AndroidFX.apk
LOCAL_OVERRIDES_PACKAGES := AudioFX
include $(BUILD_PREBUILT)
```

The `LOCAL_OVERRIDES_PACKAGES := AudioFX` line means the module is intended to replace the stock `AudioFX` package when included.

## Step 4: Ensure `libviperaidl` exists

The native library module in `Android.bp` declares:

```bp
required: [
    "libviperaidl",
],
```

Your build tree must provide a module named `libviperaidl`.

If the build fails with a missing module error for `libviperaidl`, resolve it by doing one of the following:

- Add the ROM/device source package that provides `libviperaidl`
- Add the matching prebuilt module if your ROM expects one
- Remove or adjust the `required` dependency only if you know your target does not need it

Do not remove the dependency blindly. It may be required for the app, driver service, or ROM-specific ViPER integration.

## Step 5: Add SELinux policy rules

The repository README requires SELinux additions in your device policy.

Common file locations:

```text
device/<vendor>/<device>/sepolicy/vendor/audioserver.te
device/<vendor>/<device>/sepolicy/audioserver.te
device/<vendor>/<device>/sepolicy/private/audioserver.te
```

Add:

```te
get_prop(audioserver, vendor_audio_prop)

allow audioserver unlabeled:file { read write open getattr };
allow hal_audio_default hal_audio_default:process { execmem };
```

For Google or MTK devices, the README says to skip:

```te
get_prop(audioserver, vendor_audio_prop)
```

### SELinux rule notes

`get_prop(audioserver, vendor_audio_prop)` allows `audioserver` to read vendor audio properties. Some platforms already provide this permission, and some platforms do not use `vendor_audio_prop`.

The `audioserver unlabeled:file` rule is broad. If possible, prefer a more specific rule based on the actual AVC denial from your device. Use the broad rule only if it matches your tree's known integration requirement.

The `execmem` permission is required because the native audio effect may request executable memory at runtime:

```te
allow hal_audio_default hal_audio_default:process { execmem };
```

Some device trees may use this equivalent style:

```te
allow hal_audio_default self:process execmem;
```

Follow the style already used by your device policy.

### Debugging SELinux denials

After flashing and booting, check for denials:

```bash
adb logcat -b all | grep -i avc
```

or:

```bash
adb shell dmesg | grep -i avc
```

Look for denials involving:

- `audioserver`
- `hal_audio_default`
- `soundfx`
- `libv4a_re.so`
- `vendor_audio_prop`
- `unlabeled`

If the policy causes `neverallow` failures, do not force it without understanding the violation. Inspect the denial and use a narrower policy rule if possible.

## Step 6: Add or update audio effect configuration

This repository installs the ViPER native effect library, but it does not include an `audio_effects.xml` or `audio_effects.conf` file. Many Android builds require the effect to be registered in the device audio effects configuration.

Common audio effect config locations:

```text
device/<vendor>/<device>/audio_effects.xml
device/<vendor>/<device>/audio/audio_effects.xml
device/<vendor>/<device>/configs/audio_effects.xml
device/<vendor>/<device>/audio_effects.conf
vendor/etc/audio_effects.xml
vendor/etc/audio_effects.conf
odm/etc/audio_effects.xml
odm/etc/audio_effects.conf
```

Modern Android versions generally use XML. Older trees may still use `.conf`.

### XML audio effect configuration

If your device uses `audio_effects.xml`, add this library entry under `<libraries>`:

```xml
<library name="v4a_re" path="libv4a_re.so"/>
```

Add this effect entry under `<effects>`:

```xml
<effect name="v4a_standard_fx" library="v4a_re" uuid="41d3c987-e6cf-11e3-a88a-11aba5d5c51b"/>
```

Example minimal XML:

```xml
<audio_effects_conf version="2.0" xmlns="http://schemas.android.com/audio/audio_effects_conf/v2_0">
    <libraries>
        <library name="v4a_re" path="libv4a_re.so"/>
    </libraries>

    <effects>
        <effect name="v4a_standard_fx" library="v4a_re" uuid="41d3c987-e6cf-11e3-a88a-11aba5d5c51b"/>
    </effects>
</audio_effects_conf>
```

If your existing file already has `<libraries>` and `<effects>` sections, add only the ViPER entries to the existing sections. Do not replace the whole file unless you are intentionally creating a new complete config.

### Legacy `audio_effects.conf` configuration

If your device uses the legacy `.conf` format, add:

```conf
libraries {
  v4a_re {
    path /vendor/lib/soundfx/libv4a_re.so
  }
}

effects {
  v4a_standard_fx {
    library v4a_re
    uuid 41d3c987-e6cf-11e3-a88a-11aba5d5c51b
  }
}
```

The repository installs both:

```text
/vendor/lib/soundfx/libv4a_re.so
/vendor/lib64/soundfx/libv4a_re.so
```

The audio effect loader should select the correct library path for the process architecture.

### Copying a new audio effects config

If your device tree already copies an audio effects config, modify the existing source file.

If your device tree does not already provide one, add one such as:

```text
device/<vendor>/<device>/audio/audio_effects.xml
```

Then add a copy rule to your device makefile:

```makefile
PRODUCT_COPY_FILES += \
    device/<vendor>/<device>/audio/audio_effects.xml:$(TARGET_COPY_OUT_VENDOR)/etc/audio_effects.xml
```

Do not create duplicate `PRODUCT_COPY_FILES` entries for the same destination. If another file already installs to `$(TARGET_COPY_OUT_VENDOR)/etc/audio_effects.xml`, edit that existing source file instead.

### Optional post-processing configuration

Some ROMs attach effects through post-processing configuration. If ViPER does not attach automatically and your existing audio config uses post-processing blocks, you may need an entry like:

```xml
<postprocess>
    <stream type="music">
        <apply effect="v4a_standard_fx"/>
    </stream>
</postprocess>
```

Only add this if your ROM/device audio stack expects stream effects to be explicitly applied. Many effects are attached dynamically by the app through Android's audio effects API.

## Step 7: Build the module

From the root of your Android source tree:

```bash
source build/envsetup.sh
lunch <target>
m ViPER4AndroidFX libv4a_re
```

Or build the full ROM using your ROM's standard command, for example:

```bash
m
```

Some ROMs use wrappers such as:

```bash
mka bacon
```

Use the command appropriate for your ROM.

## Step 8: Verify build output

After a successful build, check the output directory:

```bash
ls out/target/product/<device>/system/app/ViPER4AndroidFX/
ls out/target/product/<device>/vendor/lib/soundfx/
ls out/target/product/<device>/vendor/lib64/soundfx/
```

Expected output artifacts:

```text
out/target/product/<device>/system/app/ViPER4AndroidFX/ViPER4AndroidFX.apk
out/target/product/<device>/vendor/lib/soundfx/libv4a_re.so
out/target/product/<device>/vendor/lib64/soundfx/libv4a_re.so
```

## Step 9: Flash and verify on device

After flashing and booting the device, confirm the files exist:

```bash
adb shell ls -l /system/app/ViPER4AndroidFX/
adb shell ls -l /vendor/lib/soundfx/libv4a_re.so
adb shell ls -l /vendor/lib64/soundfx/libv4a_re.so
```

Verify that the effect is registered:

```bash
adb shell grep -R "v4a\|ViPER\|41d3c987" /vendor/etc /odm/etc /system/etc 2>/dev/null
```

Check audio service state:

```bash
adb shell dumpsys media.audio_flinger | grep -iE "v4a|viper|41d3c987"
```

Check runtime logs:

```bash
adb logcat | grep -iE "viper|v4a|audioeffect|soundfx|audioserver"
```

Check SELinux denials:

```bash
adb logcat -b all | grep -i avc
```

## Troubleshooting

### `call : The term 'call' is not recognized`

You ran this makefile line in a shell:

```makefile
$(call inherit-product, packages/apps/ViPER4AndroidFX/config.mk)
```

Add it to your device/product makefile instead. It is not a terminal command.

### Missing `libviperaidl`

If the build fails because `libviperaidl` is missing, your ROM tree does not provide a module required by this repository. Add the matching module source/prebuilt or adjust the dependency only if you know your integration does not require it.

### APK is not installed

Check that:

- `$(call inherit-product, packages/apps/ViPER4AndroidFX/config.mk)` is in a makefile used by your selected product
- `ViPER4AndroidFX` appears in `PRODUCT_PACKAGES`
- The source APK exists at `system/app/ViPER4AndroidFX/ViPER4AndroidFX.apk`

### Native library is not installed

Check that:

- `libv4a_re` appears in `PRODUCT_PACKAGES`
- `PRODUCT_SOONG_NAMESPACES` includes `packages/apps/ViPER4AndroidFX`
- Both prebuilt `.so` files exist in the repository

### Effect does not load

Check that:

- `audio_effects.xml` or `audio_effects.conf` registers `v4a_re`
- The UUID is present: `41d3c987-e6cf-11e3-a88a-11aba5d5c51b`
- The libraries exist under `/vendor/lib/soundfx` and `/vendor/lib64/soundfx`
- There are no SELinux denials blocking `audioserver` or `hal_audio_default`

## Full integration checklist

1. Place the repo at `packages/apps/ViPER4AndroidFX`.
2. Add `$(call inherit-product, packages/apps/ViPER4AndroidFX/config.mk)` to your device/product makefile.
3. Confirm `ViPER4AndroidFX` and `libv4a_re` are included in `PRODUCT_PACKAGES`.
4. Ensure `libviperaidl` exists in the ROM tree.
5. Add the required SELinux policy rules to `audioserver.te`.
6. Register `libv4a_re` in `audio_effects.xml` or `audio_effects.conf` if your tree does not already do so.
7. Build the module or full ROM.
8. Verify the APK and both native libraries are present in the build output.
9. Flash the build.
10. Verify on-device files, audio effect registration, audio service logs, and SELinux denials.
