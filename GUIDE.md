# SM-A135F (Exynos850) — Kernel + KernelSU (RKSU) Build/Flash Guide

This guide matches your current workspace layout:

```
/home/kucingsakti/dev/SM-A135F
├── build/               # outputs + repack workdir
├── raw/                 # prepared images: boot.img dtbo.img vbmeta.img
├── src/Kernel/          # kernel tree
└── tools/               # toolchains + android tools (mkbootimg/avb) + patches
```

It covers:
- Toolchain setup (clang-r383902 + GCC 4.9) under `tools/`
- KernelSU integration using **rsuntk/KernelSU** (RKSU)
- Manual hook patch application using `rksuorg/kernel_patches`
- Build + repack `boot_rksu.img`
- Backup current boot from device (Magisk)
- Flash with Heimdall (split + resume) ✅

---

## 0) Variables (run once per shell)

```bash
set -euo pipefail

export WORK=/home/kucingsakti/dev/SM-A135F
export TOOLS=$WORK/tools
export KERNEL=$WORK/src/Kernel
export RAW=$WORK/raw
export OUT=$WORK/build

mkdir -p "$TOOLS" "$TOOLS/_src" "$TOOLS/toolchains" "$OUT"
```

---

## 1) Host dependencies

```bash
sudo apt update
sudo apt install -y git curl ca-certificates \
  flex bison bc build-essential make \
  libssl-dev libelf-dev dwarves rsync unzip zstd python3 lz4 \
  heimdall-flash
```

---

## 2) Toolchains under `tools/toolchains`

### 2.1 Clang r383902 (AOSP) — stable install (sparse checkout)

```bash
cd "$TOOLS/_src"
rm -rf prebuilts_clang_linux-x86

git clone --depth=1 --filter=blob:none --sparse \
  --branch android-11.0.0_r3 \
  https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86 \
  prebuilts_clang_linux-x86

cd prebuilts_clang_linux-x86
git sparse-checkout set clang-r383902

rm -rf "$TOOLS/toolchains/clang-r383902"
cp -a clang-r383902 "$TOOLS/toolchains/clang-r383902"

"$TOOLS/toolchains/clang-r383902/bin/clang" --version | head -n 2
```

### 2.2 GCC 4.9 (CodeLinaro) — checkout tag used by Samsung trees

```bash
cd "$TOOLS/_src"
rm -rf aarch64-linux-android-4.9

git clone https://git.codelinaro.org/clo/la/platform/prebuilts/gcc/linux-x86/aarch64/aarch64-linux-android-4.9.git \
  aarch64-linux-android-4.9

cd aarch64-linux-android-4.9
git checkout webview-m40_r4

rm -rf "$TOOLS/toolchains/aarch64-linux-android-4.9"
cp -a . "$TOOLS/toolchains/aarch64-linux-android-4.9"

"$TOOLS/toolchains/aarch64-linux-android-4.9/bin/aarch64-linux-android-gcc" --version | head -n 1
```

---

## 3) Android tools under `tools/android-tools`

### 3.1 mkbootimg

```bash
mkdir -p "$TOOLS/android-tools"
cd "$TOOLS/android-tools"
[ -d mkbootimg ] || git clone https://android.googlesource.com/platform/system/tools/mkbootimg
```

### 3.2 avbtool (optional)

```bash
cd "$TOOLS/android-tools"
[ -d avb ] || git clone https://android.googlesource.com/platform/external/avb avb
python3 "$TOOLS/android-tools/avb/avbtool.py" --help >/dev/null && echo "avbtool OK"
```

---

## 4) Kernel tree quirks (Samsung vendor tree fixes)

### 4.1 Samsung tree hardcodes `./toolchain/clang/...` path

Even if you export `CC=...`, parts of the build call:

```
./toolchain/clang/host/linux-x86/clang-r383902/bin/clang
```

Fix by symlinking kernel-tree path to your `tools/` clang:

```bash
cd "$KERNEL"
mkdir -p toolchain/clang/host/linux-x86
ln -snf "$TOOLS/toolchains/clang-r383902" toolchain/clang/host/linux-x86/clang-r383902

"$KERNEL/toolchain/clang/host/linux-x86/clang-r383902/bin/clang" --version | head -n 2
```

### 4.2 Missing Himax Makefile during clean/mrproper

If you hit errors like:
`drivers/input/touchscreen/himax/himax_83xxx_i2c/Makefile: No such file`

Create a placeholder:

```bash
cd "$KERNEL"
mkdir -p drivers/input/touchscreen/himax/himax_83xxx_i2c
: > drivers/input/touchscreen/himax/himax_83xxx_i2c/Makefile
```

---

## 5) Integrate KernelSU (rsuntk/KernelSU) + Manual Hook

### 5.1 Inject RKSU into the kernel tree

```bash
cd "$KERNEL"
curl -LSs "https://raw.githubusercontent.com/rsuntk/KernelSU/main/kernel/setup.sh" | bash -s v3.0.0-30-legacy
```

> Notes:
> - `v3.0.0-30-legacy` is a common “legacy” target for non-GKI kernels like 4.19.
> - The script typically creates `KernelSU/` and hooks it into `drivers/` / Kconfig.

### 5.2 Enable Manual Hook in defconfig

Edit:

```bash
nano "$KERNEL/arch/arm64/configs/exynos850-a13xx_defconfig"
```

Ensure at least:

```text
CONFIG_KSU=y
CONFIG_KSU_MANUAL_HOOK=y
# CONFIG_KPROBES is not set
# CONFIG_KPROBE_EVENTS is not set
```

### 5.3 Apply manual hook patches (required for non-GKI when hook points aren’t present)

