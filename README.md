# u-boot-radxa-rock-4d
This repository provides a custom-built U-Boot firmware for the Radxa ROCK 4D (RK3576). Unlike the stock bootloader from the vendor, this version is specifically configured to initialize the onboard USB 3.0/2.0 controllers and the Naneng Combo PHY, enabling seamless and automatic booting from USB drives.

## Background

1. **Vendor Limitations:** The official Radxa ROCK 4D bootloader does not support booting from USB disks; it is primarily designed for eMMC, SD, or NVMe storage via the board built-in FPC connector.
2. **OS Limitations:** Standard OS images (such as Raspbian or Debian ports for ROCK 4D) include different bootloaders that lack the necessary drivers to initialize the USB controllers during the boot sequence.
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
# Pre-requisites
sudo apt-get install bc bison build-essential coccinelle \
  device-tree-compiler dfu-util efitools flex gdisk graphviz imagemagick \
  libgnutls28-dev libguestfs-tools libncurses-dev \
  libpython3-dev libsdl2-dev libssl-dev lz4 lzma lzma-alone openssl \
  pkg-config python3 python3-asteval python3-coverage python3-filelock \
  python3-pkg-resources python3-pycryptodome python3-pyelftools \
  python3-pytest python3-pytest-xdist python3-sphinxcontrib.apidoc \
  python3-sphinx-rtd-theme python3-subunit python3-testtools \
  python3-venv swig uuid-dev

# Build the u-boot firmware
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

### Obtained results
U-Boot initializes the USB 3.x controller, detects the VIA Labs bridge, and hands over the boot sequence to Armbian without any manual console intervention.

#### Boot Log (UART Debug)

<details>
<summary>Click to expand the full boot sequence log</summary>

```text
DR 2f85f4b2d4 cym 24/11/07-19:07:28,fwver: v1.09
In
ch0 ttot6
ch1 ttot6
... [DDR Initialization] ...
U-Boot SPL 2026.04-rc1-00063-geed514b11d04 (Jan 31 2026 - 22:29:40 +0100)
Trying to boot from SPI
## Checking hash(es) for config config-1 ... OK
...
U-Boot 2026.04-rc1-00063-geed514b11d04 (Jan 31 2026 - 22:29:40 +0100)
Model: Radxa ROCK 4D
SoC:   RK3576
DRAM:  4 GiB
...
USB XHCI 1.10
USB XHCI 1.10
Bus usb@23000000: 2 USB Device(s) found
Bus usb@23400000: 3 USB Device(s) found
Scanning bootdev 'usb_mass_storage.lun0.bootdev':
  0  script       ready   usb_mass_    1  usb_mass_storage.lun0.boo /boot/boot.scr
** Booting bootflow 'usb_mass_storage.lun0.bootdev.part_1' with script
Boot script loaded from usb 0:1
...
Starting kernel ...
Armbian 26.2.0-trunk.363 noble ttyFIQ0 
radxa-rock-4d login:
```
</details>

#### Performance Benchmarks - Sequential Write Bandwidth and Latency
To quantify the benefits of enabling USB 3.2 boot support, a comparative I/O performance test was conducted using `fio`. The tests compare the **SSK USB 3.2 SSD Flash Drive** (connected via the reconfigured USB 3.0 port) against the sdcard storage **SandDisk Ultra microSDXC UHS-I A1** (`mmcblk1`).

| Storage Media | Write Bandwidth (BW) | IOPS | Avg. Latency (clat) |
| :--- | :--- | :--- | :--- |
| **SSK USB 3.2 SSD (USB 3.0)** | **317 MB/s** | **2417** | **51.01 ms** |
| **SandDisk Ultra microSDXC UHS-I A1 sdcard** | **25.3 MB/s** | **192** | **660.92 ms** |


### Conclusions
1.  **Speed:** The USB 3.2 SSD is **12.5x faster** than the A1-rated microSD card in sequential write operations.
2.  **Responsiveness:** The average latency on the USB drive is **13x lower** (51ms vs 660ms), which significantly improves system responsiveness, especially during heavy I/O tasks or software updates.
3.  **IOPS:** The USB setup handles over **2400 operations per second**, compared to fewer than 200 on the sdcard media, making it a superior choice for running a database or a desktop environment.

Enabling USB boot via this custom U-Boot build doesn't just provide a convenient storage alternative; it unlocks a massive performance tier for the ROCK 4D, transforming it from a standard SBC into a high-performance node capable of saturating the USB 3.0 bus.

