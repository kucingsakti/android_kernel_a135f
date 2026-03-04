# 🔧 Panduan Lengkap Build Kernel Samsung Galaxy A13 5G (SM-A135F)
### Branch: `v1` — Kernel Vanilla / Stock Base

> **Base firmware:** `A135FXXSDEYJ1` (XID Region)  
> **SoC:** Samsung Exynos 3830 (universal3830)  
> **Android:** Android 12 (S) — One UI 4.x  
> **Tipe:** Kernel stock murni tanpa modifikasi root  
> **Sumber build:** Berdasarkan `README_Kernel.txt` resmi Samsung

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
9. [Output Build](#9-output-build)
10. [Modifikasi Kernel (Opsional)](#10-modifikasi-kernel-opsional)
11. [Membuat Flashable ZIP](#11-membuat-flashable-zip)
12. [Flashing ke Perangkat](#12-flashing-ke-perangkat)
13. [Menggunakan v1 Sebagai Base Pengembangan](#13-menggunakan-v1-sebagai-base-pengembangan)
14. [Troubleshooting](#14-troubleshooting)
15. [Referensi & Kredit](#15-referensi--kredit)

---

## 1. Tentang Branch v1

Branch `v1` adalah kernel source **Samsung Galaxy A13 5G (SM-A135F) yang bersih**, langsung dari Samsung Open Source, tanpa tambahan patch root atau submodule eksternal apapun. Branch ini berfungsi sebagai:

- **Kernel stock yang bisa dicompile ulang** — titik validasi bahwa build environment sudah benar
- **Base yang stabil** untuk pengembang yang ingin menambahkan patch custom sendiri
- **Titik awal belajar** sebelum beralih ke branch `rksu` atau eksplorasi modifikasi lanjutan

### Spesifikasi Teknis

| Properti | Detail |
|---|---|
| **Perangkat** | Samsung Galaxy A13 5G (SM-A135F) |
| **SoC** | Samsung Exynos 3830 |
| **Arsitektur** | ARM64 (aarch64) |
| **Versi Android** | Android 12 (S) / One UI 4.x |
| **Versi Kernel** | Linux 5.4.x |
| **Defconfig** | `exynos850-a13xx_defconfig` |
| **Clang Resmi** | `clang-r383902` |
| **KernelSU** | ❌ Tidak ada |
| **Submodule** | ❌ Tidak ada |
| **Jumlah Commit** | 8 commits |
| **Region** | XID (A135FXXSDEYJ1) |
| **Tipe** | Vanilla / Stock |

---

## 2. Perbandingan Branch v1 vs rksu

| Fitur | `v1` | `rksu` |
|---|---|---|
| **KernelSU** | ❌ Tidak ada | ✅ Ada (rsuntk fork) |
| **Git Submodule** | ❌ Tidak ada | ✅ KernelSU sebagai submodule |
| **`.gitmodules`** | ❌ Tidak ada | ✅ Ada |
| **Jumlah Commits** | 8 | 10 |
| **`GUIDE.md`** | ❌ Tidak ada | ✅ Ada |
| **`GUIDE_v1.md`** | ✅ Ada | ✅ Ada |
| **Clone Command** | `git clone -b v1` | `git clone --recurse-submodules -b rksu` |
| **Cocok untuk** | Base pengembangan, kernel bersih | Kernel dengan root KernelSU |
| **Tingkat Kesulitan** | ⭐⭐ Lebih mudah | ⭐⭐⭐ Lebih kompleks |

> **Pilih `v1`** jika kamu ingin belajar build kernel, menambahkan patch sendiri, atau menginginkan kernel sedekat mungkin dengan stock Samsung.

> **Pilih `rksu`** jika kamu ingin kernel dengan akses root via KernelSU dan ekosistem Zygisk Next / LSPosed.

---

## 3. Prasyarat & Dependensi

### Sistem Operasi
**Ubuntu 20.04 LTS** atau **Ubuntu 22.04 LTS** sangat direkomendasikan.

> ⚠️ **Windows tidak didukung secara native.** Gunakan WSL2 atau mesin virtual Ubuntu.

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
    git curl wget zip unzip bc bison flex \
    libssl-dev libelf-dev make gcc g++ \
    python3 python3-pip cpio kmod \
    build-essential libncurses5-dev libncursesw5-dev \
    rsync ccache lld llvm lz4 zstd \
    device-tree-compiler adb fastboot \
    xz-utils tar
```

### Verifikasi Instalasi

```bash
git --version        # Minimal 2.x
make --version       # Minimal 4.x
python3 --version    # Minimal 3.6
bc --version
flex --version
bison --version
```

---

## 4. Setup Lingkungan Build

### 4.1 Setup CCache (Sangat Direkomendasikan)

CCache menyimpan hasil kompilasi sebelumnya sehingga rebuild jauh lebih cepat (5–10x lebih cepat pada build ulang).

```bash
sudo apt install ccache -y
ccache --max-size=50G
ccache --show-stats
```

Tambahkan ke `~/.bashrc` agar otomatis aktif:

```bash
cat >> ~/.bashrc << 'EOF'

# CCache untuk kernel build
export USE_CCACHE=1
export CCACHE_EXEC=$(which ccache)
export CCACHE_DIR=$HOME/.ccache
EOF

source ~/.bashrc
```

### 4.2 Tingkatkan Batas File Descriptor

```bash
echo "$(whoami) soft nofile 65536" | sudo tee -a /etc/security/limits.conf
echo "$(whoami) hard nofile 65536" | sudo tee -a /etc/security/limits.conf
```

---

## 5. Kloning Repository

### 5.1 Clone Branch v1

Branch `v1` tidak memiliki submodule sehingga proses clone lebih sederhana:

```bash
git clone -b v1 \
    https://github.com/kucingsakti/android_kernel_a135f.git \
    kernel_a135f_v1

cd kernel_a135f_v1
```

### 5.2 Clone Shallow (Hemat Bandwidth)

```bash
git clone --depth=1 -b v1 \
    https://github.com/kucingsakti/android_kernel_a135f.git \
    kernel_a135f_v1

cd kernel_a135f_v1
```

> Untuk restore history penuh: `git fetch --unshallow`

### 5.3 Verifikasi Clone

```bash
git branch                   # Output: * v1
git log --oneline | wc -l    # Seharusnya 8 commits

# Pastikan TIDAK ada KernelSU dan .gitmodules (ciri khas v1)
ls KernelSU   2>/dev/null || echo "OK: Tidak ada KernelSU"
ls .gitmodules 2>/dev/null || echo "OK: Tidak ada .gitmodules"
```

### 5.4 Lihat Perbedaan dengan Branch rksu (Opsional)

```bash
git fetch origin rksu:rksu
git diff v1 rksu --stat
git log v1..rksu --oneline
```

---

## 6. Konfigurasi Toolchain

Berdasarkan `README_Kernel.txt` resmi Samsung, ada **tiga komponen toolchain** yang harus dikonfigurasi:

| Variabel | Nilai (dari README resmi) | Keterangan |
|---|---|---|
| `CROSS_COMPILE` | `.../aarch64-linux-android-4.9/bin/aarch64-linux-android-` | GCC cross-compiler AArch64 |
| `CC` | `.../clang-r383902/bin/clang` | Clang compiler utama |
| `CLANG_TRIPLE` | `.../clang-r383902/bin/aarch64-linux-gnu-` | Prefix binutils di dir bin Clang |

> 💡 **Catatan penting:** `CLANG_TRIPLE` **bukan** toolchain terpisah. Ini adalah path prefix yang mengarah ke file-file `aarch64-linux-gnu-*` yang ada **di dalam direktori `bin/` Clang itu sendiri**.

### 6.1 Download Clang r383902

Ini adalah versi Clang yang secara eksplisit disebutkan di README resmi Samsung untuk kernel ini.

```bash
mkdir -p ~/toolchains && cd ~/toolchains

# Opsi A: Clone dari mirror (cara tercepat)
git clone --depth=1 \
    https://github.com/rsuntk/android_prebuilts_clang_host_linux-x86_clang-r383902b.git \
    clang-r383902

# Opsi B: Download archive dari AOSP
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

### 6.2 Download GCC AArch64 4.9

```bash
cd ~/toolchains

git clone --depth=1 \
    https://github.com/LineageOS/android_prebuilts_gcc_linux-x86_aarch64_aarch64-linux-android-4.9.git \
    gcc-aarch64
```

Verifikasi:
```bash
~/toolchains/gcc-aarch64/bin/aarch64-linux-android-gcc --version
# Output: aarch64-linux-android-gcc (GCC) 4.9.x ...
```

### 6.3 Buat Script Wrapper Toolchain

```bash
cat > ~/setup_toolchain_a135f.sh << 'EOF'
#!/bin/bash
# Setup toolchain resmi Samsung untuk build kernel A135F (v1)

CLANG_DIR="$HOME/toolchains/clang-r383902"
GCC64_DIR="$HOME/toolchains/gcc-aarch64"

# Validasi
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
echo "   Clang             : $(clang --version | head -1)"
echo "   GCC64             : $(aarch64-linux-android-gcc --version | head -1)"
echo "   PLATFORM_VERSION  : $PLATFORM_VERSION"
echo "   ANDROID_MAJOR_VERSION: $ANDROID_MAJOR_VERSION"
echo "   ARCH              : $ARCH"
EOF

chmod +x ~/setup_toolchain_a135f.sh
```

Gunakan dengan: `source ~/setup_toolchain_a135f.sh`

---

## 7. Memahami Struktur & Build Config

### 7.1 Defconfig Resmi Samsung

Berdasarkan `README_Kernel.txt` resmi:

```
arch/arm64/configs/exynos850-a13xx_defconfig
```

> ⚠️ **Perhatian:** Nama defconfig ini adalah `exynos850-a13xx_defconfig`, sesuai README Samsung. Beberapa referensi online menyebutkan nama yang berbeda — selalu prioritaskan README resmi.

### 7.2 Struktur Direktori Penting

```
kernel_a135f_v1/
├── arch/
│   └── arm64/
│       ├── configs/
│       │   └── exynos850-a13xx_defconfig    ← Defconfig resmi Samsung
│       └── boot/
│           └── Image                         ← Output kernel image
├── drivers/
│   └── */*.ko                                ← Output modul kernel
├── Makefile                                  ← Edit path toolchain di sini (Metode A)
├── build.config.universal3830_s              ← Build config untuk Android S
├── build_kernel.sh                           ← Script build otomatis
└── GUIDE_v1.md                               ← Guide asli Samsung (singkat)
```

### 7.3 Build Config yang Relevan untuk A135F

| File | Kegunaan |
|---|---|
| `build.config.universal3830_s` | **Config utama** — Exynos 3830, Android 12 (S) |
| `build.config.universal3830_stu_mr` | Android S + maintenance release |
| `build.config.universal3830_st_mr` | Android S trunk + MR |
| `build.config.erd3830` | ERD (Engineering Reference Design) board |
| `build.config.common` | Config base yang di-include semua config |

---

## 8. Proses Build Kernel

README resmi Samsung mendokumentasikan dua hal yang perlu disiapkan: edit Makefile dan jalankan perintah build. Berikut kedua metode tersebut.

---

### ⭐ Metode A: Edit Makefile (Cara Resmi Samsung)

Ini adalah metode yang **secara eksplisit dideskripsikan di `README_Kernel.txt`** Samsung.

**Langkah 1 — Edit Makefile di root source:**

```bash
cd kernel_a135f_v1
nano Makefile   # atau gunakan editor pilihan kamu
```

Cari dan ubah tiga baris berikut (biasanya di dekat bagian awal Makefile, setelah header lisensi):

```makefile
# Ubah CROSS_COMPILE ke path GCC aarch64:
CROSS_COMPILE = /home/YOUR_USER/toolchains/gcc-aarch64/bin/aarch64-linux-android-

# Ubah CC ke path Clang r383902:
CC = /home/YOUR_USER/toolchains/clang-r383902/bin/clang

# Ubah CLANG_TRIPLE ke prefix di direktori bin Clang yang sama:
CLANG_TRIPLE = /home/YOUR_USER/toolchains/clang-r383902/bin/aarch64-linux-gnu-
```

> Ganti `YOUR_USER` dengan username kamu. Gunakan `echo $HOME` untuk mendapatkan path lengkap.

**Langkah 2 — Export variabel & build (sesuai README Samsung):**

```bash
export PLATFORM_VERSION=12
export ANDROID_MAJOR_VERSION=s
export ARCH=arm64

# Generate defconfig
make exynos850-a13xx_defconfig

# Build kernel (tambahkan -j untuk paralel — tidak disebutkan Samsung tapi sangat disarankan)
make -j$(nproc) 2>&1 | tee build.log
```

---

### Metode B: Argumen Command Line (Lebih Fleksibel, Tanpa Edit Makefile)

Metode ini tidak mengubah source tree dan lebih mudah dikelola jika berganti toolchain:

```bash
cd kernel_a135f_v1

# Aktifkan toolchain
source ~/setup_toolchain_a135f.sh

CLANG_DIR="$HOME/toolchains/clang-r383902"
GCC64_DIR="$HOME/toolchains/gcc-aarch64"
THREADS=$(nproc --all)

# Langkah 1: Generate defconfig
make \
    ARCH=arm64 \
    PLATFORM_VERSION=12 \
    ANDROID_MAJOR_VERSION=s \
    CC=$CLANG_DIR/bin/clang \
    CLANG_TRIPLE=$CLANG_DIR/bin/aarch64-linux-gnu- \
    CROSS_COMPILE=$GCC64_DIR/bin/aarch64-linux-android- \
    exynos850-a13xx_defconfig

# Langkah 2: (Opsional) Modifikasi config secara interaktif
# make ARCH=arm64 menuconfig

# Langkah 3: Build kernel
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
cd kernel_a135f_v1
cat build_kernel.sh        # Periksa path toolchain di dalam script
chmod +x build_kernel.sh
./build_kernel.sh
```

### 8.2 Cek Hasil Build

```bash
if [ $? -eq 0 ]; then
    echo "✅ Build BERHASIL"
    ls -lh arch/arm64/boot/Image
else
    echo "❌ Build GAGAL — cek build.log"
    grep -i "error:" build.log | tail -20
fi
```

### 8.3 Clean Build

Sesuai `README_Kernel.txt` resmi Samsung:

```bash
# Clean output build
make clean

# Clean lebih menyeluruh (termasuk generated config)
make mrproper
```

---

## 9. Output Build

Berdasarkan `README_Kernel.txt` resmi Samsung, ada dua jenis output:

### 9.1 Kernel Image

```
arch/arm64/boot/Image
```

File ini adalah kernel image utama yang diflash ke partisi `boot`.

```bash
# Verifikasi
ls -lh arch/arm64/boot/Image
file arch/arm64/boot/Image
# Output: Linux kernel ARM64 boot executable Image, ...
```

### 9.2 Kernel Modules

```
drivers/*/*.ko
```

File `.ko` (kernel object) adalah modul driver yang dapat dimuat secara dinamis. Cari seluruh module yang dihasilkan:

```bash
find . -name "*.ko" -not -path "./.git/*" | sort
# Contoh output:
# ./drivers/net/wireless/qualcomm/wcn399x/wcnss/wcnss_wlan.ko
# ./drivers/staging/nanohub/nanohub.ko
```

> 💡 **Catatan:** Output resmi Samsung adalah `Image` (uncompressed), bukan `Image.gz` atau `Image.gz-dtb`. Jika AnyKernel3 membutuhkan format lain, buat secara manual (lihat Troubleshooting).

---

## 10. Modifikasi Kernel (Opsional)

### 10.1 Ubah Custom Kernel String

```bash
nano arch/arm64/configs/exynos850-a13xx_defconfig
# Cari: CONFIG_LOCALVERSION
# Ubah nilai sesuai keinginan, contoh:
# CONFIG_LOCALVERSION="-MyKernel-v1.0"
```

### 10.2 Aktifkan Opsi Tambahan

Jalankan dulu generate defconfig, kemudian gunakan `scripts/config`:

```bash
make ARCH=arm64 exynos850-a13xx_defconfig

# Tambah CPU governor
scripts/config --enable CONFIG_CPU_FREQ_GOV_CONSERVATIVE
scripts/config --enable CONFIG_CPU_FREQ_GOV_ONDEMAND

# Aktifkan ZRAM dengan LZ4
scripts/config --enable CONFIG_ZRAM
scripts/config --enable CONFIG_ZRAM_DEF_COMP_LZ4

# Perbaiki dependensi
make ARCH=arm64 olddefconfig
```

### 10.3 Modifikasi Config Interaktif

```bash
make ARCH=arm64 menuconfig
```

### 10.4 Menerapkan Patch

```bash
patch -p1 < /path/to/patch.patch
# atau
git am /path/to/patch.patch
```

### 10.5 Simpan Config yang Dimodifikasi

```bash
make ARCH=arm64 savedefconfig
cp defconfig arch/arm64/configs/exynos850-a13xx_defconfig
```

---

## 11. Membuat Flashable ZIP

### 11.1 Clone AnyKernel3

```bash
cd ~
git clone https://github.com/osm0sis/AnyKernel3.git AnyKernel3_a135f_v1
cd AnyKernel3_a135f_v1
```

### 11.2 Konfigurasi `anykernel.sh`

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

### 11.3 Copy Kernel Image

Output resmi Samsung adalah `arch/arm64/boot/Image`:

```bash
# Hapus contoh bawaan AnyKernel3
rm -f Image Image.gz Image.gz-dtb zImage *.zip

# Copy kernel image hasil build
cp ~/kernel_a135f_v1/arch/arm64/boot/Image ~/AnyKernel3_a135f_v1/

ls -lh ~/AnyKernel3_a135f_v1/Image
```

### 11.4 (Opsional) Sertakan Kernel Modules

```bash
mkdir -p ~/AnyKernel3_a135f_v1/modules/system/lib/modules

find ~/kernel_a135f_v1/drivers -name "*.ko" \
    -exec cp {} ~/AnyKernel3_a135f_v1/modules/system/lib/modules/ \;
```

### 11.5 Buat ZIP

```bash
cd ~/AnyKernel3_a135f_v1

ZIP_NAME="StockBase_v1_A135F_$(date +%Y%m%d_%H%M%S).zip"

zip -r9 "$ZIP_NAME" . -x "*.git*" -x "*.zip"

echo "✅ ZIP: $ZIP_NAME ($(du -sh $ZIP_NAME | cut -f1))"
```

---

## 12. Flashing ke Perangkat

### Prasyarat
- ✅ Custom Recovery terpasang (TWRP)
- ✅ Firmware Samsung One UI 4.x / Android 12 (SDEYJ1 atau kompatibel)
- ✅ Data sudah di-backup
- ✅ Baterai minimal 50%

### 12.1 Transfer ke Perangkat

```bash
adb push ~/AnyKernel3_a135f_v1/StockBase_v1_A135F_*.zip /sdcard/
adb shell ls -lh /sdcard/StockBase_v1_A135F_*.zip
```

### 12.2 Boot ke Recovery

```bash
adb reboot recovery
# Atau manual: tahan Volume Up + Power saat perangkat mati
```

### 12.3 Flash via TWRP

1. Tap **Install** → Pilih file ZIP
2. **Swipe to Confirm Flash**
3. **Reboot System**

### 12.4 Flash Langsung via Heimdall (Tanpa Recovery)

```bash
sudo apt install heimdall-flash -y

# Setup udev rules agar tidak butuh sudo
echo 'SUBSYSTEM=="usb", ATTRS{idVendor}=="04e8", MODE="0666", GROUP="plugdev"' | \
    sudo tee /etc/udev/rules.d/51-samsung.rules
sudo udevadm control --reload-rules && sudo udevadm trigger
sudo usermod -aG plugdev $USER
# Logout & login kembali

# Boot ke Download Mode: Volume Down + Power → Volume Up
heimdall detect

# Flash Image ke partisi boot
heimdall flash --BOOT ~/kernel_a135f_v1/arch/arm64/boot/Image --no-reboot
```

---

## 13. Menggunakan v1 Sebagai Base Pengembangan

### 13.1 Migrasi v1 ke Setara rksu (Tambah KernelSU Manual)

```bash
cd kernel_a135f_v1

# Tambahkan KernelSU rsuntk via setup script
curl -LSs "https://raw.githubusercontent.com/rsuntk/KernelSU/main/kernel/setup.sh" | bash -s main

# Verifikasi
ls KernelSU/
grep "CONFIG_KSU" arch/arm64/configs/exynos850-a13xx_defconfig
```

### 13.2 Buat Branch Custom Sendiri

```bash
git checkout -b my-custom-kernel

# Lakukan modifikasi, lalu commit
git add -A
git commit -m "feat: custom governor + ZRAM LZ4"

# Lihat perbedaan dari v1
git diff v1..my-custom-kernel --stat
```

### 13.3 Generate Patch dari Modifikasi

```bash
git format-patch v1..my-custom-kernel --output-directory patches/
ls patches/
```

---

## 14. Troubleshooting

### ❌ `make: exynos850-a13xx_defconfig: No rule to make target`

**Penyebab:** `ARCH=arm64` belum di-export.

```bash
export ARCH=arm64
make exynos850-a13xx_defconfig
```

---

### ❌ `clang: command not found` atau `aarch64-linux-android-gcc: command not found`

**Penyebab:** PATH toolchain belum di-export.

```bash
source ~/setup_toolchain_a135f.sh
# Atau:
export PATH="$HOME/toolchains/clang-r383902/bin:$HOME/toolchains/gcc-aarch64/bin:$PATH"
which clang && which aarch64-linux-android-gcc
```

---

### ❌ `error: implicit declaration of function`

**Penyebab:** Kemungkinan menggunakan Clang versi yang berbeda dari yang direkomendasikan Samsung.

**Solusi:** Pastikan menggunakan **clang-r383902** sesuai README resmi:

```bash
clang --version | grep -i "r383902"
# Harus muncul di output
```

---

### ❌ `scripts/gcc-version.sh: command not found` atau error GCC

**Penyebab:** GCC host atau cross-compiler tidak ditemukan di PATH.

```bash
export PATH="$HOME/toolchains/gcc-aarch64/bin:$PATH"
# Atau install GCC host:
sudo apt install gcc -y
```

---

### ❌ Build berhasil tapi AnyKernel3 butuh `Image.gz` atau `Image.gz-dtb`

Output resmi Samsung hanya `Image`. Buat format lain secara manual jika diperlukan:

```bash
# Buat Image.gz
gzip -k -f arch/arm64/boot/Image
ls arch/arm64/boot/Image.gz

# Buat Image.gz-dtb (gabungkan Image.gz dengan DTB)
cat arch/arm64/boot/Image.gz \
    arch/arm64/boot/dts/exynos/exynos3830*.dtb \
    > arch/arm64/boot/Image.gz-dtb
```

---

### ❌ Kernel Bootloop / Panic Setelah Flash

**Solusi — Restore kernel stock:**

```bash
# Download firmware A135FXXSDEYJ1 dari SamFw/SamMobile
# Extract dan decompress boot.img:
lz4 -d boot.img.lz4 boot.img

# Flash stock kernel
heimdall flash --BOOT boot.img
```

---

### 🔍 Membaca Build Log

```bash
grep -n "error:" build.log            # Semua error
grep -n -B3 -A3 "error:" build.log | head -60  # Error dengan konteks
grep "^  CC " build.log | tail -5     # File terakhir dikompilasi
```

---

## 15. Referensi & Kredit

### Repository

| Link | Keterangan |
|---|---|
| [kucingsakti/android_kernel_a135f (v1)](https://github.com/kucingsakti/android_kernel_a135f/tree/v1) | Kernel source (branch v1) |
| [kucingsakti/android_kernel_a135f (rksu)](https://github.com/kucingsakti/android_kernel_a135f/tree/rksu) | Branch dengan KernelSU |
| [osm0sis/AnyKernel3](https://github.com/osm0sis/AnyKernel3) | Framework flashable ZIP kernel |
| [rsuntk/KernelSU](https://github.com/rsuntk/KernelSU) | KernelSU fork untuk Non-GKI |

### Firmware & Komunitas

| Link | Keterangan |
|---|---|
| [XDA Galaxy A13 Forum](https://xdaforums.com/f/samsung-galaxy-a13.12569/) | Forum komunitas |
| [SamMobile A135F Firmware](https://www.sammobile.com/samsung/galaxy-a13/firmware/SM-A135F/) | Download firmware Samsung |
| [SamFw](https://samfw.com/firmware/SM-A135F) | Alternatif download firmware |

### Tools

| Tool | Kegunaan |
|---|---|
| [Heimdall](https://github.com/Benjamin-Dobell/Heimdall) | Flash firmware Samsung dari Linux/Mac |
| [TWRP](https://twrp.me) | Custom recovery |

### Kredit
- **Samsung** — Kernel source asli + `README_Kernel.txt` sebagai acuan build resmi
- **kucingsakti** — Maintainer repository ini
- **osm0sis** — Penulis AnyKernel3

---

<div align="center">

**⚠️ DISCLAIMER**

*Modifikasi kernel dapat membatalkan garansi dan berpotensi merusak perangkat. Selalu backup data sebelum melakukan modifikasi sistem. Gunakan panduan ini atas risiko Anda sendiri.*

</div>

---

*Guide ini disusun berdasarkan analisis repository `kucingsakti/android_kernel_a135f` branch `v1` dan `README_Kernel.txt` resmi Samsung.*
