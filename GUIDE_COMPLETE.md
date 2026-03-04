# 🔧 Panduan Lengkap Build Kernel Samsung Galaxy A13 5G (SM-A135F)
### Branch: `rksu` — Kernel dengan KernelSU (rsuntk fork)

> **Base firmware:** `A135FXXSDEYJ1` (XID Region)  
> **SoC:** Samsung Exynos 3830 (universal3830)  
> **Android:** Android 12 (S)  
> **KernelSU:** rsuntk/KernelSU fork (Non-GKI, Manual Hook)  
> **Sumber build:** Berdasarkan `README_Kernel.txt` resmi Samsung

---

## 📋 Daftar Isi

1. [Tentang Proyek Ini](#1-tentang-proyek-ini)
2. [Prasyarat & Dependensi](#2-prasyarat--dependensi)
3. [Setup Lingkungan Build](#3-setup-lingkungan-build)
4. [Kloning Repository](#4-kloning-repository)
5. [Konfigurasi Toolchain](#5-konfigurasi-toolchain)
6. [Memahami Build Config](#6-memahami-build-config)
7. [Tentang KernelSU (RKSU)](#7-tentang-kernelsu-rksu)
8. [Proses Build Kernel](#8-proses-build-kernel)
9. [Output Build](#9-output-build)
10. [Membuat Flashable ZIP](#10-membuat-flashable-zip)
11. [Flashing ke Perangkat](#11-flashing-ke-perangkat)
12. [Verifikasi KernelSU](#12-verifikasi-kernelsu)
13. [Troubleshooting](#13-troubleshooting)
14. [Referensi & Kredit](#14-referensi--kredit)

---

## 1. Tentang Proyek Ini

Repository ini berisi kernel source Samsung Galaxy A13 5G (SM-A135F) yang telah diintegrasikan dengan **KernelSU** menggunakan fork dari [rsuntk/KernelSU](https://github.com/rsuntk/KernelSU) — fork yang dirancang khusus untuk mendukung kernel Non-GKI (4.4 ~ 6.18+).

### Spesifikasi Teknis

| Properti | Detail |
|---|---|
| **Perangkat** | Samsung Galaxy A13 5G (SM-A135F) |
| **SoC** | Samsung Exynos 3830 |
| **Arsitektur** | ARM64 (aarch64) |
| **Versi Android** | Android 12 (S) |
| **Versi Kernel** | Linux 5.4.x |
| **Defconfig** | `exynos850-a13xx_defconfig` |
| **Clang Resmi** | `clang-r383902` |
| **Hook Method** | Manual Hook (Non-GKI) |
| **KernelSU** | rsuntk fork @ commit `5402479` |
| **Region** | XID (A135FXXSDEYJ1) |

### Perbedaan Branch

| Branch | Deskripsi |
|---|---|
| `v1` | Kernel stock Samsung, tanpa modifikasi root, tanpa submodule |
| `rksu` | Kernel dengan KernelSU (rsuntk fork) terintegrasi sebagai submodule |

---

## 2. Prasyarat & Dependensi

### Sistem Operasi
**Ubuntu 20.04 LTS** atau **Ubuntu 22.04 LTS** sangat direkomendasikan.

> ⚠️ **Windows tidak didukung secara native.** Gunakan WSL2 atau mesin virtual Ubuntu.

### Spesifikasi Hardware

| Komponen | Minimum | Direkomendasikan |
|---|---|---|
| **RAM** | 8 GB | 16 GB atau lebih |
| **CPU** | 4 core | 8 core atau lebih |
| **Storage** | 50 GB | 80 GB (dengan ccache) |
| **Koneksi** | Diperlukan | Broadband stabil |

### Instalasi Paket Sistem

```bash
sudo apt update && sudo apt upgrade -y

sudo apt install -y \
    git curl wget zip unzip bc bison flex \
    libssl-dev libelf-dev make gcc g++ \
    python3 python3-pip cpio kmod \
    build-essential libncurses5-dev libncursesw5-dev \
    rsync ccache lld llvm lz4 zstd \
    device-tree-compiler adb fastboot \
    xz-utils tar
```

---

## 3. Setup Lingkungan Build

### 3.1 Setup CCache (Sangat Direkomendasikan)

```bash
sudo apt install ccache -y
ccache --max-size=50G

cat >> ~/.bashrc << 'EOF'

# CCache untuk kernel build
export USE_CCACHE=1
export CCACHE_EXEC=$(which ccache)
export CCACHE_DIR=$HOME/.ccache
EOF

source ~/.bashrc
```

### 3.2 Tingkatkan Batas File Descriptor

```bash
echo "$(whoami) soft nofile 65536" | sudo tee -a /etc/security/limits.conf
echo "$(whoami) hard nofile 65536" | sudo tee -a /etc/security/limits.conf
```

---

## 4. Kloning Repository

### 4.1 Clone dengan Submodule (WAJIB!)

Branch `rksu` menggunakan **KernelSU sebagai git submodule**. Harus menggunakan flag `--recurse-submodules` saat clone:

```bash
git clone --recurse-submodules -b rksu \
    https://github.com/kucingsakti/android_kernel_a135f.git \
    kernel_a135f_rksu

cd kernel_a135f_rksu
```

### 4.2 Jika Sudah Clone Tanpa Submodule

```bash
cd kernel_a135f_rksu
git submodule update --init --recursive
```

### 4.3 Verifikasi Submodule KernelSU

```bash
# Cek status submodule
git submodule status
# Output: 5402479cfa418753593032f960aa3de20bbf7fb7 KernelSU (...)

# Pastikan folder KernelSU tidak kosong
ls -la KernelSU/
# Harus ada: kernel/, manager/, userspace/, dll.
```

### 4.4 Update Submodule (Jika Diperlukan)

```bash
git submodule update --remote KernelSU
```

---

## 5. Konfigurasi Toolchain

Berdasarkan `README_Kernel.txt` resmi Samsung, ada **tiga komponen toolchain** yang harus dikonfigurasi:

| Variabel | Nilai (dari README resmi) | Keterangan |
|---|---|---|
| `CROSS_COMPILE` | `.../aarch64-linux-android-4.9/bin/aarch64-linux-android-` | GCC cross-compiler AArch64 |
| `CC` | `.../clang-r383902/bin/clang` | Clang compiler utama |
| `CLANG_TRIPLE` | `.../clang-r383902/bin/aarch64-linux-gnu-` | Prefix binutils di dir bin Clang |

> 💡 **Catatan penting:** `CLANG_TRIPLE` **bukan** toolchain terpisah. Ini adalah path prefix yang mengarah ke file-file `aarch64-linux-gnu-*` yang ada **di dalam direktori `bin/` Clang itu sendiri**.

### 5.1 Download Clang r383902 (Resmi Samsung)

```bash
mkdir -p ~/toolchains && cd ~/toolchains

# Opsi A: Clone dari mirror (tercepat)
git clone --depth=1 \
    https://github.com/rsuntk/android_prebuilts_clang_host_linux-x86_clang-r383902b.git \
    clang-r383902

# Opsi B: Download dari AOSP
mkdir -p clang-r383902
wget -O /tmp/clang-r383902.tar.gz \
    "https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86/+archive/refs/heads/android12-release/clang-r383902b.tar.gz"
tar -xzf /tmp/clang-r383902.tar.gz -C clang-r383902/
```

Verifikasi:
```bash
~/toolchains/clang-r383902/bin/clang --version
# Output: Android (7284624, based on r383902) clang version 11.0.x ...
```

### 5.2 Download GCC AArch64 4.9

```bash
cd ~/toolchains

git clone --depth=1 \
    https://github.com/LineageOS/android_prebuilts_gcc_linux-x86_aarch64_aarch64-linux-android-4.9.git \
    gcc-aarch64
```

### 5.3 Buat Script Wrapper Toolchain

```bash
cat > ~/setup_toolchain_a135f.sh << 'EOF'
#!/bin/bash
# Setup toolchain resmi Samsung untuk build kernel A135F

CLANG_DIR="$HOME/toolchains/clang-r383902"
GCC64_DIR="$HOME/toolchains/gcc-aarch64"

[ ! -f "$CLANG_DIR/bin/clang" ] && echo "❌ Clang tidak ditemukan" && return 1
[ ! -f "$GCC64_DIR/bin/aarch64-linux-android-gcc" ] && echo "❌ GCC tidak ditemukan" && return 1

export PATH="$CLANG_DIR/bin:$GCC64_DIR/bin:$PATH"
export PLATFORM_VERSION=12
export ANDROID_MAJOR_VERSION=s
export ARCH=arm64
export SUBARCH=arm64
export USE_CCACHE=1
export CCACHE_EXEC=$(which ccache)

echo "✅ Toolchain siap:"
echo "   Clang : $(clang --version | head -1)"
echo "   GCC64 : $(aarch64-linux-android-gcc --version | head -1)"
echo "   PLATFORM_VERSION      = $PLATFORM_VERSION"
echo "   ANDROID_MAJOR_VERSION = $ANDROID_MAJOR_VERSION"
echo "   ARCH                  = $ARCH"
EOF

chmod +x ~/setup_toolchain_a135f.sh
```

Gunakan dengan: `source ~/setup_toolchain_a135f.sh`

---

## 6. Memahami Build Config

### 6.1 Defconfig Resmi Samsung

Berdasarkan `README_Kernel.txt` resmi:

```
arch/arm64/configs/exynos850-a13xx_defconfig
```

> ⚠️ Nama defconfig adalah `exynos850-a13xx_defconfig`. Beberapa referensi online menyebutkan nama berbeda — selalu gunakan yang tercantum di README resmi Samsung.

### 6.2 File Build Config yang Relevan

| File | Keterangan |
|---|---|
| `build.config.universal3830_s` | **Config utama** untuk Exynos 3830, Android S (12) |
| `build.config.universal3830_stu_mr` | Config untuk Android S + maintenance release |
| `build.config.erd3830` | Config ERD (Engineering Reference Design) board |
| `build.config.common` | Config base yang di-include semua config |

### 6.3 Struktur KernelSU di Repo Ini

```
kernel_a135f_rksu/
├── KernelSU/                    ← Git submodule rsuntk/KernelSU @ 5402479
│   ├── kernel/                  ← Komponen kernel KSU (GPL-2.0)
│   │   ├── ksu.c
│   │   ├── sucompat.c
│   │   └── ...
│   ├── manager/                 ← Android Manager App
│   └── userspace/ksud/          ← KSU Daemon (Rust)
├── arch/arm64/configs/
│   └── exynos850-a13xx_defconfig  ← Defconfig (sudah include CONFIG_KSU)
└── .gitmodules                  ← Definisi submodule KernelSU
```

---

## 7. Tentang KernelSU (RKSU)

### Mengapa Menggunakan rsuntk Fork?

Official KernelSU (tiann/KernelSU) **telah menghentikan dukungan untuk Non-GKI kernels**. Exynos 3830 menjalankan kernel Non-GKI (kernel 5.4 Samsung), sehingga repo ini menggunakan fork [rsuntk/KernelSU](https://github.com/rsuntk/KernelSU) yang masih mendukung kernel 4.4–6.18+.

### Hook Method: Manual Hook

Karena kernel ini **Non-GKI** dan `CONFIG_KPROBES` dimatikan oleh Samsung secara default, repo ini menggunakan **Manual Hook**:

| Hook Method | Digunakan | Keterangan |
|---|---|---|
| **Manual Hook** | ✅ Ya | Patch langsung ke kernel source, tidak butuh KPROBES |
| Syscall Hook | ❌ Tidak | Butuh `CONFIG_KPROBES=y`, tidak tersedia di Non-GKI Samsung |

Config KernelSU yang harus aktif di defconfig:
```
CONFIG_KSU=y
CONFIG_KSU_MANUAL_HOOK=y
```

### Verifikasi KSU Config di Defconfig

```bash
grep "CONFIG_KSU" arch/arm64/configs/exynos850-a13xx_defconfig
# Output yang diharapkan:
# CONFIG_KSU=y
# CONFIG_KSU_MANUAL_HOOK=y
```

### Update KernelSU ke Versi Terbaru (Opsional)

> ⚠️ Submodule yang sudah ada di branch `rksu` sudah teruji kompatibel. Update hanya jika tahu risikonya.

```bash
# Update ke main (terbaru)
curl -LSs "https://raw.githubusercontent.com/rsuntk/KernelSU/main/kernel/setup.sh" | bash -s main

# Update ke tag tertentu
curl -LSs "https://raw.githubusercontent.com/rsuntk/KernelSU/main/kernel/setup.sh" | bash -s v3.0.0-30-legacy
```

---

## 8. Proses Build Kernel

Build rksu menggunakan proses yang **identik dengan v1**, dengan tambahan bahwa defconfig sudah mengandung `CONFIG_KSU=y` dan `CONFIG_KSU_MANUAL_HOOK=y`.

---

### ⭐ Metode A: Edit Makefile (Cara Resmi Samsung)

**Langkah 1 — Edit Makefile:**

```bash
cd kernel_a135f_rksu
nano Makefile
```

Cari dan ubah tiga baris berikut:

```makefile
CROSS_COMPILE = /home/YOUR_USER/toolchains/gcc-aarch64/bin/aarch64-linux-android-
CC            = /home/YOUR_USER/toolchains/clang-r383902/bin/clang
CLANG_TRIPLE  = /home/YOUR_USER/toolchains/clang-r383902/bin/aarch64-linux-gnu-
```

**Langkah 2 — Export variabel & build (sesuai README Samsung):**

```bash
export PLATFORM_VERSION=12
export ANDROID_MAJOR_VERSION=s
export ARCH=arm64

# Generate defconfig
make exynos850-a13xx_defconfig

# Build kernel
make -j$(nproc) 2>&1 | tee build.log
```

---

### Metode B: Argumen Command Line (Tanpa Edit Makefile)

```bash
cd kernel_a135f_rksu
source ~/setup_toolchain_a135f.sh

CLANG_DIR="$HOME/toolchains/clang-r383902"
GCC64_DIR="$HOME/toolchains/gcc-aarch64"
THREADS=$(nproc --all)

# Generate defconfig
make \
    ARCH=arm64 \
    PLATFORM_VERSION=12 \
    ANDROID_MAJOR_VERSION=s \
    CC=$CLANG_DIR/bin/clang \
    CLANG_TRIPLE=$CLANG_DIR/bin/aarch64-linux-gnu- \
    CROSS_COMPILE=$GCC64_DIR/bin/aarch64-linux-android- \
    exynos850-a13xx_defconfig

# Build kernel
make \
    ARCH=arm64 \
    PLATFORM_VERSION=12 \
    ANDROID_MAJOR_VERSION=s \
    CC=$CLANG_DIR/bin/clang \
    CLANG_TRIPLE=$CLANG_DIR/bin/aarch64-linux-gnu- \
    CROSS_COMPILE=$GCC64_DIR/bin/aarch64-linux-android- \
    -j$THREADS 2>&1 | tee build.log
```

---

### 8.1 Menggunakan `build_kernel.sh`

```bash
cd kernel_a135f_rksu
cat build_kernel.sh        # Periksa path toolchain
chmod +x build_kernel.sh
./build_kernel.sh
```

### 8.2 Verifikasi KernelSU Terbuild

```bash
# Setelah defconfig di-generate, cek config KSU aktif
grep "CONFIG_KSU" .config
# Output yang diharapkan:
# CONFIG_KSU=y
# CONFIG_KSU_MANUAL_HOOK=y
```

### 8.3 Cek Hasil Build

```bash
if [ $? -eq 0 ]; then
    echo "✅ Build BERHASIL"
    ls -lh arch/arm64/boot/Image
else
    echo "❌ Build GAGAL — cek build.log"
    grep -i "error:" build.log | tail -20
fi
```

### 8.4 Clean Build

```bash
make clean      # Clean output build (sesuai README resmi Samsung)
make mrproper   # Clean menyeluruh termasuk generated config
```

---

## 9. Output Build

Berdasarkan `README_Kernel.txt` resmi Samsung:

### 9.1 Kernel Image

```
arch/arm64/boot/Image
```

```bash
ls -lh arch/arm64/boot/Image
file arch/arm64/boot/Image
# Output: Linux kernel ARM64 boot executable Image, ...
```

### 9.2 Kernel Modules

```
drivers/*/*.ko
```

```bash
find . -name "*.ko" -not -path "./.git/*" | sort
```

> 💡 Output resmi adalah `Image` (uncompressed). Jika perlu format lain untuk AnyKernel3, lihat seksi Troubleshooting.

---

## 10. Membuat Flashable ZIP

### 10.1 Clone AnyKernel3

```bash
cd ~
git clone https://github.com/osm0sis/AnyKernel3.git AnyKernel3_a135f_rksu
cd AnyKernel3_a135f_rksu
```

### 10.2 Konfigurasi `anykernel.sh`

```bash
cat > anykernel.sh << 'EOF'
# AnyKernel3 Ramdisk Mod Script
# osm0sis @ xda-developers

## AnyKernel setup
properties() { '
kernel.string=RKSU Kernel for Samsung Galaxy A13 5G (SM-A135F)
do.devicecheck=1
do.modules=0
do.systemless=1
do.cleanup=1
do.cleanuponabort=0
device.name1=a13x
device.name2=a13xnsxx
device.name3=SM-A135F
device.name4=a135f
supported.versions=12
'; }

# shell variables
block=/dev/block/platform/13500000.ufs/by-name/boot;
is_slot_device=0;
ramdisk_compression=auto;
patch_vbmeta_flag=auto;

## AnyKernel methods (DO NOT CHANGE)
. tools/ak3-core.sh;

## AnyKernel install
split_boot;
flash_boot;
## end install
EOF
```

### 10.3 Copy Kernel Image

Output resmi Samsung adalah `arch/arm64/boot/Image`:

```bash
# Hapus file contoh bawaan AnyKernel3
rm -f Image Image.gz Image.gz-dtb zImage *.zip

# Copy kernel image
cp ~/kernel_a135f_rksu/arch/arm64/boot/Image ~/AnyKernel3_a135f_rksu/

ls -lh ~/AnyKernel3_a135f_rksu/Image
```

### 10.4 Buat ZIP

```bash
cd ~/AnyKernel3_a135f_rksu

ZIP_NAME="RKSU_Kernel_A135F_$(date +%Y%m%d_%H%M%S).zip"

zip -r9 "$ZIP_NAME" . -x "*.git*" -x "*.zip"

echo "✅ ZIP: $ZIP_NAME ($(du -sh $ZIP_NAME | cut -f1))"
```

---

## 11. Flashing ke Perangkat

### Prasyarat
- ✅ Custom Recovery terpasang (TWRP)
- ✅ Firmware Samsung One UI 4.x / Android 12
- ✅ Data sudah di-backup
- ✅ Baterai minimal 50%

### 11.1 Transfer ke Perangkat

```bash
adb push ~/AnyKernel3_a135f_rksu/RKSU_Kernel_A135F_*.zip /sdcard/
adb shell ls -lh /sdcard/RKSU_Kernel_A135F_*.zip
```

### 11.2 Boot ke Recovery

```bash
adb reboot recovery
# Atau manual: tahan Volume Up + Power saat perangkat mati
```

### 11.3 Flash via TWRP

1. Tap **Install** → Pilih file ZIP
2. **Swipe to Confirm Flash**
3. **Reboot System**

### 11.4 Flash via Heimdall (Tanpa Recovery)

```bash
sudo apt install heimdall-flash -y

# Setup udev rules
echo 'SUBSYSTEM=="usb", ATTRS{idVendor}=="04e8", MODE="0666", GROUP="plugdev"' | \
    sudo tee /etc/udev/rules.d/51-samsung.rules
sudo udevadm control --reload-rules && sudo udevadm trigger
sudo usermod -aG plugdev $USER

# Boot ke Download Mode: Volume Down + Power → Volume Up
heimdall detect

# Flash Image
heimdall flash --BOOT ~/kernel_a135f_rksu/arch/arm64/boot/Image --no-reboot
```

---

## 12. Verifikasi KernelSU

### 12.1 Install KernelSU Manager

Download dari [rsuntk/KernelSU Releases](https://github.com/rsuntk/KernelSU/releases/latest):

```bash
adb install KernelSU_Manager.apk
```

### 12.2 Cek Status di Manager

Buka **KernelSU Manager** setelah reboot:
- ✅ **Working** — KernelSU aktif dan berfungsi normal
- ❌ **Not working** — Ada masalah dengan kernel hook (lihat Troubleshooting)

### 12.3 Verifikasi via ADB Shell

```bash
adb shell

# Cek versi kernel
uname -r
# Contoh output: 5.4.xxx-android12-9-...

# Cek versi KernelSU
su -v

# Test root access
su -c "id"
# Output: uid=0(root) gid=0(root) ...
```

---

## 13. Troubleshooting

### ❌ `make: exynos850-a13xx_defconfig: No rule to make target`

**Penyebab:** `ARCH=arm64` belum di-export.

```bash
export ARCH=arm64
make exynos850-a13xx_defconfig
```

---

### ❌ `clang: command not found` atau `aarch64-linux-android-gcc: command not found`

```bash
source ~/setup_toolchain_a135f.sh
which clang && which aarch64-linux-android-gcc
```

---

### ❌ Submodule KernelSU Kosong

**Penyebab:** Clone tanpa `--recurse-submodules`.

```bash
cd kernel_a135f_rksu
git submodule update --init --recursive
git submodule status
```

---

### ❌ KernelSU Manager Menampilkan "Not Working"

**Penyebab:** `CONFIG_KSU_MANUAL_HOOK` tidak aktif di defconfig.

```bash
# Cek config setelah generate defconfig
grep "CONFIG_KSU" .config

# Jika tidak ada, aktifkan di defconfig terlebih dahulu:
echo "CONFIG_KSU=y" >> arch/arm64/configs/exynos850-a13xx_defconfig
echo "CONFIG_KSU_MANUAL_HOOK=y" >> arch/arm64/configs/exynos850-a13xx_defconfig

# Regenerate dan rebuild
make ARCH=arm64 exynos850-a13xx_defconfig
make ARCH=arm64 PLATFORM_VERSION=12 ANDROID_MAJOR_VERSION=s \
     CC=$HOME/toolchains/clang-r383902/bin/clang \
     CLANG_TRIPLE=$HOME/toolchains/clang-r383902/bin/aarch64-linux-gnu- \
     CROSS_COMPILE=$HOME/toolchains/gcc-aarch64/bin/aarch64-linux-android- \
     -j$(nproc)
```

---

### ❌ `error: implicit declaration of function`

**Penyebab:** Menggunakan Clang versi berbeda dari yang direkomendasikan Samsung.

```bash
# Verifikasi versi Clang
clang --version | grep -i "r383902"
# Harus muncul di output
```

---

### ❌ Kernel Panic / Bootloop Setelah Flash

**Solusi — Restore kernel stock:**

```bash
# Download firmware A135FXXSDEYJ1 dari SamFw/SamMobile
lz4 -d boot.img.lz4 boot.img
heimdall flash --BOOT boot.img
```

---

### ❌ Build berhasil tapi AnyKernel3 butuh `Image.gz` atau `Image.gz-dtb`

Output resmi Samsung hanya `Image`. Buat format lain secara manual:

```bash
# Buat Image.gz
gzip -k -f arch/arm64/boot/Image

# Buat Image.gz-dtb (gabungkan Image.gz dengan DTB)
cat arch/arm64/boot/Image.gz \
    arch/arm64/boot/dts/exynos/exynos3830*.dtb \
    > arch/arm64/boot/Image.gz-dtb
```

---

### ❌ Error "CONFIG_KSU_SUSFS" tidak dikenali

**Penyebab:** Commit KernelSU yang digunakan di branch `rksu` belum mendukung susfs.

**Solusi:** Gunakan branch `susfs-rksu-master` jika ingin fitur susfs, tapi perhatikan branch tersebut tidak selalu diupdate.

---

### 🔍 Membaca Build Log

```bash
grep -n "error:" build.log
grep -n -B3 -A3 "error:" build.log | head -60
grep "^  CC " build.log | tail -5

# Build dengan log verbose
make ... -j$(nproc) 2>&1 | tee build.log
grep -i "error:" build.log | head -50
```

---

## 14. Referensi & Kredit

### Repository Utama

| Link | Keterangan |
|---|---|
| [kucingsakti/android_kernel_a135f (rksu)](https://github.com/kucingsakti/android_kernel_a135f/tree/rksu) | Kernel source (branch rksu) |
| [kucingsakti/android_kernel_a135f (v1)](https://github.com/kucingsakti/android_kernel_a135f/tree/v1) | Branch stock/vanilla |
| [rsuntk/KernelSU](https://github.com/rsuntk/KernelSU) | KernelSU fork Non-GKI |
| [rsuntk/KernelSU Releases](https://github.com/rsuntk/KernelSU/releases/latest) | Download Manager APK |
| [osm0sis/AnyKernel3](https://github.com/osm0sis/AnyKernel3) | Framework flashable ZIP kernel |

### Komunitas & Diskusi

| Link | Keterangan |
|---|---|
| [RKSU Telegram Group: @rsukrnlsu_grp](https://t.me/rsukrnlsu_grp) | Grup diskusi RKSU |
| [RKSU Telegram Channel: @rsukrnlsu](https://t.me/rsukrnlsu) | Channel update RKSU |
| [XDA Galaxy A13 Forum](https://xdaforums.com/f/samsung-galaxy-a13.12569/) | Forum komunitas |
| [SamFw Firmware A135F](https://samfw.com/firmware/SM-A135F) | Download firmware Samsung |

### Kredit
- **Samsung** — Kernel source asli + `README_Kernel.txt` sebagai acuan build resmi
- **tiann** — Penulis asli KernelSU
- **rsuntk** — Fork KernelSU yang mendukung Non-GKI kernel
- **kucingsakti** — Maintainer kernel source ini
- **osm0sis** — Penulis AnyKernel3

---

<div align="center">

**⚠️ DISCLAIMER**

*Modifikasi kernel dapat membatalkan garansi dan berpotensi merusak perangkat jika dilakukan secara tidak benar. Gunakan panduan ini atas risiko Anda sendiri. Selalu backup data sebelum melakukan modifikasi sistem.*

</div>

---

*Guide ini disusun berdasarkan analisis repository `kucingsakti/android_kernel_a135f` branch `rksu`, `README_Kernel.txt` resmi Samsung, dan dokumentasi rsuntk/KernelSU.*
