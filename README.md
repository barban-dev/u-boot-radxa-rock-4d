# u-boot-radxa-rock-4d
This repository provides a custom-built U-Boot firmware for the Radxa ROCK 4D (RK3576). Unlike the stock bootloader from the vendor, this version is specifically configured to initialize the onboard USB 3.0/2.0 controllers and the Naneng Combo PHY, enabling seamless and automatic booting from USB drives.

## Background

1. **Vendor Limitations:** The official Radxa ROCK 4D bootloader does not support booting from USB disks; it is primarily designed for eMMC, SD, or NVMe storage via the board built-in FPC connector.
2. **OS Limitations:** Standard OS images (such as Raspbian or Debian ports for ROCK 4D) include a bootloader that lacks the necessary drivers to initialize the USB controllers during the boot sequence.
3. **Solution:** A custom configuration of Mainlline U-Boot from source, specifically tailored for the ROCK 4D (RK3576), to include USB Host, xHCI, and Naneng Combo PHY support.

## Build Processon 

The following steps were performed to generate the working firmware:

### 1. Source and binaries retrieval
```bash
# Clone Rockchip binary blobs (TPL/BL31)
git clone --depth 1 https://github.com/rockchip-linux/rkbin

# Clone Mainline U-Boot
git clone https://source.denx.de/u-boot/u-boot.git
```

### 2. Environment Configuration
```bash
cd u-boot
export BL31=../rkbin/bin/rk35/rk3576_bl31_v1.20.elf
export ROCKCHIP_TPL=../rkbin/bin/rk35/rk3576_ddr_lp4_2112MHz_lp5_2736MHz_v1.09.bin

# Apply the default configuration for the ROCK 4D
make rock-4d-rk3576_defconfig
```

### 3. Key config modifications (`make menuconfig`)
The following options were enabled to ensure full USB support while keeping the SPL (Secondary Program Loader) size within limits and avoiding compilation errors:

*   **USB Host Support:** `CONFIG_USB=y`, `CONFIG_USB_HOST=y`, `CONFIG_DM_USB=y`, 
*   **Drivers:** `CONFIG_USB_XHCI_HCD=y`, `CONFIG_USB_XHCI_DWC3=y`, `CONFIG_USB_DWC3=y`, `CONFIG_USB_DWC3_GENERIC=y`
*   **PHY Support:** `CONFIG_PHY_ROCKCHIP_NANENG_COMBPHY=y`, `CONFIG_PHY_ROCKCHIP_INNO_USB2=y`
*   **Storage:** `CONFIG_USB_STORAGE=y`
*   **SPL Compatibility:** `CONFIG_SPL_USB_HOST=n` (disabled to prevent SPL size overflow and linker errors)
*   **Controller Limit:** `CONFIG_USB_HOST_MAX=2`

Instead of manually configuring the build via `make menuconfig`, you can use the pre-configured config file in this repository:

1. Download the `rock-4d-rk3576_u-boot_usb_defconfig` file from this repo.
2. Rename the file as `.config` and place it into your `u-boot` source root directory.
3. Skip the `make menuconfig` step and proceed directly to compilation.

### 4. Native Compilation on ROCK 4D SBC board
The build produces the file `u-boot-rockchip-spi.bin`, which is ready to be flashed to the onboard SPI Flash.
```bash
make
```
Alternative: `make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu-` for cross-compilation on a x86_64 PC.

### 5. Flash to SPI Flash
From a running Linux system on the ROCK 4D, identify the SPI device (usually `/dev/mtdblock0`) and run:
```bash
# Reset SPI flash (optional but recommended)
sudo dd if=/dev/zero of=/dev/mtdblock0 bs=1M count=4 2>/dev/null
# flash the SPI bootloader
sudo dd if=u-boot-rockchip-spi.bin of=/dev/mtdblock0 bs=4K
sync
# Verify written image (optional but recommended): the two commands below should give the same result 
sudo head -c $(stat -c%s u-boot-rockchip-spi.bin) /dev/mtdblock0 | md5sum
md5sum u-boot-rockchip-spi.bin
```

## Expected Results
After flashing this U-Boot to the SPI Flash, the ROCK 4D is capable of:
* Initializing the USB bus automatically at power-on.
* Detecting USB Mass Storage devices.
* Booting Linux distributions (like Armbian) stored on USB disks without any manual console commands.

## Verified Test Environment
The custom U-Boot build has been successfully verified with the following configuration:

### Operating System
*   **Linux Distribution:** Armbian 26.2.0-trunk.363 (Ubuntu 24.04 LTS "noble")
*   **Kernel:** Mainline-based Armbian build for ROCK 4D

### Storage Hardware
*   **Device:** **SSK USB 3.2 SSD Flash Drive**
*   **Controller:** VIA Labs, Inc. VL817 SATA Adaptor (ID 2109:0715)
*   **Interface:** USB 3.1 SuperSpeed (bcdUSB 3.10)
*   **Performance:** Successfully initialized as a Mass Storage device (SCSI Bulk-Only) during the U-Boot sequence.

### Obtained result
U-Boot initializes the USB 3.x controller, detects the VIA Labs bridge, and hands over the boot sequence to Armbian without any manual console intervention.
