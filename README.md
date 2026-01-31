# u-boot-radxa-rock-4d
This repository provides a custom-built U-Boot firmware for the Radxa ROCK 4D (RK3576). Unlike the stock bootloader from the vendor, this version is specifically configured to initialize the onboard USB 3.0/2.0 controllers and the Naneng Combo PHY, enabling seamless and automatic booting from USB drives.

## Project Background

1. **Vendor Limitations:** The official Radxa ROCK 4D bootloader does not support booting from USB disks; it is primarily designed for eMMC, SD, or NVMe storage via the FPC connector.
2. **OS Limitations:** Standard OS images (such as Raspbian or Debian ports for ROCK 4D) include a bootloader that lacks the necessary drivers to initialize the USB controllers during the boot sequence.
3. **The Solution:** A custom compilation of U-Boot from source, specifically tailored for the RK3576 SoC, to include USB Host, xHCI, and Naneng Combo PHY support.

## Technical Build Process

The following steps were performed to generate the working firmware:

### 1. Source and Binaries Retrieval
```bash
# Clone Rockchip binary blobs (TPL/BL31)
git clone --depth 1 https://github.com

# Clone Mainline U-Boot
git clone https://source.denx.de
cd u-boot
