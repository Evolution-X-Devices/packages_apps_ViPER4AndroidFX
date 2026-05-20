## ViPER4Android FX

This repository contains prebuilt ViPER4Android FX artifacts for integration into an AOSP-based ROM build tree.

It provides:

- `ViPER4AndroidFX`, a presigned prebuilt system app that overrides `AudioFX`.
- `libv4a_re`, prebuilt 32-bit and 64-bit vendor sound effect libraries installed under `vendor/lib*/soundfx`.

## Requirements

- A working AOSP or custom ROM source tree.
- This repository checked out at `packages/apps/ViPER4AndroidFX`.
- A product or device makefile where product packages can be inherited, usually `device.mk`.
- Device SELinux policy sources, including `audioserver.te`.
- The `libviperaidl` module available elsewhere in the build tree.

## Integration steps

### 1. Place the package in the source tree

Clone or copy this repository into the Android source tree at:

```text
packages/apps/ViPER4AndroidFX
```

The path matters because `config.mk` declares `BUILD_PATH := packages/apps/ViPER4AndroidFX` and adds that path to `PRODUCT_SOONG_NAMESPACES`.

### 2. Inherit the product configuration

Add the package configuration to your device or product makefile, usually `device.mk`:

```makefile
$(call inherit-product, packages/apps/ViPER4AndroidFX/config.mk)
```

This adds the following modules to the product:

- `ViPER4AndroidFX`
- `libv4a_re`

It also adds this directory to `PRODUCT_SOONG_NAMESPACES` and enables `RELAX_USES_LIBRARY_CHECK`.

### 3. Verify the external dependency

`libv4a_re` declares `libviperaidl` as a required module in `Android.bp`.

Before building, confirm that your ROM tree already provides `libviperaidl`. If it does not, add the missing module from the ROM/device source that normally provides ViPER audio support.

### 4. Add SELinux policy

Add the following rules to your device `audioserver.te` policy file:

```te
get_prop(audioserver, vendor_audio_prop) # If Google or MTK device skip line

allow audioserver unlabeled:file { read write open getattr };
allow hal_audio_default hal_audio_default:process { execmem };
```

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

## Notes

- This is not a Gradle/Android Studio project and does not build a new APK from source.
- `ViPER4AndroidFX` overrides `AudioFX` through `LOCAL_OVERRIDES_PACKAGES := AudioFX`.
- The native effect library is installed as a vendor module using Soong and is built for both 32-bit and 64-bit targets.
