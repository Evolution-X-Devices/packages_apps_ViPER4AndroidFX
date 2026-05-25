## ViPER4Android FX

This repository contains prebuilt ViPER4Android FX artifacts for integration into an AOSP-based or custom Android ROM build tree.

It provides:

- `ViPER4AndroidFX`, a presigned prebuilt system app that overrides `AudioFX`.
- `libv4a_re`, prebuilt 32-bit and 64-bit vendor sound effect libraries installed under `vendor/lib*/soundfx`.

## Requirements

- A working AOSP or custom ROM source tree.
- This repository checked out at `packages/apps/ViPER4AndroidFX`.
- A product or device makefile where product packages can be inherited, usually `device.mk`.
- Device SELinux policy sources, usually under a device `sepolicy` directory.
- The `libviperaidl` module available elsewhere in the build tree.

## Integration steps

### 1. Place the package in the source tree

Clone or copy this repository into the Android source tree at:

```text
packages/apps/ViPER4AndroidFX
```

The path matters because `config.mk` declares `BUILD_PATH := packages/apps/ViPER4AndroidFX` and adds that path to `PRODUCT_SOONG_NAMESPACES`.

### 2. Inherit the product configuration

Add the package configuration to your device or product makefile, usually `device/<vendor>/<device>/device.mk`:

```makefile
$(call inherit-product, packages/apps/ViPER4AndroidFX/config.mk)
```

The included `config.mk` adds:

```makefile
PRODUCT_SOONG_NAMESPACES += \
    packages/apps/ViPER4AndroidFX

PRODUCT_PACKAGES += \
    ViPER4AndroidFX \
    libv4a_re

RELAX_USES_LIBRARY_CHECK := true
```

### 3. Verify build dependencies and prebuilts

The ROM tree must be able to build or provide these modules:

- `ViPER4AndroidFX`
- `libv4a_re`
- `libviperaidl`

`ViPER4AndroidFX` is a presigned prebuilt APK from:

```text
system/app/ViPER4AndroidFX/ViPER4AndroidFX.apk
```

It overrides the stock `AudioFX` package through `LOCAL_OVERRIDES_PACKAGES := AudioFX` in `system/app/Android.mk`.

`libv4a_re` is declared as a prebuilt shared vendor library and depends on standard system shared libraries:

- `liblog`
- `libm`
- `libdl`
- `libc`

The prebuilt audio effect libraries must exist at:

```text
vendor/lib/soundfx/libv4a_re.so
vendor/lib64/soundfx/libv4a_re.so
```

The module installs them to the vendor `soundfx` path through:

```bp
vendor: true
relative_install_path: "soundfx"
compile_multilib: "both"
```

### 4. Add SELinux policy

Do not run the SELinux rules in a host shell. They are Android SELinux policy statements and must be added to the device sepolicy used by the ROM build.

#### Option 1: Copy the provided policy snippet

Copy the provided policy file into your device sepolicy tree:

```text
packages/apps/ViPER4AndroidFX/sepolicy/audioserver_viper4android.te
```

For example:

```text
device/<vendor>/<device>/sepolicy/vendor/audioserver_viper4android.te
```

Then ensure the device sepolicy directory is included by the device tree, commonly in `BoardConfig.mk` or `BoardConfigCommon.mk`:

```makefile
BOARD_VENDOR_SEPOLICY_DIRS += \
    device/<vendor>/<device>/sepolicy/vendor
```

Use `BOARD_SEPOLICY_DIRS` instead if your ROM/device tree still uses a non-vendor sepolicy layout.

#### Option 2: Add the rules to an existing `audioserver.te`

If your device tree already has an `audioserver.te`, add:

```te
get_prop(audioserver, vendor_audio_prop)

allow audioserver unlabeled:file {
    getattr
    open
    read
    write
};

allow hal_audio_default hal_audio_default:process execmem;
```

For Google or MTK devices, skip `get_prop(audioserver, vendor_audio_prop)` if the device tree already grants the required vendor audio property access.

Depending on your device tree and Android version, these rules may need to be adapted to satisfy existing neverallow rules or vendor policy constraints.

### 5. Build the modules

From the root of the Android source tree, initialize the build environment and select your target:

```sh
source build/envsetup.sh
lunch <target>
```

Then build the package modules directly:

```sh
mka ViPER4AndroidFX libv4a_re
```

Alternatively, build your normal ROM target after inheriting `config.mk`.

### 6. Verify install locations

After a successful build, verify that the product output includes:

- `ViPER4AndroidFX.apk` as a system app.
- `libv4a_re.so` under the vendor sound effects library path for both supported ABIs.

The prebuilt APK is signed with its existing certificate, so the module uses `LOCAL_CERTIFICATE := PRESIGNED`.

## Integration checklist

1. Put this module at `packages/apps/ViPER4AndroidFX`.
2. Add the `inherit-product` line to `device.mk`.
3. Confirm `PRODUCT_PACKAGES` includes `ViPER4AndroidFX` and `libv4a_re` through `config.mk`.
4. Confirm `system/app/ViPER4AndroidFX/ViPER4AndroidFX.apk` exists.
5. Confirm `vendor/lib/soundfx/libv4a_re.so` and `vendor/lib64/soundfx/libv4a_re.so` exist.
6. Confirm `libviperaidl` is available in the ROM tree.
7. Add the SELinux rules through the provided policy file or an existing `audioserver.te`.
8. Ensure the device sepolicy directory is referenced by `BOARD_VENDOR_SEPOLICY_DIRS` or `BOARD_SEPOLICY_DIRS`.
9. Build the ROM and verify that `libv4a_re.so` is installed under the vendor `soundfx` directory.

## Notes

- This is not a Gradle/Android Studio project and does not build a new APK from source.
- `ViPER4AndroidFX` overrides `AudioFX` through `LOCAL_OVERRIDES_PACKAGES := AudioFX`.
- The native effect library is installed as a vendor module using Soong and is built for both 32-bit and 64-bit targets.