You already have:

```
$TOOLS/kernel_patches/manual_hook/kernel-4.19_5.4.patch
```

Dry-run then apply:

```bash
cd "$KERNEL"

patch -p1 --dry-run < "$TOOLS/kernel_patches/manual_hook/kernel-4.19_5.4.patch"
patch -p1 < "$TOOLS/kernel_patches/manual_hook/kernel-4.19_5.4.patch"
```

Quick verification:

```bash
cd "$KERNEL"
grep -R "CONFIG_KSU_MANUAL_HOOK" -n fs | head
grep -R "ksu_" -n fs | head
```

---

## 6) Build kernel

```bash
cd "$KERNEL"

export PLATFORM_VERSION=12
export ANDROID_MAJOR_VERSION=s
export ARCH=arm64
export SUBARCH=arm64

export CROSS_COMPILE="$TOOLS/toolchains/aarch64-linux-android-4.9/bin/aarch64-linux-android-"
export CC="$TOOLS/toolchains/clang-r383902/bin/clang"
export CLANG_TRIPLE="$TOOLS/toolchains/clang-r383902/bin/aarch64-linux-gnu-"

make mrproper
make exynos850-a13xx_defconfig

# Optional sanity check:
grep -E "CONFIG_KSU|CONFIG_KSU_MANUAL_HOOK|CONFIG_KPROBES|CONFIG_KPROBE_EVENTS" .config || true

make -j"$(nproc)" 2>&1 | tee "$OUT/build_rksu.log"

ls -lah arch/arm64/boot/Image
```

Output kernel image:
- `src/Kernel/arch/arm64/boot/Image`

---

## 7) Repack `boot_rksu.img` (no AP extract; use `raw/boot.img`)

You already prepared:

```
$RAW/boot.img
$RAW/dtbo.img
$RAW/vbmeta.img
```

Repack:

```bash
mkdir -p "$OUT/work_boot"
cd "$OUT/work_boot"

mkdir -p boot_unpack
python3 "$TOOLS/android-tools/mkbootimg/unpack_bootimg.py" \
  --boot_img "$RAW/boot.img" \
  --out boot_unpack

cp -f "$KERNEL/arch/arm64/boot/Image" boot_unpack/kernel

python3 "$TOOLS/android-tools/mkbootimg/mkbootimg.py" \
  --header_version 2 \
  --os_version 14.0.0 \
  --os_patch_level 2025-11 \
  --pagesize 0x00000800 \
  --base 0x00000000 \
  --kernel_offset 0x10008000 \
  --ramdisk_offset 0x11000000 \
  --second_offset 0x00000000 \
  --tags_offset 0x10000100 \
  --dtb_offset 0x0000000010000000 \
  --board SRPUK09B013 \
  --cmdline "androidboot.hardware=exynos850 androidboot.selinux=enforce loop.max_part=7" \
  --kernel boot_unpack/kernel \
  --ramdisk boot_unpack/ramdisk \
  --dtb boot_unpack/dtb \
  --output boot_rksu.img

file boot_rksu.img
ls -lah boot_rksu.img
```

Result:
- `build/work_boot/boot_rksu.img`

> If you see `fatal: not a git repository` spam: it’s harmless and comes from mkbootimg trying to query git metadata.

---

## 8) Backup current boot from device (Magisk)

Your device mapping:
`/dev/block/by-name/boot -> /dev/block/mmcblk0p18` (single-slot)

Using Magisk full-root:

```bash
adb shell 'su 0 -c "dd if=/dev/block/by-name/boot of=/sdcard/boot_backup.img bs=4096; sync"'
adb pull /sdcard/boot_backup.img ./boot_backup.img
```

Verify:

```bash
file ./boot_backup.img
sha256sum ./boot_backup.img
```

Cleanup (optional):

```bash
adb shell 'rm -f /sdcard/boot_backup.img'
```

---

## 9) Flash with Heimdall (split flash — safest flow)

### 9.1 Detect device (Download Mode)

```bash
sudo heimdall detect
```

### 9.2 Flash BOOT only (no reboot)

```bash
sudo heimdall flash --BOOT "$OUT/work_boot/boot_rksu.img" --no-reboot
```

### 9.3 Flash DTBO + VBMETA (resume session)

```bash
sudo heimdall flash --DTBO "$RAW/dtbo.img" --VBMETA "$RAW/vbmeta.img" --resume
```

### 9.4 Reboot

From Download Mode:
- Hold **Vol Down + Power** for ~10–15 seconds.

---

## 10) Post-flash checks

```bash
adb wait-for-device
adb shell getprop ro.boot.verifiedbootstate
adb shell su -c 'uname -r'
```

---

## Troubleshooting

### `dd: unknown status 'progress'`
Android `dd` may not support `status=progress`. Remove it.

### `zsh: no matches found: /sdcard/boot_backup_*.img`
Quote the wildcard:
```bash
adb pull '/sdcard/boot_backup_*.img' .
```

### `dd ... Permission denied` while using Magisk
Use full root:
```bash
adb shell 'su 0 -c "dd if=/dev/block/by-name/boot of=/sdcard/boot_backup.img bs=4096; sync"'
```

### Samsung tree ignores exported `CC` and calls `./toolchain/.../clang`
Create the symlink in **Section 4.1**.

### Clean fails on Himax Makefile missing
Create placeholder in **Section 4.2**.

---

## Files you should keep

- `raw/boot.img`, `raw/dtbo.img`, `raw/vbmeta.img` (stock baseline)
- `build/work_boot/boot_rksu.img` (your flashed image)
- `boot_backup_*.img` (device-current backup)
- `build/build_rksu.log` (build log for debugging)

---
