Kernel Compilation Guide: Exynos 1330 (SM-A146M / SM-A146B)
This guide details the step-by-step process used to build and instrument the Linux kernel (v5.15) for the Samsung Galaxy A14 5G (Exynos 1330 / s5e8535) natively on Android using Termux and a PRoot Ubuntu container.
1. Environment Setup (Termux)
First, set up storage permissions and install proot-distro inside Termux:
# Request storage permissions
termux-setup-storage

# Update Termux packages
yes | pkg update && yes | pkg upgrade

# Install PRoot and setup Ubuntu container
pkg install proot-distro
proot-distro install ubuntu
proot-distro login ubuntu


2. Dependencies & Source Preparation (Ubuntu PRoot)
Inside the Ubuntu container, install the required toolchains (LLVM/Clang 17) and build utilities:
# Update Ubuntu packages and install build dependencies
yes | apt update && yes | apt upgrade && apt install nano build-essential flex bison bc libssl-dev libelf-dev clang-17 llvm-17 lld-17 cpio xz-utils kmod lz4 git dwarves

# Copy kernel sources from Android internal storage (/sdcard) to home directory
cp /sdcard/Kernel.tar.gz ~/
cp /sdcard/A146BXXSDDZE1_kernel.tar.gz ~/

# Extract kernel sources
mkdir Kernel
tar -xf Kernel.tar.gz -C Kernel
tar -xf A146BXXSDDZE1_kernel.tar.gz -C Kernel

# Exit container to restart with isolation mode
exit


Log back into Ubuntu with isolation mode and configure LLVM 17 environment variables:
proot-distro login ubuntu --isolated

# Set up toolchain PATH
export PATH="/usr/lib/llvm-17/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
source ~/.bashrc


3. Kernel Configuration & Compilation
Navigate to the kernel source directory and set environment variables for cross-compilation with Clang/LLVM:
cd Kernel

# Target Architecture and SoC Configuration
export ARCH=arm64
export SUBARCH=arm64
export TARGET_SOC=s5e8535

# Toolchain configuration
export LLVM=1
export LLVM_IAS=1
export DEPMOD=depmod
export DTC_FLAGS="-@"
export PATH="/usr/lib/llvm-17/bin:$PATH"

# Generate defconfig for Galaxy A14
make s5e8535-a14xxx_defconfig


Initial Build Command:
make -j$(nproc) LLVM=1 ARCH=arm64 TARGET_SOC=s5e8535


4. Troubleshooting & Workarounds
During compilation with Clang 17, strict C standards may trigger build errors related to missing type specifiers and deprecated function declarations:
Encountered Errors:
Implicit Type Error: kernel/sched/ems/sysbusy.c:43:8: error: type specifier missing, defaults to 'int' [-Werror,-Wimplicit-int]
Missing Prototype Error: mm/filemap.c:2991:37: error: a function declaration without a prototype is deprecated in all versions of C [-Werror,-Wstrict-prototypes]
Resolution:
Bypass strict compiler warnings treated as errors by passing additional KCFLAGS:
make -j$(nproc) LLVM=1 ARCH=arm64 TARGET_SOC=s5e8535 \
KCFLAGS="-Wno-error=implicit-int -Wno-error=strict-prototypes"


Expected Non-Fatal Warnings during Final Linking:
During the build, harmless cp missing header warnings related to perflog.h may appear:
cp: cannot stat './../../system/core/liblog/include/log/perflog.h': No such file or directory
kperfmon_DUMMY=""


These do not prevent successful vmlinux and image linkage.
5. Boot Image Repacking & Deployment
Once compilation completes successfully, copy the output Image.gz to storage and pack it into a flashable boot image using magiskboot:
# Copy compiled kernel image to home directory and SD card
cd ~/Kernel
cp arch/arm64/boot/Image.gz /sdcard/

# Repack boot.img using magiskboot
cp /sdcard/boot.img ~/
cp /sdcard/Image ~/

# Unpack stock boot.img
magiskboot unpack boot.img

# Replace stock kernel with freshly built kernel
rm -r kernel
cp Image kernel

# Repack new boot image
magiskboot repack boot.img

# Output to internal storage
cp new-boot.img /sdcard/


You now have a custom compiled kernel packaged in new-boot.img ready for flashing via Odin, Heimdall, or custom recovery.
