# 🔧 Panduan Lengkap Build Kernel Samsung Galaxy A13 5G (SM-A135F)
### Branch: `v1` — Kernel Vanilla / Stock Base

> **Base firmware:** `A135FXXSDEYJ1` (XID Region)  
> **SoC:** Samsung Exynos 3830 (universal3830)  
> **Android:** Android 12 (S) — One UI 4.x  
> **Tipe:** Kernel stock murni tanpa modifikasi root

---

## 📋 Daftar Isi

1. [Tentang Branch v1](#1-tentang-branch-v1)
2. [Perbandingan Branch v1 vs rksu](#2-perbandingan-branch-v1-vs-rksu)
3. [Prasyarat & Dependensi](#3-prasyarat--dependensi)
4. [Setup Lingkungan Build](#4-setup-lingkungan-build)
5. [Kloning Repository](#5-kloning-repository)
6. [Konfigurasi Toolchain](#6-konfigurasi-toolchain)
7. [Memahami Struktur & Build Config](#7-memahami-struktur--build-config)
8. [Proses Build Kernel](#8-proses-build-kernel)
9. [Modifikasi Kernel (Opsional)](#9-modifikasi-kernel-opsional)
10. [Membuat Flashable ZIP](#10-membuat-flashable-zip)
11. [Flashing ke Perangkat](#11-flashing-ke-perangkat)
12. [Menggunakan v1 Sebagai Base untuk Pengembangan](#12-menggunakan-v1-sebagai-base-untuk-pengembangan)
13. [Troubleshooting](#13-troubleshooting)
14. [Referensi & Kredit](#14-referensi--kredit)

---

## 1. Tentang Branch v1

Branch `v1` adalah kernel source **Samsung Galaxy A13 5G (SM-A135F) yang bersih**, langsung dari Samsung Open Source, tanpa tambahan patch root atau submodule eksternal apapun. Branch ini berfungsi sebagai:

- **Kernel stock yang bisa dicompile ulang** — untuk memastikan build environment berjalan benar
- **Base yang stabil** untuk pengembang yang ingin menambahkan patch mereka sendiri secara manual (tanpa KernelSU)
- **Titik awal** sebelum beralih ke branch `rksu` jika ingin memahami apa yang berubah

### Spesifikasi Teknis

| Properti | Detail |
|---|---|
| **Perangkat** | Samsung Galaxy A13 5G (SM-A135F) |
| **SoC** | Samsung Exynos 3830 |
| **Arsitektur** | ARM64 (aarch64) |
| **Versi Android** | Android 12 (S) / One UI 4.x |
| **Versi Kernel** | Linux 5.4.x |
| **KernelSU** | ❌ Tidak ada |
| **Submodule** | ❌ Tidak ada |
| **Jumlah Commit** | 8 commits |
| **Region** | XID (A135FXXSDEYJ1) |
| **Tipe** | Vanilla / Stock |

---

## 2. Perbandingan Branch v1 vs rksu

Memahami perbedaan ini penting agar kamu memilih branch yang tepat:

| Fitur | `v1` | `rksu` |
|---|---|---|
| **KernelSU** | ❌ Tidak ada | ✅ Ada (rsuntk fork) |
| **Git Submodule** | ❌ Tidak ada | ✅ KernelSU sebagai submodule |
| **`.gitmodules`** | ❌ Tidak ada | ✅ Ada |
| **Jumlah Commits** | 8 | 10 |
| **`GUIDE.md`** | ❌ Tidak ada | ✅ Ada |
| **`GUIDE_v1.md`** | ✅ Ada | ✅ Ada (sama) |
| **Clone Command** | `git clone -b v1` | `git clone --recurse-submodules -b rksu` |
| **Cocok untuk** | Base pengembangan, kernel clean | Kernel dengan root KernelSU |
| **Tingkat Kesulitan** | ⭐⭐ Lebih mudah | ⭐⭐⭐ Lebih kompleks |

> **Kapan memilih `v1`?**
> - Kamu ingin memahami source kernel tanpa tambahan apapun
> - Kamu ingin menambahkan patch custom sendiri (bukan KernelSU)
> - Kamu ingin belajar build kernel dari awal tanpa variabel tambahan
> - Kamu ingin kernel yang sedekat mungkin dengan stock Samsung

> **Kapan memilih `rksu`?**
> - Kamu ingin kernel dengan akses root via KernelSU
> - Kamu ingin menggunakan Zygisk Next atau LSPosed via KernelSU

---

## 3. Prasyarat & Dependensi

### Sistem Operasi yang Didukung
**Ubuntu 20.04 LTS** atau **Ubuntu 22.04 LTS** sangat direkomendasikan. Debian dan turunannya juga bisa berjalan, namun Ubuntu paling banyak teruji untuk build kernel Samsung Exynos.

> ⚠️ **Windows tidak didukung secara native.** Gunakan WSL2 (Windows Subsystem for Linux 2) jika di Windows, atau mesin virtual dengan Ubuntu.

### Spesifikasi Hardware

| Komponen | Minimum | Direkomendasikan |
|---|---|---|
| **RAM** | 8 GB | 16 GB atau lebih |
| **CPU** | 4 core | 8 core atau lebih |
| **Storage** | 40 GB | 80 GB (dengan ccache) |
| **Koneksi** | Diperlukan | Broadband stabil |

### Instalasi Paket Sistem

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
    fastboot \
    xz-utils \
    tar
```

### Verifikasi Instalasi

```bash
# Cek versi yang dibutuhkan
git --version        # Minimal 2.x
make --version       # Minimal 4.x
python3 --version    # Minimal 3.6
clang --version      # Minimal 10.x
bc --version
flex --version
bison --version
```

---

## 4. Setup Lingkungan Build

### 4.1 Setup CCache (Sangat Direkomendasikan)

CCache menyimpan hasil kompilasi sebelumnya sehingga rebuild jauh lebih cepat (bisa hingga 5-10x pada rebuild). Sangat penting jika kamu sering mengubah konfigurasi.

```bash
# Install ccache
sudo apt install ccache -y

# Atur ukuran maksimum cache
ccache --max-size=50G

# Cek statistik ccache
ccache --show-stats

# Reset statistik (opsional, untuk tracking baru)
ccache --zero-stats
```

Tambahkan ke `~/.bashrc` agar ccache selalu aktif:

```bash
cat >> ~/.bashrc << 'EOF'

# CCache untuk kernel build
export USE_CCACHE=1
export CCACHE_EXEC=$(which ccache)
export CCACHE_DIR=$HOME/.ccache
EOF

source ~/.bashrc
```

### 4.2 Setup Variabel Lingkungan Global

```bash
cat >> ~/.bashrc << 'EOF'

# Kernel Build Variables
export ARCH=arm64
export SUBARCH=arm64
export ANDROID_MAJOR_VERSION=s
EOF

source ~/.bashrc
```

### 4.3 Aktifkan Fitur Paralel Build

Untuk build yang lebih cepat, kamu bisa meningkatkan batas file descriptor:

```bash
# Tambah ke /etc/security/limits.conf
echo "$(whoami) soft nofile 65536" | sudo tee -a /etc/security/limits.conf
echo "$(whoami) hard nofile 65536" | sudo tee -a /etc/security/limits.conf
```

---

## 5. Kloning Repository

### 5.1 Clone Branch v1 (Sederhana — Tanpa Submodule)

Keunggulan branch `v1` adalah proses clone yang jauh lebih sederhana karena **tidak ada git submodule**. Kamu tidak perlu flag `--recurse-submodules` seperti di branch `rksu`.

```bash
# Clone branch v1 (cara standar, cepat)
git clone -b v1 \
    https://github.com/kucingsakti/android_kernel_a135f.git \
    kernel_a135f_v1

cd kernel_a135f_v1
```

### 5.2 Clone Shallow (Lebih Hemat Bandwidth & Storage)

Jika koneksi atau storage terbatas, gunakan shallow clone:

```bash
# Clone hanya 1 commit terbaru (paling hemat)
git clone --depth=1 -b v1 \
    https://github.com/kucingsakti/android_kernel_a135f.git \
    kernel_a135f_v1

cd kernel_a135f_v1
```

> ⚠️ **Catatan shallow clone:** Dengan `--depth=1` kamu tidak bisa melihat history commit sepenuhnya. Jika nanti butuh history lengkap:
> ```bash
> git fetch --unshallow
> ```

### 5.3 Verifikasi Clone

```bash
# Cek branch yang aktif
git branch
# Output: * v1

# Cek jumlah commit
git log --oneline
# Seharusnya ada 8 commits

# Pastikan TIDAK ada folder KernelSU (ini membedakan v1 dari rksu)
ls KernelSU 2>/dev/null && echo "WARNING: KernelSU ada!" || echo "OK: Tidak ada KernelSU (sesuai ekspektasi v1)"

# Pastikan TIDAK ada .gitmodules
ls .gitmodules 2>/dev/null && echo "WARNING: .gitmodules ada!" || echo "OK: Tidak ada .gitmodules (sesuai ekspektasi v1)"
```

### 5.4 Melihat Perbedaan dengan Branch rksu

Jika ingin tahu secara teknis apa yang ditambahkan di `rksu` dibanding `v1`:

```bash
# Perlu fetch remote rksu terlebih dahulu
git fetch origin rksu:rksu

# Lihat file yang berbeda antara v1 dan rksu
git diff v1 rksu --stat

# Lihat commit yang ada di rksu tapi tidak di v1
git log v1..rksu --oneline
```

---

## 6. Konfigurasi Toolchain

Branch `v1` menggunakan stack compiler yang sama dengan `rksu`: **Clang sebagai compiler utama** dengan GCC sebagai cross compiler pendamping.

### Opsi A: AOSP Clang (Direkomendasikan untuk Kompatibilitas)

Ini adalah pilihan paling stabil karena digunakan Samsung dan Google untuk build kernel Android 12.

```bash
mkdir -p ~/toolchains && cd ~/toolchains

# Download AOSP Clang r416183b (versi untuk Android 12)
wget -O clang-r416183b.tar.gz \
    "https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86/+archive/refs/heads/android12-release/clang-r416183b.tar.gz"

mkdir -p clang-r416183b
tar -xzf clang-r416183b.tar.gz -C clang-r416183b/

# Verifikasi
clang-r416183b/bin/clang --version
```

### Opsi B: ProtonClang (Direkomendasikan untuk Performa Build)

ProtonClang adalah toolchain populer di komunitas custom kernel yang sering memberikan output binary lebih optimal:

```bash
mkdir -p ~/toolchains && cd ~/toolchains

git clone --depth=1 \
    https://github.com/kdrag0n/proton-clang.git \
    proton-clang

# Verifikasi
proton-clang/bin/clang --version
```

### Opsi C: Neutron Clang (Alternatif Modern)

```bash
mkdir -p ~/toolchains/neutron-clang && cd ~/toolchains/neutron-clang

# Download versi terbaru dari neutron-tc releases
bash <(curl -s "https://raw.githubusercontent.com/Neutron-Toolchains/antman/main/antman") -S=latest

bin/clang --version
```

### 6.1 Setup GCC Cross Compiler

Diperlukan untuk beberapa driver dan compat layer 32-bit:

```bash
cd ~/toolchains

# GCC AArch64 (untuk kernel ARM64)
git clone --depth=1 \
    https://github.com/LineageOS/android_prebuilts_gcc_linux-x86_aarch64_aarch64-linux-android-4.9.git \
    gcc-aarch64

# GCC ARM32 (untuk 32-bit compat layer)
git clone --depth=1 \
    https://github.com/LineageOS/android_prebuilts_gcc_linux-x86_arm_arm-linux-androideabi-4.9.git \
    gcc-arm
```

### 6.2 Export PATH Toolchain

```bash
# Sesuaikan path dengan lokasi toolchain yang kamu download
# Contoh menggunakan ProtonClang:
export PATH="$HOME/toolchains/proton-clang/bin:$HOME/toolchains/gcc-aarch64/bin:$HOME/toolchains/gcc-arm/bin:$PATH"

# Verifikasi semua compiler tersedia
which clang         # Harus menunjuk ke proton-clang/bin/clang
which aarch64-linux-android-gcc
which arm-linux-androideabi-gcc
```

### 6.3 Membuat Script Wrapper Toolchain (Opsional tapi Praktis)

Buat file `~/setup_toolchain.sh` agar tidak perlu export ulang setiap session:

```bash
cat > ~/setup_toolchain.sh << 'EOF'
#!/bin/bash
# Setup toolchain untuk build kernel A135F

TOOLCHAIN_DIR="$HOME/toolchains"

# Pilih salah satu (comment/uncomment sesuai yang diinstall)
# CLANG_DIR="$TOOLCHAIN_DIR/clang-r416183b"     # AOSP Clang
CLANG_DIR="$TOOLCHAIN_DIR/proton-clang"          # ProtonClang
# CLANG_DIR="$TOOLCHAIN_DIR/neutron-clang"       # Neutron Clang

GCC64_DIR="$TOOLCHAIN_DIR/gcc-aarch64"
GCC32_DIR="$TOOLCHAIN_DIR/gcc-arm"

export PATH="$CLANG_DIR/bin:$GCC64_DIR/bin:$GCC32_DIR/bin:$PATH"
export ARCH=arm64
export SUBARCH=arm64
export ANDROID_MAJOR_VERSION=s
export USE_CCACHE=1
export CCACHE_EXEC=$(which ccache)

echo "✅ Toolchain siap:"
echo "   Clang  : $(clang --version | head -1)"
echo "   GCC64  : $(aarch64-linux-android-gcc --version | head -1)"
echo "   GCC32  : $(arm-linux-androideabi-gcc --version | head -1)"
EOF

chmod +x ~/setup_toolchain.sh

# Gunakan dengan:
# source ~/setup_toolchain.sh
```

---

## 7. Memahami Struktur & Build Config

### 7.1 Struktur Direktori Penting

```
kernel_a135f_v1/
├── arch/
│   └── arm64/
│       ├── configs/                    ← Defconfig ada di sini
│       │   └── exynos3830-a13xnsxx_defconfig  ← Config utama A135F
│       └── boot/                       ← Output kernel image (setelah build)
├── drivers/                            ← Driver hardware (display, camera, dll)
├── scripts/                            ← Script build helper
├── build.config.universal3830_s        ← Build config utama untuk Android S
├── build_kernel.sh                     ← Script build otomatis
├── GUIDE_v1.md                         ← Guide asli (lebih singkat)
├── AndroidKernel.mk                    ← Makefile untuk integrasi AOSP
└── Makefile                            ← Makefile utama kernel
```

### 7.2 File Build Config yang Relevan untuk A135F

| File Config | Kegunaan |
|---|---|
| `build.config.universal3830_s` | **Config utama** — Exynos 3830, Android 12 (S) |
| `build.config.universal3830_stu_mr` | Android S + maintenance release |
| `build.config.universal3830_st_mr` | Android S trunk + MR |
| `build.config.universal3830` | Config generik 3830 |
| `build.config.erd3830` | ERD (Engineering Reference Design) board |
| `build.config.common` | Config base yang di-include semua config |

Untuk SM-A135F dengan firmware `SDEYJ1` (Android 12), gunakan `build.config.universal3830_s`.

### 7.3 Defconfig Kernel

Defconfig mendefinisikan semua opsi CONFIG_* yang akan dikompilasi. Untuk A135F:

```bash
# Lihat defconfig
cat arch/arm64/configs/exynos3830-a13xnsxx_defconfig

# Cari CONFIG tertentu
grep "CONFIG_ZRAM" arch/arm64/configs/exynos3830-a13xnsxx_defconfig
grep "CONFIG_CPU_FREQ" arch/arm64/configs/exynos3830-a13xnsxx_defconfig
```

### 7.4 Perbedaan Signifikan v1 vs rksu di Level Source

Branch `v1` **tidak memiliki** file-file berikut yang ada di `rksu`:
- `KernelSU/` — direktori submodule KernelSU
- `.gitmodules` — definisi submodule
- `GUIDE.md` — guide utama (hanya ada `GUIDE_v1.md`)
- Patch KernelSU di `drivers/` atau `security/`
- Modifikasi pada `security/Kconfig` dan `Makefile` untuk KSU

---

## 8. Proses Build Kernel

### 8.1 Menggunakan Script `build_kernel.sh` (Cara Termudah)

```bash
cd kernel_a135f_v1

# Aktifkan toolchain terlebih dahulu
source ~/setup_toolchain.sh

# Beri izin eksekusi pada script
chmod +x build_kernel.sh

# Lihat isi script untuk memastikan konfigurasi
cat build_kernel.sh

# Jalankan build
./build_kernel.sh
```

### 8.2 Build Manual Langkah per Langkah

Cara ini memberikan kontrol penuh atas setiap tahap build:

```bash
cd kernel_a135f_v1

# ── Step 1: Aktifkan toolchain ──────────────────────────────────────────
source ~/setup_toolchain.sh

# ── Step 2: Set variabel build ──────────────────────────────────────────
export ARCH=arm64
export SUBARCH=arm64
export ANDROID_MAJOR_VERSION=s

CLANG_PATH="$HOME/toolchains/proton-clang/bin"      # Sesuaikan
GCC64_PATH="$HOME/toolchains/gcc-aarch64/bin"
GCC32_PATH="$HOME/toolchains/gcc-arm/bin"
export PATH="$CLANG_PATH:$GCC64_PATH:$GCC32_PATH:$PATH"

THREADS=$(nproc --all)
echo "Menggunakan $THREADS thread"

# ── Step 3: Buat direktori output ───────────────────────────────────────
mkdir -p out

# ── Step 4: Generate Defconfig ──────────────────────────────────────────
make O=out \
     ARCH=arm64 \
     CC=clang \
     CLANG_TRIPLE=aarch64-linux-gnu- \
     CROSS_COMPILE=aarch64-linux-android- \
     CROSS_COMPILE_ARM32=arm-linux-androideabi- \
     exynos3830-a13xnsxx_defconfig

echo "✅ Defconfig berhasil di-generate"
echo "   File konfigurasi: out/.config"
```

### 8.3 (Opsional) Modifikasi Konfigurasi Sebelum Build

Setelah defconfig di-generate, kamu bisa memodifikasi konfigurasi:

```bash
# Cara 1: GUI interaktif (ncurses) — paling mudah untuk eksplorasi
make O=out ARCH=arm64 menuconfig

# Cara 2: GUI berbasis Qt (jika Qt terinstall)
make O=out ARCH=arm64 xconfig

# Cara 3: Edit langsung via script (untuk perubahan spesifik)
# Mengaktifkan opsi:
scripts/config --file out/.config --enable CONFIG_ZRAM
scripts/config --file out/.config --set-val CONFIG_ZRAM_DEF_COMP_LZ4 y

# Menonaktifkan opsi:
scripts/config --file out/.config --disable CONFIG_LOCALVERSION_AUTO

# Setelah edit manual, perbaiki dependensi:
make O=out ARCH=arm64 olddefconfig
```

### 8.4 Build Kernel Utama

```bash
# Build dengan semua thread yang tersedia
make O=out \
     ARCH=arm64 \
     CC=clang \
     CLANG_TRIPLE=aarch64-linux-gnu- \
     CROSS_COMPILE=aarch64-linux-android- \
     CROSS_COMPILE_ARM32=arm-linux-androideabi- \
     LD=ld.lld \
     AR=llvm-ar \
     NM=llvm-nm \
     OBJCOPY=llvm-objcopy \
     OBJDUMP=llvm-objdump \
     STRIP=llvm-strip \
     -j$THREADS 2>&1 | tee build.log

# Cek apakah build berhasil
if [ $? -eq 0 ]; then
    echo "✅ Build BERHASIL!"
else
    echo "❌ Build GAGAL! Cek build.log untuk detail error."
    grep -i "error:" build.log | tail -20
fi
```

### 8.5 Verifikasi Output Build

```bash
# Cek file output yang dihasilkan
ls -lh out/arch/arm64/boot/

# File yang seharusnya ada:
# Image            ← Kernel image uncompressed
# Image.gz         ← Kernel image compressed (gzip)
# Image.gz-dtb     ← Kernel + DTB (yang biasanya diflash ke Samsung)
# dts/             ← Device Tree blobs

# Cek ukuran file (Image.gz-dtb biasanya 10-20 MB)
du -sh out/arch/arm64/boot/Image.gz-dtb

# Verifikasi kernel image valid
file out/arch/arm64/boot/Image
# Output: Linux kernel ARM64 boot executable Image, ...
```

### 8.6 Melihat Informasi Build dari Kernel

```bash
# Cek string versi kernel yang akan dihasilkan
cat out/include/generated/utsrelease.h

# Cek konfigurasi akhir
grep "CONFIG_LOCALVERSION" out/.config
grep "CONFIG_ZRAM" out/.config
```

---

## 9. Modifikasi Kernel (Opsional)

Karena `v1` adalah base yang bersih, ini adalah tempat yang tepat untuk belajar menambahkan patch. Beberapa modifikasi umum:

### 9.1 Mengubah Nama Versi Kernel (Custom Kernel String)

```bash
# Edit defconfig
nano arch/arm64/configs/exynos3830-a13xnsxx_defconfig

# Cari dan ubah:
# CONFIG_LOCALVERSION="-a13xnsxx"
# Ganti menjadi, contoh:
# CONFIG_LOCALVERSION="-CustomKernel-v1"
```

### 9.2 Menambahkan CPU Governor

Kernel Samsung A135F secara default hanya menyertakan beberapa governor. Kamu bisa mengaktifkan lebih banyak:

```bash
# Edit .config setelah generate defconfig:
scripts/config --file out/.config --enable CONFIG_CPU_FREQ_GOV_CONSERVATIVE
scripts/config --file out/.config --enable CONFIG_CPU_FREQ_GOV_ONDEMAND
scripts/config --file out/.config --enable CONFIG_CPU_FREQ_GOV_USERSPACE
scripts/config --file out/.config --enable CONFIG_CPU_FREQ_GOV_SCHEDUTIL

make O=out ARCH=arm64 olddefconfig
```

### 9.3 Mengaktifkan ZRAM dengan LZ4

```bash
scripts/config --file out/.config --enable CONFIG_ZRAM
scripts/config --file out/.config --enable CONFIG_ZRAM_DEF_COMP_LZ4
scripts/config --file out/.config --set-str CONFIG_ZRAM_DEF_COMP "lz4"

make O=out ARCH=arm64 olddefconfig
```

### 9.4 Menerapkan Patch Eksternal

```bash
# Format: patch -p1 < nama_patch.patch
# Contoh menerapkan patch dari file:
patch -p1 < path/to/your/patch.patch

# Atau menggunakan git am untuk patch dari email format:
git am path/to/patch.patch

# Setelah apply patch, rebuild:
make O=out ARCH=arm64 CC=clang ... -j$THREADS
```

### 9.5 Menyimpan Konfigurasi yang Sudah Dimodifikasi

```bash
# Simpan config yang sudah dimodifikasi kembali ke defconfig
make O=out ARCH=arm64 savedefconfig

# Copy ke lokasi defconfig resmi
cp out/defconfig arch/arm64/configs/exynos3830-a13xnsxx_defconfig
```

---

## 10. Membuat Flashable ZIP

### 10.1 Clone AnyKernel3

```bash
cd ~
git clone https://github.com/osm0sis/AnyKernel3.git AnyKernel3_a135f_v1
cd AnyKernel3_a135f_v1
```

### 10.2 Konfigurasi `anykernel.sh` untuk SM-A135F

```bash
cat > anykernel.sh << 'EOF'
# AnyKernel3 Ramdisk Mod Script
# osm0sis @ xda-developers

## AnyKernel setup
properties() { '
kernel.string=Stock-Base Kernel v1 for Samsung Galaxy A13 5G (SM-A135F)
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

### 10.3 Membersihkan File Contoh AnyKernel3

```bash
# Hapus kernel image contoh yang ada di AnyKernel3
rm -f Image Image.gz Image.gz-dtb zImage

# Hapus README contoh jika tidak diperlukan
# rm -f README.md
```

### 10.4 Copy Kernel Image

```bash
# Copy dari output build
cp ~/kernel_a135f_v1/out/arch/arm64/boot/Image.gz-dtb ~/AnyKernel3_a135f_v1/

# Verifikasi ukuran (harus non-zero)
ls -lh ~/AnyKernel3_a135f_v1/Image.gz-dtb
```

### 10.5 Buat ZIP

```bash
cd ~/AnyKernel3_a135f_v1

# Nama ZIP dengan timestamp
ZIP_NAME="Stock_Base_Kernel_v1_A135F_$(date +%Y%m%d_%H%M%S).zip"

# Buat ZIP (exclude file yang tidak perlu)
zip -r9 "$ZIP_NAME" . \
    -x "*.git*" \
    -x "*.zip" \
    -x "*.md" \
    -x "LICENSE"

echo "✅ ZIP berhasil dibuat: $ZIP_NAME"
ls -lh "$ZIP_NAME"
```

### 10.6 (Opsional) Verifikasi Isi ZIP Sebelum Flash

```bash
# Pastikan ZIP berisi file yang benar
unzip -l "$ZIP_NAME" | grep -E "(Image|anykernel|tools)"
```

---

## 11. Flashing ke Perangkat

### Prasyarat
- ✅ Custom Recovery terpasang (TWRP direkomendasikan)
- ✅ Firmware Samsung One UI 4.x / Android 12 (versi SDEYJ1 atau kompatibel)
- ✅ Data sudah di-backup
- ✅ Baterai minimal 50%
- ✅ Developer Options & OEM Unlock aktif (untuk opsi Heimdall)

### 11.1 Transfer ZIP ke Perangkat

```bash
# Via ADB
adb push ~/AnyKernel3_a135f_v1/Stock_Base_Kernel_v1_A135F_*.zip /sdcard/

# Verifikasi berhasil
adb shell ls -lh /sdcard/Stock_Base_Kernel_v1_A135F_*.zip
```

### 11.2 Masuk ke Recovery Mode

```bash
# Via ADB (perangkat harus dalam kondisi booted dan ADB enabled)
adb reboot recovery

# Atau cara manual (matikan perangkat terlebih dahulu):
# Tekan dan tahan Volume Up + Power secara bersamaan
# Lepas saat logo Samsung muncul
```

### 11.3 Flash via TWRP

1. Di layar utama TWRP, tap **Install**
2. Navigasi ke `/sdcard/`
3. Pilih file ZIP kernel yang sudah ditransfer
4. **Swipe to Confirm Flash**
5. Tunggu hingga proses selesai (biasanya < 30 detik)
6. Tap **Reboot System**

### 11.4 Flash via Heimdall (Tanpa Custom Recovery)

Heimdall memungkinkan flash langsung ke partisi boot dari mode Download (tanpa perlu recovery):

```bash
# Install Heimdall
sudo apt install heimdall-flash -y

# Verifikasi
heimdall version

# Boot perangkat ke Download Mode:
# Matikan perangkat → Tahan Volume Down + Power → Tahan Volume Up untuk lanjut

# Deteksi perangkat
heimdall detect

# Flash kernel image langsung ke partisi BOOT
heimdall flash --BOOT ~/kernel_a135f_v1/out/arch/arm64/boot/Image.gz-dtb --no-reboot

# Reboot manual setelah flash selesai
```

> ⚠️ **Catatan Heimdall:** Pastikan driver USB Samsung (untuk Windows/WSL) atau `udev rules` (untuk Linux) sudah terkonfigurasi sebelum menggunakan Heimdall.

### 11.5 Setup udev Rules untuk Heimdall (Linux)

```bash
# Tambah rule udev untuk Samsung Download Mode
echo 'SUBSYSTEM=="usb", ATTRS{idVendor}=="04e8", MODE="0666", GROUP="plugdev"' | \
    sudo tee /etc/udev/rules.d/51-samsung.rules

sudo udevadm control --reload-rules
sudo udevadm trigger

# Tambah user ke grup plugdev
sudo usermod -aG plugdev $USER
# (logout dan login kembali agar efektif)
```

---

## 12. Menggunakan v1 Sebagai Base untuk Pengembangan

Branch `v1` adalah titik awal yang ideal untuk pengembangan kernel lanjutan. Berikut beberapa skenario penggunaan:

### 12.1 Menambahkan KernelSU Secara Manual (Dari v1 ke Setara rksu)

Jika kamu ingin memahami proses integrasi KernelSU dari awal:

```bash
cd kernel_a135f_v1

# Tambahkan KernelSU sebagai submodule (metode rsuntk)
curl -LSs "https://raw.githubusercontent.com/rsuntk/KernelSU/main/kernel/setup.sh" | bash -s main

# Verifikasi KernelSU telah ditambahkan
ls KernelSU/
grep "CONFIG_KSU" arch/arm64/configs/exynos3830-a13xnsxx_defconfig
```

### 12.2 Membuat Branch Sendiri dari v1

```bash
cd kernel_a135f_v1

# Buat branch baru untuk modifikasi kamu
git checkout -b my-custom-kernel

# Lakukan modifikasi...
# Commit perubahan
git add -A
git commit -m "Add custom modifications"

# Lihat diff dengan v1 original
git diff v1..my-custom-kernel --stat
```

### 12.3 Membuat Patch dari Modifikasi

```bash
# Generate patch dari perubahan yang ada
git format-patch v1..my-custom-kernel --output-directory patches/

# Lihat patch yang dibuat
ls patches/
cat patches/0001-*.patch
```

### 12.4 Menyinkronkan dengan Upstream Samsung

Jika Samsung merilis update kernel source, kamu bisa merge perubahannya:

```bash
# Tambah remote Samsung (jika tersedia)
git remote add samsung-upstream https://github.com/samsung-upstream/kernel-a135f.git
git fetch samsung-upstream

# Lihat perubahan sebelum merge
git log HEAD..samsung-upstream/main --oneline

# Merge (hati-hati, mungkin ada conflict)
git merge samsung-upstream/main
```

---

## 13. Troubleshooting

### ❌ Error: `fatal: could not read Username`

**Penyebab:** Masalah autentikasi GitHub saat clone.

**Solusi:**
```bash
# Gunakan HTTPS dengan credential helper
git config --global credential.helper store

# Atau gunakan SSH key
# git clone git@github.com:kucingsakti/android_kernel_a135f.git -b v1
```

---

### ❌ Error: `make: clang: No such file or directory`

**Penyebab:** Clang tidak ditemukan di PATH.

**Solusi:**
```bash
# Cek lokasi clang
find ~/toolchains -name "clang" -type f 2>/dev/null

# Export PATH yang benar
export PATH="$HOME/toolchains/proton-clang/bin:$PATH"

# Verifikasi
which clang
clang --version
```

---

### ❌ Error: `aarch64-linux-android-gcc: command not found`

**Penyebab:** GCC cross compiler tidak di PATH.

**Solusi:**
```bash
# Cek lokasi gcc
find ~/toolchains -name "aarch64-linux-android-gcc" -type f 2>/dev/null

# Export path yang benar
export PATH="$HOME/toolchains/gcc-aarch64/bin:$PATH"

# Verifikasi
aarch64-linux-android-gcc --version
```

---

### ❌ Error: `scripts/dtc/dtc: error while loading shared libraries`

**Penyebab:** Library sistem kurang, atau dtc tidak kompatibel.

**Solusi:**
```bash
# Install dtc dari sistem
sudo apt install device-tree-compiler -y

# Atau build dtc dari source yang ada di kernel
make O=out ARCH=arm64 CC=clang scripts
```

---

### ❌ Build Error: `error: implicit declaration of function 'xyz'`

**Penyebab:** Header file tidak di-include, atau fungsi deprecated di versi Clang baru.

**Solusi:**
```bash
# Coba dengan versi Clang yang lebih lama (misalnya AOSP r383902b)
# Atau tambahkan flag:
make O=out ARCH=arm64 CC=clang \
     KCFLAGS="-Wno-implicit-function-declaration" \
     ... -j$THREADS
```

---

### ❌ Kernel Boots tapi Perangkat Mati Mendadak (Reboot Loop)

**Penyebab umum:**
- Defconfig mengaktifkan fitur yang tidak kompatibel dengan firmware
- DTB tidak cocok dengan partisi vendor

**Solusi:**
```bash
# 1. Boot ke recovery
# 2. Flash kernel stock Samsung untuk recovery:

# Download firmware A135FXXSDEYJ1 dari SamFw atau SamMobile
# Extract boot.img.lz4 dari AP_A135FXXSDEYJ1_*.tar.md5
# Decompress:
lz4 -d boot.img.lz4 boot.img

# Flash via Heimdall:
heimdall flash --BOOT boot.img

# 3. Cek dmesg untuk clue setelah berhasil boot dengan stock:
adb logcat -b kernel | grep -E "(error|fault|panic)"
```

---

### ❌ Image.gz-dtb Tidak Muncul Setelah Build

**Penyebab:** Build selesai tapi hanya menghasilkan `Image` tanpa DTB.

**Solusi:**
```bash
# Build DTB secara eksplisit
make O=out ARCH=arm64 CC=clang ... dtbs -j$THREADS

# Build Image.gz-dtb secara eksplisit
make O=out ARCH=arm64 CC=clang ... Image.gz-dtb -j$THREADS

# Atau cek apakah CONFIG_BUILD_ARM64_APPENDED_DTB_IMAGE aktif
grep "CONFIG_BUILD_ARM64_APPENDED_DTB" out/.config
```

---

### 🔍 Cara Membaca Build Log untuk Debug Lebih Dalam

```bash
# Tampilkan semua error
grep -n "error:" build.log

# Tampilkan error beserta konteksnya (5 baris sebelum dan sesudah)
grep -n -A5 -B5 "error:" build.log | head -100

# Cari file yang gagal dikompilasi
grep "^  CC\|^  LD\|^  AR" build.log | tail -20

# Estimasi waktu build (dari timestamp log)
head -1 build.log
tail -1 build.log
```

---

## 14. Referensi & Kredit

### Repository

| Link | Keterangan |
|---|---|
| [kucingsakti/android_kernel_a135f (v1)](https://github.com/kucingsakti/android_kernel_a135f/tree/v1) | Repository kernel source ini (branch v1) |
| [kucingsakti/android_kernel_a135f (rksu)](https://github.com/kucingsakti/android_kernel_a135f/tree/rksu) | Branch dengan KernelSU |
| [osm0sis/AnyKernel3](https://github.com/osm0sis/AnyKernel3) | Framework untuk membuat flashable ZIP kernel |
| [rsuntk/KernelSU](https://github.com/rsuntk/KernelSU) | KernelSU fork untuk Non-GKI (jika ingin upgrade ke rksu) |

### Komunitas & Forum

| Link | Keterangan |
|---|---|
| [XDA — Galaxy A13 SM-A135F Custom Kernel](https://xdaforums.com/t/galaxy-a13-sm-a135f-and-sm-a137f-custom-kernel.4440617/) | Thread XDA untuk kernel A135F |
| [XDA Galaxy A13 Forum](https://xdaforums.com/f/samsung-galaxy-a13.12569/) | Forum umum Galaxy A13 |
| [SamMobile Firmware](https://www.sammobile.com/samsung/galaxy-a13/firmware/SM-A135F/) | Download firmware Samsung A135F |
| [SamFw](https://samfw.com/firmware/SM-A135F) | Alternatif download firmware |

### Tools

| Tool | Kegunaan |
|---|---|
| [Heimdall](https://github.com/Benjamin-Dobell/Heimdall) | Flash firmware Samsung dari Linux/Mac |
| [TWRP](https://twrp.me) | Custom recovery untuk flashing |
| [7-Zip / Zarchiver](https://www.7-zip.org) | Ekstrak file firmware .tar.md5 |

### Kredit
- **Samsung** — Kernel source asli berdasarkan Linux kernel GPL
- **kucingsakti** — Maintainer repository ini
- **physwizz** — Pionir custom kernel untuk Galaxy A13 SM-A135F di komunitas XDA
- **osm0sis** — Penulis AnyKernel3

---

<div align="center">

**⚠️ DISCLAIMER**

*Branch `v1` adalah kernel stock Samsung yang dicompile ulang. Meski lebih aman dari branch yang membawa patch eksternal, modifikasi tetap dapat membatalkan garansi dan berpotensi merusak perangkat jika dilakukan secara tidak benar. Selalu backup data sebelum melakukan modifikasi sistem apapun. Gunakan panduan ini atas risiko Anda sendiri.*

</div>

---

*Panduan ini dibuat berdasarkan analisis repository `kucingsakti/android_kernel_a135f` branch `v1`, dibandingkan dengan branch `rksu`, serta referensi dari komunitas XDA Galaxy A13.*
