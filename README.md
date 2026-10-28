# MSM8953 Linux Kernel (Improved Mainline)

This repository provides an improved close-to-mainline Linux kernel (based on version 7.1.3) optimized for Qualcomm Snapdragon 625 (MSM8953) hardware, with first-class support for postmarketOS, Debian/Alpine chroots, and the Xiaomi Redmi 5 Plus (codename: vince).

## Overview

The Qualcomm Snapdragon 625 (MSM8953) is an octa-core ARM Cortex-A53 system-on-chip with an Adreno 506 GPU. While modern Linux kernels support many MSM8953 features, certain critical hardware interfaces (notably TDDI in-cell touchscreens and Bluetooth network encapsulation) require driver modifications and tailored device tree configurations.

This repository resolves these issues to provide a stable, feature-complete mainline kernel experience.

## Key Improvements

### 1. Synaptics ClearPad TD4310 In-Cell TDDI Touchscreen Fix

* File: `drivers/input/rmi4/rmi_driver.c`
* Problem: The Synaptics ClearPad TD4310 TDDI integrated display/touch controller reports an extended presence register size (72 bytes) during Function 12 descriptor discovery. Upstream RMI4 drivers enforce a strict check (`size_presence_reg > 35 -> return -EIO`), causing touchscreen initialization to fail completely on Xiaomi Redmi 5 Plus and other TDDI-equipped devices.
* Solution: `rmi_read_register_desc()` now clamps `size_presence_reg` to `sizeof(buf)` (35 bytes). This eliminates stack buffer overflow risks while successfully reading and registering all standard F12 query, control, and data descriptors. Touch digitizer input is fully functional out of the box.

### 2. Device Tree Optimization for Xiaomi Redmi 5 Plus (vince)

* File: `arch/arm64/boot/dts/qcom/msm8953-xiaomi-vince.dts`
* Enables the Synaptics RMI4 touch controller node (`&rmi4_ts`) in conjunction with the driver fix.
* Retains calibrated touchscreen regulators, interrupt lines (`tlmm 65`), I2C bus wiring, and display continuous splash timings.

### 3. Bluetooth PAN (BNEP) Kernel Module Support

* Files: `arch/arm64/configs/msm8953.config`, `arch/arm64/configs/xiaomi-vince.config`, `arch/arm64/configs/pmos_msm8953_defconfig`
* Enabled kernel options:
  * `CONFIG_BT_BNEP=m` (Bluetooth Network Encapsulation Protocol)
  * `CONFIG_BT_BNEP_MC_FILTER=y` (Multicast filter support)
  * `CONFIG_BT_BNEP_PROTO_FILTER=y` (Protocol filter support)
* Enables native Bluetooth Personal Area Networking (PAN / NAP / PANU) tethering.

### 4. Verified postmarketOS & Debian Chroot Defconfig

* File: `arch/arm64/configs/pmos_msm8953_defconfig`
* Provides a complete, tested kernel configuration derived from production hardware running postmarketOS and Debian Bookworm. Includes full support for USB RNDIS gadget networking (`dwc3`), PMIC charging management, and hardware crypto accelerators.

## Building the Kernel

### Prerequisites

Install build dependencies and cross-compilation toolchains on an Ubuntu or Debian build host:

```bash
sudo apt update
sudo apt install -y build-essential libncurses-dev bison flex libssl-dev libelf-dev bc git
sudo apt install -y clang lld llvm gcc-aarch64-linux-gnu
```

### Configuration

To configure for Xiaomi Redmi 5 Plus (vince):

```bash
export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-gnu-

# Generate base configuration
make LLVM=1 defconfig msm8953.config xiaomi-vince.config
```

Alternatively, use the full verified production defconfig:

```bash
make LLVM=1 pmos_msm8953_defconfig
```

### Compilation

Compile the kernel image, device tree blobs, and modules:

```bash
make -j$(nproc) LLVM=1 Image dtbs modules
```

The compiled kernel image will be located at `arch/arm64/boot/Image`, and device tree blobs at `arch/arm64/boot/dts/qcom/`.

## Booting and Deployment

### Bootloader Compatibility

MSM8953 devices typically use `lk2nd` (Little Kernel secondary bootloader) as an intermediate boot stage:
1. Flash `lk2nd` to your device's boot partition (`boot` or `recovery`).
2. `lk2nd` presents a standard fastboot interface and loads the mainline Linux kernel (`Image`) and appended device tree (`msm8953-xiaomi-vince.dtb`).

### Packaging with mkbootimg

```bash
cat arch/arm64/boot/Image arch/arm64/boot/dts/qcom/msm8953-xiaomi-vince.dtb > Image.gz-dtb

mkbootimg \
  --kernel Image.gz-dtb \
  --ramdisk initramfs.cpio.gz \
  --base 0x80000000 \
  --pagesize 2048 \
  --cmdline "console=ttyMSM0,115200 earlycon=msm_geni_serial,0x75b0000 root=/dev/mmcblk0p50 rootwait rw" \
  --output boot.img
```

Flash or boot via fastboot:

```bash
fastboot boot boot.img
```

## Supported Devices

* Xiaomi Redmi 5 Plus (`vince`)
* Compatible with other Qualcomm MSM8953 (Snapdragon 625) devices sharing common display, power, and sensor subsystems.

## License

This project is licensed under the GNU General Public License v2.0 (GPL-2.0-only), matching the upstream Linux kernel license. See `COPYING` for details.
