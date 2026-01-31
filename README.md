# u-boot-radxa-rock-4d
This repository provides a custom-built U-Boot firmware for the Radxa ROCK 4D (RK3576). Unlike the stock bootloader from the vendor, this version is specifically configured to initialize the onboard USB 3.0/2.0 controllers and the Naneng Combo PHY, enabling seamless and automatic booting from USB drives.

## Background

1. **Vendor Limitations:** The official Radxa ROCK 4D bootloader does not support booting from USB disks; it is primarily designed for eMMC, SD, or NVMe storage via the FPC connector.
2. **OS Limitations:** Standard OS images (such as Raspbian or Debian ports for ROCK 4D) include a bootloader that lacks the necessary drivers to initialize the USB controllers during the boot sequence.
3. **The Solution:** A custom compilation of U-Boot from source, specifically tailored for the RK3576 SoC, to include USB Host, xHCI, and Naneng Combo PHY support.

## Build Process

The following steps were performed to generate the working firmware:

### 1. Source and binaries retrieval
```bash
# Clone Rockchip binary blobs (TPL/BL31)
git clone --depth 1 https://github.com

# Clone Mainline U-Boot
git clone https://source.denx.de
cd u-boot
```

### 2. Environment Configuration
```bash
export BL31=../rkbin/bin/rk35/rk3576_bl31_v1.20.elf
export ROCKCHIP_TPL=../rkbin/bin/rk35/rk3576_ddr_lp4_2112MHz_lp5_2736MHz_v1.09.bin

# Apply the default configuration for the ROCK 4D
make rock-4d-rk3576_defconfig
```

### 3. Key Modifications (`make menuconfig`)
The following options were enabled to ensure full USB support while keeping the SPL (Secondary Program Loader) size within limits and avoiding compilation errors:

*   **General USB:** `CONFIG_USB=y`, `CONFIG_DM_USB=y`, `CONFIG_USB_HOST=y`
*   **Drivers:** `CONFIG_USB_XHCI_HCD=y`, `CONFIG_USB_XHCI_DWC3=y`, `CONFIG_USB_DWC3=y`, `CONFIG_USB_DWC3_GENERIC=y`
*   **PHY Support:** `CONFIG_PHY_ROCKCHIP_NANENG_COMBPHY=y`, `CONFIG_PHY_ROCKCHIP_INNO_USB2=y`
*   **Storage:** `CONFIG_USB_STORAGE=y`
*   **SPL Compatibility:** `CONFIG_SPL_USB_HOST=n` (crucial to prevent linker errors and size overflow during the SPL phase).




