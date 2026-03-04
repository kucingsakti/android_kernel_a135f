# 🔧 Panduan Lengkap Build Kernel Samsung Galaxy A13 5G (SM-A135F)
### Branch: `rksu` — Kernel dengan KernelSU (rsuntk fork)

> **Base firmware:** `A135FXXSDEYJ1` (XID Region)  
> **SoC:** Samsung Exynos 3830 (universal3830)  
> **Android:** Android 12 (S)  
> **KernelSU:** rsuntk/KernelSU fork (Non-GKI, Manual Hook)

---

## 📋 Daftar Isi

1. [Tentang Proyek Ini](#1-tentang-proyek-ini)
2. [Prasyarat & Dependensi](#2-prasyarat--dependensi)
3. [Setup Lingkungan Build](#3-setup-lingkungan-build)
4. [Kloning Repository](#4-kloning-repository)
5. [Konfigurasi Toolchain](#5-konfigurasi-toolchain)
6. [Memahami Build Config](#6-memahami-build-config)
7. [Proses Build Kernel](#7-proses-build-kernel)
8. [Tentang KernelSU (RKSU)](#8-tentang-kernelsu-rksu)
9. [Membuat Flashable ZIP](#9-membuat-flashable-zip)
10. [Flashing ke Perangkat](#10-flashing-ke-perangkat)
11. [Verifikasi KernelSU](#11-verifikasi-kernelsu)
12. [Troubleshooting](#12-troubleshooting)
13. [Referensi & Kredit](#13-referensi--kredit)

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
| **Hook Method** | Manual Hook (Non-GKI) |
| **KernelSU** | rsuntk fork @ commit `5402479` |
| **Region** | XID (A135FXXSDEYJ1) |

### Perbedaan Branch
- **`main`** — Kernel stock Samsung tanpa modifikasi
- **`rksu`** — Kernel yang sudah diintegrasikan KernelSU (rsuntk fork)

---

## 2. Prasyarat & Dependensi

### Sistem Operasi
Gunakan **Ubuntu 20.04 LTS** atau **Ubuntu 22.04 LTS** (direkomendasikan). Debian dan turunannya juga bisa, namun Ubuntu paling banyak teruji.

> ⚠️ **Windows tidak didukung secara native.** Gunakan WSL2 di Windows jika diperlukan.

### Spesifikasi Hardware yang Direkomendasikan
- **RAM:** Minimal 8 GB (16 GB sangat direkomendasikan)
- **CPU:** 4 core atau lebih (build akan lebih lambat di 2 core)
- **Storage:** Minimal 50 GB ruang kosong
- **Koneksi Internet:** Diperlukan untuk mengunduh toolchain

### Paket yang Dibutuhkan

```bash
sudo apt update && sudo apt upgrade -y

sudo apt install -y \
    git \
    curl \
    wget \
    zip \
    unzip \
    bc \
    bison \
    flex \
    libssl-dev \
    libelf-dev \
    make \
    gcc \
    g++ \
    python3 \
    python3-pip \
    cpio \
    kmod \
    build-essential \
    libncurses5-dev \
    libncursesw5-dev \
    rsync \
    ccache \
    lld \
    llvm \
    clang \
    lz4 \
    zstd \
    device-tree-compiler \
    adb \
    fastboot
```

### Verifikasi Python
```bash
python3 --version   # Minimal Python 3.6
```

---

## 3. Setup Lingkungan Build

### 3.1 Setup CCache (Opsional tapi Sangat Direkomendasikan)

CCache akan mempercepat build ulang secara signifikan (hingga 5x lebih cepat):

```bash
# Install ccache
sudo apt install ccache -y

# Atur ukuran cache (sesuaikan dengan kapasitas storage)
ccache --max-size=50G

# Aktifkan ccache
export USE_CCACHE=1
export CCACHE_EXEC=$(which ccache)

# Tambahkan ke .bashrc agar permanen
echo 'export USE_CCACHE=1' >> ~/.bashrc
echo 'export CCACHE_EXEC=$(which ccache)' >> ~/.bashrc
echo 'export CCACHE_DIR=$HOME/.ccache' >> ~/.bashrc
source ~/.bashrc
```

### 3.2 Setup Variabel Lingkungan Dasar

```bash
# Tambahkan ke ~/.bashrc untuk persistensi
export ARCH=arm64
export SUBARCH=arm64
export ANDROID_MAJOR_VERSION=s
```

---

## 4. Kloning Repository

### 4.1 Clone dengan Submodule (PENTING!)

Branch `rksu` menggunakan **KernelSU sebagai git submodule**. Kamu **wajib** menggunakan flag `--recurse-submodules` saat clone, atau inisialisasi submodule secara manual setelahnya.

```bash
# Clone sekaligus dengan submodule (direkomendasikan)
git clone --recurse-submodules -b rksu \
    https://github.com/kucingsakti/android_kernel_a135f.git \
    kernel_a135f

cd kernel_a135f
```

### 4.2 Jika Sudah Clone Tanpa Submodule

Jika kamu sudah clone sebelumnya tanpa `--recurse-submodules`:

```bash
cd kernel_a135f
git submodule update --init --recursive
```

### 4.3 Verifikasi Submodule KernelSU

```bash
# Cek status submodule
git submodule status

# Output yang diharapkan:
# 5402479cfa418753593032f960aa3de20bbf7fb7 KernelSU (...)
```

Pastikan folder `KernelSU/` tidak kosong:

```bash
ls -la KernelSU/
# Seharusnya ada folder: kernel/, manager/, userspace/, dll.
```

### 4.4 Update Submodule (Jika Diperlukan)

```bash
git submodule update --remote KernelSU
```

---

## 5. Konfigurasi Toolchain

Kernel Exynos 3830 menggunakan **Clang** sebagai compiler utama. Kamu membutuhkan Clang versi yang kompatibel.

### Opsi A: Menggunakan Toolchain Samsung (Direkomendasikan)

Samsung menyediakan toolchain Clang khusus untuk kernel mereka:

```bash
mkdir -p ~/toolchains && cd ~/toolchains

# Clone toolchain Clang Samsung (gunakan Clang 12 atau sesuai versi di build config)
git clone --depth=1 \
    https://github.com/rsuntk/android_prebuilts_clang_host_linux-x86_clang-r383902b.git \
    clang-r383902b

# Atau gunakan AOSP Clang
git clone --depth=1 \
    https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86/+archive/refs/heads/android12-release/clang-r416183b.tar.gz
```

### Opsi B: Menggunakan AOSP Prebuilt Clang

```bash
mkdir -p ~/toolchains/clang && cd ~/toolchains/clang

# Download AOSP Clang r416183b (dipakai untuk Android 12)
wget https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86/+archive/refs/heads/android12-release/clang-r416183b.tar.gz

tar -xzf clang-r416183b.tar.gz
```

### Opsi C: Menggunakan ProtonClang (Alternatif)

```bash
mkdir -p ~/toolchains && cd ~/toolchains

git clone --depth=1 \
    https://github.com/kdrag0n/proton-clang.git \
    proton-clang
```

### 5.1 Setup GCC Cross Compiler (Untuk AArch64 dan ARM32)

Beberapa driver kernel memerlukan GCC sebagai cross compiler:

```bash
mkdir -p ~/toolchains && cd ~/toolchains

# GCC AArch64
git clone --depth=1 \
    https://github.com/LineageOS/android_prebuilts_gcc_linux-x86_aarch64_aarch64-linux-android-4.9.git \
    gcc-aarch64

# GCC ARM32 (untuk compat layer)
git clone --depth=1 \
    https://github.com/LineageOS/android_prebuilts_gcc_linux-x86_arm_arm-linux-androideabi-4.9.git \
    gcc-arm
```

### 5.2 Verifikasi Toolchain

```bash
~/toolchains/clang/bin/clang --version
# Output: clang version 12.x.x ...

~/toolchains/gcc-aarch64/bin/aarch64-linux-android-gcc --version
# Output: aarch64-linux-android-gcc (GCC) 4.9.x ...
```

---

## 6. Memahami Build Config

Repository ini memiliki banyak file `build.config.*`. Untuk SM-A135F (Exynos 3830, Android 12), gunakan konfigurasi berikut:

### File Build Config yang Relevan

| File | Keterangan |
|---|---|
| `build.config.universal3830_s` | **Config utama** untuk Exynos 3830, Android S (12) |
| `build.config.universal3830_stu_mr` | Config untuk Android S dengan maintenance release |
| `build.config.erd3830` | Config ERD (Engineering Reference Design) board |
| `build.config.common` | Config umum yang di-include semua config |

### Melihat Isi Build Config

```bash
cat build.config.universal3830_s
```

### Defconfig Kernel

Defconfig untuk perangkat ini berada di:
```
arch/arm64/configs/exynos3830-a13xnsxx_defconfig
```

Atau bisa juga:
```
arch/arm64/configs/a13x_defconfig
```

---

## 7. Proses Build Kernel

### 7.1 Menggunakan Script `build_kernel.sh` (Cara Mudah)

Repository sudah menyediakan script build otomatis. Ini cara yang paling mudah:

```bash
cd kernel_a135f

# Pastikan script executable
chmod +x build_kernel.sh

# Lihat isi script terlebih dahulu
cat build_kernel.sh

# Jalankan build
./build_kernel.sh
```

### 7.2 Build Manual (Cara Lengkap)

Jika ingin build secara manual dengan kontrol penuh:

```bash
cd kernel_a135f

# --- Variabel Environment ---
export ARCH=arm64
export SUBARCH=arm64
export ANDROID_MAJOR_VERSION=s

# Path toolchain (sesuaikan dengan lokasi toolchain kamu)
CLANG_PATH="$HOME/toolchains/clang/bin"
GCC64_PATH="$HOME/toolchains/gcc-aarch64/bin"
GCC32_PATH="$HOME/toolchains/gcc-arm/bin"

export PATH="$CLANG_PATH:$GCC64_PATH:$GCC32_PATH:$PATH"

# Compiler flags
export CC=clang
export CLANG_TRIPLE=aarch64-linux-gnu-
export CROSS_COMPILE=aarch64-linux-android-
export CROSS_COMPILE_ARM32=arm-linux-androideabi-

# Jumlah thread build (gunakan jumlah CPU core + 2)
THREADS=$(nproc --all)

# --- Direktori Output ---
mkdir -p out

# --- Generate Defconfig ---
make O=out \
     ARCH=arm64 \
     CC=clang \
     CROSS_COMPILE=aarch64-linux-android- \
     CROSS_COMPILE_ARM32=arm-linux-androideabi- \
     CLANG_TRIPLE=aarch64-linux-gnu- \
     exynos3830-a13xnsxx_defconfig

# --- (Opsional) Modifikasi Config ---
# make O=out menuconfig   # GUI berbasis ncurses

# --- Build Kernel ---
make O=out \
     ARCH=arm64 \
     CC=clang \
     CROSS_COMPILE=aarch64-linux-android- \
     CROSS_COMPILE_ARM32=arm-linux-androideabi- \
     CLANG_TRIPLE=aarch64-linux-gnu- \
     -j$THREADS 2>&1 | tee build.log
```

### 7.3 Hasil Build

Jika build berhasil, kamu akan menemukan file berikut:

```
out/arch/arm64/boot/Image          # Kernel image (raw)
out/arch/arm64/boot/Image.gz       # Kernel image (compressed)
out/arch/arm64/boot/Image.gz-dtb   # Kernel + DTB (untuk Samsung)
out/arch/arm64/boot/dts/           # Device Tree Blobs
```

> ⚠️ Samsung biasanya menggunakan `Image.gz-dtb` atau `Image` tergantung firmware. Cek format yang dipakai di AnyKernel3 template untuk A135F.

### 7.4 Memastikan KernelSU Terbuild

```bash
# Cek apakah KernelSU config aktif
grep -i "CONFIG_KSU" out/.config

# Output yang diharapkan:
# CONFIG_KSU=y
# CONFIG_KSU_MANUAL_HOOK=y
# CONFIG_KSU_DEBUG=n (atau y jika debug build)
```

---

## 8. Tentang KernelSU (RKSU)

### Apa itu KernelSU?

KernelSU adalah solusi root berbasis kernel untuk perangkat Android. Berbeda dengan Magisk yang beroperasi di userspace, KernelSU bekerja langsung di level kernel sehingga lebih sulit dideteksi.

### Mengapa `rksu` Branch Menggunakan rsuntk Fork?

Official KernelSU (tiann/KernelSU) **telah menghentikan dukungan untuk Non-GKI kernels**. Exynos 3830 menjalankan kernel Non-GKI (kernel 5.4 Samsung), sehingga menggunakan fork [rsuntk/KernelSU](https://github.com/rsuntk/KernelSU) yang masih mendukung kernel 4.4 hingga 6.18+.

### Hook Method: Manual Hook

Karena kernel ini **Non-GKI** dan `CONFIG_KPROBES` kemungkinan dimatikan secara default oleh Samsung, proyek ini menggunakan **Manual Hook**:

- Config yang dibutuhkan: `CONFIG_KSU_MANUAL_HOOK=y`
- Tidak memerlukan `CONFIG_KPROBES`
- Patch manual langsung ke kernel source

### Struktur KernelSU di Repo Ini

```
KernelSU/                    # Git submodule dari rsuntk/KernelSU
├── kernel/                  # Komponen kernel KernelSU (GPL-2.0)
│   ├── ksu.c
│   ├── sucompat.c
│   └── ...
├── manager/                 # Android Manager App
└── userspace/ksud/          # KSU Daemon (Rust)
```

### Menambahkan/Update KernelSU Secara Manual

Jika ingin update KernelSU ke versi terbaru:

```bash
# Dari root kernel source
curl -LSs "https://raw.githubusercontent.com/rsuntk/KernelSU/main/kernel/setup.sh" | bash -s main

# Atau target branch susfs (lebih eksperimental)
# curl -LSs "https://raw.githubusercontent.com/rsuntk/KernelSU/main/kernel/setup.sh" | bash -s susfs-rksu-master
```

> ⚠️ Sebaiknya gunakan submodule yang sudah ada di branch `rksu` karena sudah teruji kompatibel.

---

## 9. Membuat Flashable ZIP

Setelah kernel berhasil dibuild, bungkus menggunakan **AnyKernel3**:

### 9.1 Clone AnyKernel3

```bash
cd ~
git clone https://github.com/osm0sis/AnyKernel3.git AnyKernel3_a135f
cd AnyKernel3_a135f
```

### 9.2 Konfigurasi AnyKernel3

Edit file `anykernel.sh`:

```bash
# Ganti isi anykernel.sh dengan konfigurasi berikut:
cat > anykernel.sh << 'EOF'
# AnyKernel3 Ramdisk Mod Script
# osm0sis @ xda-developers

## AnyKernel setup
# global properties
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

### 9.3 Copy Kernel Image

```bash
# Copy kernel image ke folder AnyKernel3
cp ~/kernel_a135f/out/arch/arm64/boot/Image.gz-dtb ~/AnyKernel3_a135f/

# Atau jika menggunakan format Image saja:
# cp ~/kernel_a135f/out/arch/arm64/boot/Image ~/AnyKernel3_a135f/
```

### 9.4 Buat ZIP

```bash
cd ~/AnyKernel3_a135f

# Hapus file yang tidak perlu
rm -f *.zip

# Buat ZIP
zip -r9 RKSU_Kernel_A135F_$(date +%Y%m%d).zip . -x "*.git*" "*.zip"

echo "ZIP berhasil dibuat: RKSU_Kernel_A135F_$(date +%Y%m%d).zip"
```

---

## 10. Flashing ke Perangkat

### Prasyarat Flashing
- Custom Recovery terpasang (TWRP direkomendasikan)
- Minimal Samsung One UI 4.x / Android 12
- Data backup sudah dilakukan
- Baterai minimal 50%

### 10.1 Transfer ZIP ke Perangkat

```bash
# Via ADB
adb push ~/AnyKernel3_a135f/RKSU_Kernel_A135F_*.zip /sdcard/

# Verifikasi transfer
adb shell ls -la /sdcard/RKSU_Kernel_A135F_*.zip
```

### 10.2 Boot ke Recovery

```bash
# Via ADB
adb reboot recovery

# Atau tekan tombol: Volume Up + Power saat perangkat mati
```

### 10.3 Flash via TWRP

1. Di TWRP, pilih **Install**
2. Navigasi ke file ZIP kernel
3. Swipe untuk konfirmasi flash
4. Tunggu hingga proses selesai
5. Pilih **Reboot System**

### 10.4 Flash via Heimdall (Alternatif — Tanpa Recovery)

Jika tidak memiliki custom recovery, bisa flash langsung via Heimdall:

```bash
# Install Heimdall
sudo apt install heimdall-flash -y

# Boot perangkat ke Download Mode
# Tekan: Volume Down + Power, lalu Volume Up untuk konfirmasi

# Flash kernel image langsung
heimdall flash --BOOT out/arch/arm64/boot/Image.gz-dtb

# Atau menggunakan Odin (di Windows) — flash ke slot AP/PDA
```

---

## 11. Verifikasi KernelSU

### 11.1 Install KernelSU Manager

Download KernelSU Manager APK dari:
- [rsuntk/KernelSU Releases](https://github.com/rsuntk/KernelSU/releases/latest)

```bash
# Install via ADB
adb install KernelSU_Manager.apk
```

### 11.2 Cek Status Root

Setelah reboot, buka **KernelSU Manager**. Kamu akan melihat:
- ✅ **Working** — KernelSU aktif dan berfungsi normal
- ❌ **Not working** — Ada masalah dengan kernel hook

### 11.3 Verifikasi via ADB Shell

```bash
adb shell

# Cek versi kernel
uname -r
# Output contoh: 5.4.xxx-android12-9-...

# Cek apakah KernelSU tersedia
which su
# Output: /system/bin/su (atau lokasi lain)

# Cek versi KernelSU
su -v
```

### 11.4 Test Root Access

```bash
adb shell su -c "id"
# Output yang diharapkan: uid=0(root) gid=0(root) ...
```

---

## 12. Troubleshooting

### ❌ Build Error: "No such file or directory" pada Toolchain

**Penyebab:** Path toolchain salah atau toolchain belum terinstall.

**Solusi:**
```bash
# Verifikasi path toolchain
which clang
ls $HOME/toolchains/clang/bin/clang

# Pastikan PATH sudah di-export dengan benar
echo $PATH
```

---

### ❌ Build Error: "scripts/gcc-version.sh: line XX"

**Penyebab:** GCC cross compiler tidak ditemukan.

**Solusi:**
```bash
# Pastikan GCC aarch64 sudah diinstall dan ada di PATH
export CROSS_COMPILE=aarch64-linux-android-
aarch64-linux-android-gcc --version
```

---

### ❌ Submodule KernelSU Kosong / Error

**Penyebab:** Clone tanpa `--recurse-submodules`.

**Solusi:**
```bash
cd kernel_a135f
git submodule update --init --recursive
git submodule status  # Verifikasi
```

---

### ❌ Kernel Panic / Bootloop Setelah Flash

**Penyebab umum:**
- Defconfig tidak kompatibel dengan firmware
- KernelSU hook bermasalah
- DTB tidak cocok

**Solusi:**
```bash
# 1. Boot ke Recovery (Volume Up + Power)
# 2. Flash kernel stock Samsung untuk recovery

# Download kernel stock dari firmware A135FXXSDEYJ1
# Extract boot.img dari firmware, lalu:
heimdall flash --BOOT boot_stock.img
```

---

### ❌ KernelSU Manager Menampilkan "Not Working"

**Penyebab:** Manual hook tidak terpasang dengan benar, atau `CONFIG_KSU_MANUAL_HOOK` tidak aktif.

**Solusi:**
```bash
# Cek config build
grep "CONFIG_KSU" out/.config

# Pastikan baris berikut ada:
# CONFIG_KSU=y
# CONFIG_KSU_MANUAL_HOOK=y

# Jika tidak ada, aktifkan di defconfig:
echo "CONFIG_KSU=y" >> arch/arm64/configs/exynos3830-a13xnsxx_defconfig
echo "CONFIG_KSU_MANUAL_HOOK=y" >> arch/arm64/configs/exynos3830-a13xnsxx_defconfig

# Rebuild
make O=out ARCH=arm64 CC=clang ... -j$(nproc)
```

---

### ❌ Error "CONFIG_KSU_SUSFS" tidak dikenali

**Penyebab:** Branch `rksu` menggunakan commit KernelSU tertentu yang mungkin belum mendukung susfs.

**Solusi:** Gunakan branch `susfs-rksu-master` dari rsuntk/KernelSU jika ingin fitur susfs, tetapi perhatikan bahwa branch ini tidak selalu diupdate.

---

### 🔍 Melihat Build Log untuk Debug

```bash
# Build dengan logging lengkap
make O=out ... -j$(nproc) 2>&1 | tee build.log

# Cari error di log
grep -i "error:" build.log | head -50
grep -i "warning:" build.log | grep -v "^scripts" | head -20
```

---

## 13. Referensi & Kredit

### Repository Utama
- 🔗 [kucingsakti/android_kernel_a135f (rksu)](https://github.com/kucingsakti/android_kernel_a135f/tree/rksu) — Kernel source utama
- 🔗 [rsuntk/KernelSU](https://github.com/rsuntk/KernelSU) — KernelSU fork yang digunakan
- 🔗 [rsuntk/KernelSU Releases](https://github.com/rsuntk/KernelSU/releases/latest) — Download Manager APK

### Komunitas & Diskusi
- 💬 [RKSU Telegram Group: @rsukrnlsu_grp](https://t.me/rsukrnlsu_grp)
- 💬 [RKSU Telegram Channel: @rsukrnlsu](https://t.me/rsukrnlsu)
- 🌐 [XDA Galaxy A13 SM-A135F Forum](https://xdaforums.com/f/samsung-galaxy-a13.12569/)

### Tools yang Digunakan
- [AnyKernel3](https://github.com/osm0sis/AnyKernel3) — Kernel flashing framework
- [Heimdall](https://github.com/Benjamin-Dobell/Heimdall) — Samsung firmware flashing tool
- [KernelSU](https://kernelsu.org) — Kernel-based root solution

### Kredit
- **tiann** — Penulis asli KernelSU
- **rsuntk** — Fork KernelSU yang mendukung Non-GKI kernel
- **kucingsakti** — Maintainer kernel source ini
- **Samsung** — Kernel source asli berdasarkan Linux kernel

---

<div align="center">

**⚠️ DISCLAIMER**

*Modifikasi kernel dapat membatalkan garansi dan berpotensi merusak perangkat jika dilakukan secara tidak benar. Gunakan panduan ini atas risiko Anda sendiri. Selalu backup data sebelum melakukan modifikasi sistem.*

</div>

---

*Panduan ini dibuat berdasarkan analisis repository `kucingsakti/android_kernel_a135f` branch `rksu` dan dokumentasi rsuntk/KernelSU.*
