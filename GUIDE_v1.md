# GUIDE — Build & Flash Kernel **v1** (SM‑A135F / Exynos850) — **Tanpa KernelSU**
> Target: Debian 13 host, build kernel dari source kamu di `/mnt/DATA/SM-A135F/src/Kernel`  
> Output: boot image custom yang bisa di-flash via Heimdall  
> Mode: **Stock base** (no root) dan **Magisk base** (root tetap)

---

## 0) Folder & Prasyarat
### 0.1 Struktur folder yang dipakai (contoh)
- Kernel tree: `/mnt/DATA/SM-A135F/src/Kernel`
- Firmware AP: `/mnt/DATA/SM-A135F/raw/AP_*.tar.md5`
- Workspace boot: `/mnt/DATA/SM-A135F/raw/work_ap`

### 0.2 Paket yang dibutuhkan (Debian 13)
```bash
sudo apt update
sudo apt install -y \
  git bc bison flex make gcc g++ \
  clang lld llvm \
  libssl-dev libelf-dev dwarves python3 perl \
  lz4 unzip zip \
  android-tools-mkbootimg \
  heimdall-flash
```

### 0.3 Tools AOSP mkbootimg (script) yang dipakai di GUIDE ini
Pastikan kamu punya:
- `~/android-tools/mkbootimg/unpack_bootimg.py`
- `~/android-tools/mkbootimg/mkbootimg.py`

Cek cepat:
```bash
python3 ~/android-tools/mkbootimg/unpack_bootimg.py --help | head
python3 ~/android-tools/mkbootimg/mkbootimg.py --help | head
```

---

## 1) Build Kernel (v1, tanpa KernelSU)
### 1.1 Masuk kernel tree + export env
```bash
cd /mnt/DATA/SM-A135F/src/Kernel

export PLATFORM_VERSION=12
export ANDROID_MAJOR_VERSION=s
export ARCH=arm64
export SUBARCH=arm64

export CROSS_COMPILE=toolchain/gcc/linux-x86/aarch64/aarch64-linux-android-4.9/bin/aarch64-linux-android-
export CC=toolchain/clang/host/linux-x86/clang-r383902/bin/clang

export KBUILD_BUILD_USER=kucingsakti
export KBUILD_BUILD_HOST=debian13
```

> (Opsional) Kalau kamu mau custom string versi:
- edit `arch/arm64/configs/exynos850-a13xx_defconfig` lalu tambahkan:
  - `CONFIG_LOCALVERSION="-apin-v1"`

### 1.2 Clean + defconfig + build
```bash
make clean
make mrproper
make exynos850-a13xx_defconfig
make -j"$(nproc)" 2>&1 | tee build_stock_v1.log
```

### 1.3 Pastikan output kernel ada
```bash
ls -lah arch/arm64/boot/Image
ls -lah arch/arm64/boot/Image.gz
```

**Catatan penting (sesuai yang terbukti WORK di device kamu):**
- Untuk repack boot image, gunakan **`arch/arm64/boot/Image`** (bukan `Image.gz`).

---

## 2) Ekstrak partisi dari Firmware AP (Stock dtbo/vbmeta)
> Dipakai oleh kedua mode (stock base & magisk base).

```bash
cd /mnt/DATA/SM-A135F/raw
mkdir -p stock_part && cd stock_part

tar -xvf ../AP_*.tar.md5 dtbo.img.lz4 vbmeta.img.lz4 boot.img.lz4
lz4 -d dtbo.img.lz4 dtbo_stock.img
lz4 -d vbmeta.img.lz4 vbmeta_stock.img
lz4 -d boot.img.lz4 boot_stock.img

ls -lah dtbo_stock.img vbmeta_stock.img boot_stock.img
```

---

# MODE A — Base **STOCK** (tanpa Magisk / tanpa root)
## A1) Unpack boot_stock.img
```bash
cd /mnt/DATA/SM-A135F/raw
mkdir -p work_ap && cd work_ap

rm -rf bootstock_unpack
mkdir -p bootstock_unpack

python3 ~/android-tools/mkbootimg/unpack_bootimg.py \
  --boot_img /mnt/DATA/SM-A135F/raw/stock_part/boot_stock.img \
  --out bootstock_unpack

ls -lah bootstock_unpack
```

Pastikan ada minimal: `kernel`, `ramdisk`, `dtb`.

## A2) Replace kernel dengan hasil build v1 (Image)
```bash
cp -f /mnt/DATA/SM-A135F/src/Kernel/arch/arm64/boot/Image bootstock_unpack/kernel
file bootstock_unpack/kernel
```

## A3) Repack boot (parameter lengkap yang cocok untuk device kamu)
```bash
python3 ~/android-tools/mkbootimg/mkbootimg.py \
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
  --kernel bootstock_unpack/kernel \
  --ramdisk bootstock_unpack/ramdisk \
  --dtb bootstock_unpack/dtb \
  --output boot_stock_v1.img

file boot_stock_v1.img
ls -lah boot_stock_v1.img
```

## A4) Flash via Heimdall (split step, sesuai yang terbukti stabil)
Masuk **Download Mode** dulu.

```bash
sudo heimdall detect

# 1) flash BOOT, jangan reboot
sudo heimdall flash --BOOT /mnt/DATA/SM-A135F/raw/work_ap/boot_stock_v1.img --no-reboot

# 2) flash DTBO + VBMETA stock, resume
sudo heimdall flash \
  --DTBO /mnt/DATA/SM-A135F/raw/stock_part/dtbo_stock.img \
  --VBMETA /mnt/DATA/SM-A135F/raw/stock_part/vbmeta_stock.img \
  --resume
```

---

# MODE B — Base **boot-magisk.img** (root Magisk tetap)
> Ini mode yang kamu pakai dan sudah **WORKS**: ramdisk Magisk dipertahankan, kernel diganti.

## B0) Siapkan `boot-magisk.img`
Letakkan file `boot-magisk.img` di:
- `/mnt/DATA/SM-A135F/raw/work_ap/boot-magisk.img`

Cek format (harus sama dengan stock):
```bash
cd /mnt/DATA/SM-A135F/raw/work_ap
file /mnt/DATA/SM-A135F/raw/stock_part/boot_stock.img boot-magisk.img
```

## B1) Unpack boot-magisk.img
```bash
cd /mnt/DATA/SM-A135F/raw/work_ap

rm -rf bootmagisk_unpack
mkdir -p bootmagisk_unpack

python3 ~/android-tools/mkbootimg/unpack_bootimg.py \
  --boot_img boot-magisk.img \
  --out bootmagisk_unpack

ls -lah bootmagisk_unpack
```

Pastikan ada minimal: `kernel`, `ramdisk`, `dtb` (ramdisk ini = Magisk).

## B2) Replace kernel dengan hasil build v1 (Image)
```bash
cp -f /mnt/DATA/SM-A135F/src/Kernel/arch/arm64/boot/Image bootmagisk_unpack/kernel
file bootmagisk_unpack/kernel
```

## B3) Repack boot (parameter lengkap, sama seperti mode stock)
```bash
python3 ~/android-tools/mkbootimg/mkbootimg.py \
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
  --kernel bootmagisk_unpack/kernel \
  --ramdisk bootmagisk_unpack/ramdisk \
  --dtb bootmagisk_unpack/dtb \
  --output boot-kernelcustom-magisk.img

file boot-kernelcustom-magisk.img
ls -lah boot-kernelcustom-magisk.img
```

## B4) Flash via Heimdall (split step, dtbo/vbmeta tetap stock)
Masuk **Download Mode** dulu.

```bash
sudo heimdall detect

# 1) flash BOOT (Magisk base + kernel custom), jangan reboot
sudo heimdall flash --BOOT /mnt/DATA/SM-A135F/raw/work_ap/boot-kernelcustom-magisk.img --no-reboot

# 2) flash DTBO + VBMETA stock, resume
sudo heimdall flash \
  --DTBO /mnt/DATA/SM-A135F/raw/stock_part/dtbo_stock.img \
  --VBMETA /mnt/DATA/SM-A135F/raw/stock_part/vbmeta_stock.img \
  --resume
```

---

## 3) Verifikasi Setelah Boot
### 3.1 Verifikasi kernel string
Di Android (adb shell / terminal):
```sh
uname -a
cat /proc/version
```

### 3.2 Verifikasi root (Mode B)
```sh
su -c id
```

---

## 4) Recovery cepat kalau bootloop
### 4.1 Flash balik stock (BOOT + DTBO + VBMETA)
```bash
sudo heimdall detect
sudo heimdall flash --BOOT /mnt/DATA/SM-A135F/raw/stock_part/boot_stock.img --no-reboot
sudo heimdall flash \
  --DTBO /mnt/DATA/SM-A135F/raw/stock_part/dtbo_stock.img \
  --VBMETA /mnt/DATA/SM-A135F/raw/stock_part/vbmeta_stock.img \
  --resume
```

---

## 5) Notes penting (biar tetap stabil)
- Repack dengan parameter lengkap (header v2 + offsets + board + dtb_offset + tags_offset) adalah kunci supaya **nggak bootloop cepat**.
- Flash split (BOOT `--no-reboot`, lalu DTBO+VBMETA `--resume`) adalah pola yang **terbukti paling aman** untuk device kamu.
- `Image` vs `Image.gz`: untuk setup kamu, yang terbukti berhasil adalah **`Image`**.
