---
sidebar_position: 2
---

# Boot Development Guide

## 1. Overview

### 1.1 Purpose of the Document

This document primarily introduces the firmware flashing and boot process for SpacemiT, along with custom configuration options. It also covers driver development related to U-Boot and OpenSBI, basic debugging methods, aiming to facilitate quick start or secondary development for developers.

### 1.2 Scope of Application

This document is applicable to the K1 series SoC (System on Chip) of SpacemiT.

### 1.3 Relevant Personnel

- Firmware flashing and boot development engineers
- Kernel development engineers

### 1.4 Document Structure

The document starts by detailing the firmware flashing and boot process and configuration methods, followed by compilation configuration and driver development debugging for U-Boot/OpenSBI. Finally, it records common troubleshooting tips.

## 2. Firmware Flashing and Boot Process

This chapter describes the configuration and implementation principles related to brushing and startup, and provides custom brushing and startup methods.
Note that brushing and startup are related, and if you need to customize the brushing machine, startup may also need to make relevant adjustments, and vice versa.

### 2.1 Firmware Layout

Common firmware layouts for the K1 series SoC include three types:

![a](static/image_play.png)

- eMMC

i. As shown in the diagram, eMMC has two areas: boot0 and user_data_area. This storage feature is inherent and will not be elaborated further.

ii. `bootinfo_emmc.bin` and `FSBL.bin` are stored in the boot0 area. The user_data_area's fsbl partition does not contain actual content. After brom boots, it loads `bootinfo_emmc.bin` and `FSBL.bin` from the boot0 area.

- SD Card

i. Similar to eMMC but only includes the user_data_area. `bootinfo_sd.bin`occupies the first 80 bytes, with the GPT table located at 1 blk position.

- NOR + SSD

i. Includes NOR firmware (fsbl + opensbi + uboot) and SSD firmware (bootfs + rootfs).

ii. Also supports combinations like NOR + eMMC, defaulting to NOR + eMMC if no SSD is inserted and the switch is set to NOR boot.

iii. Only supports SSD as a firmware flashing medium when inserted into pcie0 interface. If an SSD is inserted into a non-pcie0 interface, it functions solely as storage.

iv. When using TitanFlash tool for firmware flashing, the SSD's partition table includes fsbl/ubot/opensbi, but these partitions do not take effect. The actual functionality is determined by the actual partition table.

### 2.2 Firmware Flashing Process and Configuration

Firmware flashing and writing refer to the same concept. Mass production of cards is essentially firmware flashing, with data sourced from SD cards.

Card Production and Card Boot are distinct concepts. Card Boot involves burning the image onto the SD card and booting the system from the SD card.

#### 2.2.1 Firmware Flashing Process

The firmware flashing process utilizes the fastboot protocol to transmit images from a PC to the device, then writes them to the corresponding storage medium via interfaces such as mmc/mtd/nvme. A complete firmware flashing process includes entering U-Boot fastboot mode, image transmission, detection of storage media, creation of the GPT table, and writing data to the storage medium.

##### 2.2.1.1 U-Boot Fastboot Mode

In this platform, the firmware flashing communication protocol is based on the fastboot protocol with some custom extensions. Actual firmware flashing operations occur within the U-Boot fastboot mode.

After entering U-Boot fastboot mode, executing fastboot devices on the PC detects the device (PC fastboot environment needs to be configured).

```sh
~$ fastboot devices
c3bc939586f0         Android Fastboot
```

Below are three methods to enter U-Boot fastboot mode:

1.By pressing the combination of FEL and RESET buttons on the board to enter brom-fastboot, then executing the following commands on the host machine (PC), the board enters U-Boot flashing interaction interface. Brom-fastboot is used only for loading and booting U-Boot fastboot.

```sh
fastboot stage factory/FSBL.bin
fastboot continue
#sleep wait for uboot ready
#linux环境下
sleep 1
#windows环境下                
#timeout /t 1 >null     
fastboot stage u-boot.itb
fastboot continue
```

2.For devices already booted into OS, run adb reboot bootloader on the host machine (PC) to enter U-Boot flashing interaction interface. (Some firmware versions might remove adb functionality, making this method non-universal.)

3.During boot, hold the 's' key via serial port to enter U-Boot shell, then execute fastboot 0 to enter U-Boot flashing interaction interface.

Below are various combinations of media for firmware flashing commands, where brom can switch between different boot media (NOR/NAND/eMMC) based on boot pins, depending on hardware design. Please refer to the hardware reference design for more details.

##### 2.2.1.2 EMMC Flashing

- EMMC Flashing Process

The process for flashing the EMMC is outlined below. The initial part covers the transition from BROM-Fastboot to U-Boot Fastboot mode, which is similar for other media as well.

```sh
fastboot stage factory/FSBL.bin
fastboot continue

#sleep to wait for uboot ready
#linux env
sleep 1
#windows env
#timeout /t 1 >null   

fastboot stage u-boot.itb
fastboot continue

fastboot flash gpt partition_universal.json
fastboot flash bootinfo factory/bootinfo_emmc.bin
fastboot flash fsbl factory/FSBL.bin
fastboot flash env env.bin
fastboot flash opensbi fw_dynamic.itb
fastboot flash uboot u-boot.itb
fastboot flash bootfs bootfs.img
fastboot flash rootfs rootfs.ext4
```

The content of `bootinfo_emmc.bin` has no practical effect; please refer to FAQ section 6.4. However, the flashing step still needs to be executed.

For EMMC, the content of `bootinfo_emmc.bin` is embedded within the U-Boot code. This is done to ensure compatibility with using the same `partition_universal.json` table for SD card booting, among others. Both `bootinfo_emmc.bin` and `FSBL.bin` are actually written to the EMMC's Boot0 partition.

- EMMC Partition Table Configuration
The partition table is stored in `buildroot-ext/board/spacemit/k1/partition_universal.json`. The bootinfo is not explicitly defined as a partition but is used to store information related to the boot medium.

```sh
{
  "version": "1.0",
  "format": "gpt",
  "partitions": [
    {
      "name": "bootinfo",
      "offset": "0",
      "size": "80B",
      "image": "factory/bootinfo_sd.bin"
    },
    {
      "name": "fsbl",
      "offset": "256K",
      "size": "256K",
      "image": "factory/FSBL.bin"
    },
    {
      "name": "env",
      "offset": "512K",
      "size": "128K"
    },
    {
      "name": "opensbi",
      "offset": "1M",
      "size": "1M",
      "image": "opensbi.itb"
    },
    {
      "name": "uboot",
      "offset": "2M",
      "size": "2M",
      "image": "u-boot.itb"
    },
    {
      "name": "bootfs",
      "offset": "4M",
      "size": "128M",
      "image": "bootfs.img"
    },
    {
      "name": "rootfs",
      "size": "-"
    }
  ]
}
```

##### 2.2.1.3 NOR+BLK device Flashing

- NOR+BLK device Flashing process

The K1 supports a combination of NOR + SSD/EMMC for flashing and booting, and it does so in an adaptive manner. If both SSD and EMMC are present, the default flashing target is the SSD.

The firmware tool will write the image based on the actual partition table. Therefore, for NOR + BLK configurations, the partitions such as FSBL (First Stage Boot Loader) and U-Boot on the BLK device do not serve any practical purpose.

```sh
fastboot stage factory/FSBL.bin
fastboot continue
#sleep wait for uboot ready
#linux env
sleep 1                 
#windows env
#timeout /t 1 >null      
fastboot stage u-boot.itb
fastboot continue

#flashing to spi nor
fastboot flash mtd partition_2M.json
fastboot flash bootinfo factory/bootinfo_spinor.bin
fastboot flash fsbl factory/FSBL.bin
fastboot flash env env.bin
fastboot flash opensbi fw_dynamic.itb
fastboot flash uboot u-boot.itb

#flashing to block device
fastboot flash gpt partition_universal.json
fastboot flash bootfs bootfs.img
fastboot flash rootfs rootfs.ext4
```

- NOR + BLK Partition Table Configuration
The NOR partition table, for example, `buildroot-ext/board/spacemit/k1/partition_2M.json`. For NAND/NOR devices, the partition tables are named partition_xM.json, and they need to be renamed according to the actual Flash capacity; otherwise, the corresponding partition table will not be found during the flashing process.

The NOR/NAND partition tables will be compatible with the minimum capacity. For instance, if the NOR medium capacity is 8 MB and the flashing package only contains `partition_2M.json`, then it will match the `partition_2M.json` partition table.

For partition table configuration, MTD storage media (NAND/NOR Flash) are represented in terms of capacity such as `partition_2M.json`, while BLK devices (including eMMC/SD/SSD, etc.) are all named `partition_universal.json`.

Modifications to the NOR Flash Partition Table:

1. The starting address and size of partitions are aligned by default to 64 KB (corresponding to an erase size of 64 KB).

2. If the starting address and size need to be changed to align with 4 KB, the U-Boot compilation configuration `CONFIG_SPI_FLASH_USE_4K_SECTORS` must be enabled.

```sh
//buildroot-ext/board/spacemit/k1/partition_2M.json
{
  "version": "1.0",
  "format": "mtd",
  "partitions": [
    {
      "name": "bootinfo",
      "offset": "0",
      "size": "128K",
      "image": "factory/bootinfo_spinor.bin"
    },
    {
      "name": "fsbl",
      "offset": "128K",
      "size": "256K",
      "image": "factory/FSBL.bin"
    },
    {
      "name": "env",
      "offset": "384K",
      "size": "64K"
    },
    {
      "name": "opensbi",
      "offset": "448K",
      "size": "192K",
      "image": "fw_dynamic.itb"
    },
    {
      "name": "uboot",
      "offset": "640K",
      "size": "-",
      "image": "u-boot.itb"
    }
  ]
}
```

SSD Partition Table. For the partition table of BLK devices, it is always `partition_universal.json`. In this case, partitions such as bootinfo, fsbl, env, opensbi, uboot, and the data within them, will not affect normal boot processes.

```sh
//buildroot-ext/board/spacemit/k1/partition_universal.json
{
    "version": "1.0",
    "format": "gpt",
    "partitions": [
        {
            "name": "bootinfo",
            "offset": "0K",
            "size": "512",
            "image": "factory/bootinfo_sd.bin",
            "holes": "{\"(80;512)\"}"
        },
        {
            "name": "fsbl",
            "offset": "128K",
            "size": "256K",
            "image": "factory/FSBL.bin"
        },
        {
            "name": "env",
            "offset":"384K",
            "size": "64K",
            "image": "env.bin"
        },
        {
            "name": "opensbi",
            "size": "384K",
            "image": "fw_dynamic.itb"
        },
        {
            "name": "uboot",
            "size": "2M",
            "image": "u-boot.itb"
        },
        {
            "name": "bootfs",
            "offset": "4M",
            "size": "256M",
            "image": "bootfs.img",
            "compress": "gzip-5"
        },
        {
            "name": "rootfs",
            "offset": "260M",
            "size": "-",
            "image": "rootfs.ext4",
            "compress": "gzip-5"
        }
    ]
}
```

##### 2.2.1.4 nand Flashing

- NAND Flash Programming Process

The K1 supports NAND flash booting. However, since the NAND and NOR share the same SPI interface, only one of these schemes can be selected, with the default support being for NOR booting.

If you need to support NAND flash booting, please refer to the NAND boot configuration section in the programming boot process to configure NAND booting

```sh
fastboot stage factory/FSBL.bin
fastboot continue
#sleep wait for uboot ready
#linux env
sleep 1       
#windows env          
#timeout /t 1 >null      
fastboot stage u-boot.itb
fastboot continue

fastboot flash mtd partition_64M.json
fastboot flash bootinfo factory/bootinfo_spinand.bin
fastboot flash fsbl factory/FSBL.bin
fastboot flash env env.bin
fastboot flash opensbi fw_dynamic.itb
fastboot flash uboot u-boot.itb

fastboot flash user-bootfs bootfs.img
fastboot flash user-rootfs rootfs.img
```

File system management on the NAND relies on UBIFS. The methods for creating the bootfs and rootfs partitions are as follows:

```sh
#Creating bootfs.img
mkfs.ubifs -F -m 2048 -e 124KiB -c 8124 -x zlib -o output/bootfs.img -d input_bootfs/

#Creating rootfs.img
mkfs.ubifs -F -m 2048 -e 124KiB -c 8124 -x zlib -o output/rootfs.img -d input_rootfs/

#input_bootfs和input_rootfs应该存放bootfs和rootfs中的文件
# For example, for bootfs, include the following files: Image.itb, env_k1-x.txt, k1-x.bmp

#Different NANDs require modification of the corresponding parameters; here are some parameter descriptions:
-m 2048：Sets the minimum input/output unit size to 2048 bytes (2 KiB), which should match the page size of the NAND Flash.
-e 124KiB：Sets the logical erase block size to 124 KiB (kilobytes), which should be smaller than the actual physical erase block size to leave space for UBI management structures.
-c 8124：Sets the maximum number of logical erase blocks to 8124, which limits the maximum size of the filesystem based on the size and number of logical erase blocks.
-x zlib: Sets the compression type to zlib, which will compress the data in the filesystem using zlib.
-o output/ubifs.img: Specifies the output file name, saving the generated UBIFS filesystem image as output/ubifs.img.
-d input/: Specifies the source directory as input/, where mkfs.ubifs will create a UBIFS filesystem image from the files and directory structure in this directory.
```

Additionally, this platform also supports the NAND + BLK device boot method. The flashing commands can be referenced from the NOR + BLK device flashing section.

- NAND Partition Table Configuration

For a 256MB capacity NAND, the partition table is partition_256M.json.

```sh
//buildroot-ext/board/spacemit/k1/partition_64M.json
{
  "version": "1.0",
  "format": "mtd",
  "partitions": [
    {
      "name": "bootinfo",
      "comment": "private partition, should not overrided it",
      "offset": "0",
      "size": "128K",
      "image": "factory/bootinfo_spinand.bin"
    },
    {
      "name": "fsbl",
      "offset": "128K",
      "size": "256K",
      "image": "factory/FSBL.bin"
    },
    {
      "name": "env",
      "offset": "384K",
      "size": "128K"
    },
    {
      "name": "opensbi",
      "offset": "640K",
      "size": "384K",
      "image": "opensbi.itb"
    },
    {
      "name": "uboot",
      "offset": "1M",
      "size": "2M",
      "image": "u-boot.itb"
    },
    {
      "name": "user",
      "offset": "3M",
      "size": "-",
             "volume_images": {"bootfs": "bootfs.img", "rootfs": "rootfs.img"}
    }
  ]
}
```

##### 2.2.1.5 Card Booting

Card booting refers to writing the image to an SD card. When the device is powered on with this SD card inserted, the system will boot from the SD card.

The K1 platform supports card booting and defaults to first attempting to boot from the SD card; if that fails, it will then boot from the storage medium selected by pin selection.

This platform does not support using the fastboot protocol to flash images onto the SD card. Instead, you need to use TitanFlash tool or the `dd` command to create a card bootable image.

Below is a description of how to create a card bootable image using the flashing tool:

- i. Insert the TF card into a card reader and connect it to the computer's USB port.

- ii. On the computer, open the TitanFlash flashing tool (installation instructions can be found in the flashing tool usage section). Click on Development Tools -> Card Booting, select the SD card and the firmware package. Then click Execute.

- iii. After the flashing is successful, insert the TF card into the device. Upon powering on, the device will boot from the card.

![alt text](static/flash_tool_1.png)

##### 2.2.1.6 Card Flashing

Card Flashing refers to the process of writing firmware-related image files to the SD card. When the device is powered on and boots from the SD card, it detects the Card Flashing mode and writes the image files stored on the SD card to the storage medium selected via pin configuration according to the partition table.

The K1 supports mass production flashing. By using the TitanFlash tool, you can turn an SD card into a mass production card. Once this SD card is inserted into the device and the device is powered on, the image will be flashed onto the eMMC, NOR, NAND, or other storage media.

The steps for creating a mass production card are as follows:

- Insert the TF card into a card reader and connect it to the computer's USB port.
- Open the TitanFlash tool, select 'Mass Production Tool' -> 'Create Mass Production Card', choose the image for mass production, and click 'Execute'.
- After the mass production card has been created, insert the SD card into the device. Upon powering on, the device will automatically enter the flash mode.
- After the flashing is completed, remove the SD card. The device should then power on and start up normally. (If the SD card is not removed, powering on again will restart the mass production process.)

![alt text](static/flash_tool_2.png)

#### 2.2.2 Flashing Tools

This section provides a brief introduction to the flashing tools used in the firmware update process.

##### 2.2.2.1 Usage of Flashing Tools

For firmware flashing, you can choose to install the TitanFlash tool or use the Fastboot tool. For generating firmware, refer to the document titled "Download and Build" at this link [Download and Build](https://bianbu-linux.spacemit.com/en/download_and_build/).

- TitanFlash Tool
 This tool is designed for full firmware packages and is suitable for general developers. Installation and usage instructions for the TitanFlash tool can be found at [this](https://developer.spacemit.com/documentation?token=O6wlwlXcoiBZUikVNh2cczhin5d)

- Fastboot Tool
This tool is used for flashing individual partition images and is suitable for developers with a certain level of expertise. Incorrect flashing of single partitions may lead to system boot issues, so caution is advised. The flashing procedure is detailed in the flashing process chapter. Instructions for setting up a Fastboot environment can be found at [link](https://doc.e.foundation/pages/install-adb-windows) or [link](https://www.linuxbabe.com/ubuntu/how-to-install-adb-fastboot-ubuntu).

##### 2.2.2.2 Selection of Flashing Media

- The hardware provides a boot download selection switch (Boot Download Sel switch). Refer to the Boot Download Sel & JTAG Sel section in the [MUSE Pi User Guide](https://developer.spacemit.com/documentation?token=ZugWwIVmkiGNAik55hzc4C3Ln6d)
- Other Solutions: Refer to the hardware user guide for specific solutions.
- The flashing process is designed to adapt automatically to different boot media.

### 2.3 Boot Configuration

This section describes the boot configurations for eMMC, SD, NOR, and NAND, along with important considerations for custom boot configurations. Successful firmware flashing requires correct partition tables and boot configurations.

#### 2.3.1 Boot Process

This section outlines the boot process for the platform chip and provides methods for customizing the boot process. The boot process consists of several stages: brom -> fsbl -> opensbi -> U-Boot -> kernel. The bootinfo provides information about the offset, size, and signature of the fsbl.

![alt text](static/boot_proc.png)

Based on the boot pin selection, the software loads the next-level image from the SD, eMMC, or NOR storage medium. The boot process is consistent across different boot media, as shown in the diagram above. Specific boot pin selections are described in the section on selection of flashing media.

Upon power-on, the K1 first attempts to boot from the SD card; if this fails, it then proceeds based on the boot pin selection to load the U-Boot and OpenSBI images from the eMMC, NAND, or NOR storage media.

Subsequent sections will detail the boot configurations for bootinfo, fsbl, OpenSBI, U-Boot, and the kernel, in order of the boot sequence.

##### 2.3.1.1 brom

Brom is a pre-compiled program embedded within the SoC that cannot be changed once the device is manufactured. Upon power-up or reset, the chip runs the brom, which, depending on the boot pin configuration, instructs the brom to load the bootinfo and fsbl from the corresponding storage medium. For details on boot pins, please refer to the section on selection of flashing media.

##### 2.3.1.2 bootinfo

After the brom initializes, it retrieves bootinfo from the physical address offset 0 of the corresponding storage medium (for eMMC, this would be from the boot0 region's address 0). The bootinfo contains information such as the location and size of fsbl and fsbl1, along with their CRC values. The bootinfo* descriptor files are located in the following directory:

```sh
uboot-2022.10$ ls board/spacemit/k1-x/configs/
bootinfo_emmc.json  bootinfo_sd.json  bootinfo_spinand.json  bootinfo_spinor.json
```

Here is an example of what these JSON files contain, showing the default locations of fsbl and fsbl1 for various storage media. Fsbl1 serves as a backup. If you need to change the offset addresses, you can modify the `bootinfo_* JSON` files and recompile U-Boot.

The location of fsbl should generally remain as the original factory settings. If customization is required, consider the characteristics of different storage media, such as the need for sector alignment in NAND. For eMMC, fsbl can only be loaded from the boot partition.

```sh
emmc:boot0:0x200, boot1:0x0
sd:0x20000/0x80000
nor:0x20000/0x70000
nand:0x20000/0x80000
```

When compiling U-Boot, the corresponding `*.bin` files and bootinfo image files are generated in the root directory. The generation method can be referenced in `uboot-2022.10/board/spacemit/k1-x/config.mk`.

```sh
uboot-2022.10$ ls -l
bootinfo_emmc.bin
bootinfo_sd.bin
bootinfo_spinand.bin
bootinfo_spinor.bin
```

##### 2.3.1.3 fsbl

After obtaining the bootinfo, brom will load the fsbl from the specified offset.

The fsbl file consists of a 4KB header plus the uboot-spl.bin binary. This header includes essential information for uboot-spl, such as the CRC checksum.

Once fsbl is launched, it first initializes the DDR memory. Then, it loads opensbi and uboot into designated memory locations from the partition table, followed by running opensbi, which in turn launches uboot. (The specifics of how fsbl initializes DDR are beyond the scope of this document.)

##### 2.3.1.4 Loading and Starting opensbi and uboot

The methods for loading opensbi and uboot vary across different storage media. Below, we describe the configurations for SPL (Second Program Loader) loading and starting uboot/opensbi based on the storage medium used.

The K1 platform supports loading standalone partitions for opensbi and uboot, as well as loading them as a single image (`uboot-opensbi.itb`). By default, they are loaded by partition names. For loading a single image (`uboot-opensbi.itb`), refer to the FAQ section.

###### 2.3.1.4.1 eMMC and SD Card

Both eMMC and SD cards use the MMC driver. The U-Boot code flow selects between eMMC and SD based on the boot pin configuration. The startup configurations for both are similar.

Opensbi and uboot can have different image formats, including raw partitions, files, and FIT (Flattened Image Tree) format, depending on their storage location.
By default, the K1 platform first attempts to boot from the SD card; if this fails, it tries other media (eMMC, NOR, NAND, etc.).

SPL-DTS needs to enable the configuration for eMMC/SD.

```sh
//uboot-2022.10/arch/riscv/dts/k1-x_spl.dts
        /* eMMC */
        sdh@d4281000 {
                bus-width = <8>;
                non-removable;
                mmc-hs400-1_8v;
                mmc-hs400-enhanced-strobe;
                sdh-phy-module = <1>;
                status = "okay";
                u-boot,dm-spl;
        };
        
        /* sd */
      sdh@d4280000 {
          pinctrl-names = "default";
          pinctrl-0 = <&pinctrl_mmc1 &gpio80_pmx_func0>;
          bus-width = <4>;
          cap-sd-highspeed;
          sdh-phy-module = <0>;
          status = "okay";
          u-boot,dm-spl;
      };
```

The following introduces different boot methods for eMMC/SD, with a default approach of loading by partition name.

- Raw Partition Method

1.fFSBL Loading and Starting U-Boot/OpenSBI Process (Using eMMC as an example; for SD/NOR, etc., the process is similar but not elaborated here due to space constraints)

Execute `make uboot_menuconfig` to enter the SPL compilation configuration (this is the entry point for all SPL-related configurations; subsequent steps are not detailed).

![alt text](static/spl-config_1.png)

Select loading via the partition table. Here, separate loading of U-Boot/OpenSBI is supported by default. SPL will first look for partitions named 'opensbi' and 'uboot'. If these partition names are not found, it will attempt to load raw data from partition numbers 1 and 2 respectively.

![alt text](static/spl-config_2.png)

![alt text](static/spl-config_3.png)

After SPL successfully loads OpenSBI and U-Boot, it starts OpenSBI and passes the memory address of U-Boot along with the DTB (Device Tree Blob) information. OpenSBI then boots U-Boot according to the provided address and DTB.

- Absolute Offset

Load OpenSBI/U-Boot using absolute offset addresses.
Enable the following configuration to specify the offset address for eMMC. In this mode, the images of U-Boot/OpenSBI are loaded according to the absolute offset address of the eMMC. This method does not support separate loading of OpenSBI and U-Boot; instead, they need to be packaged together in FIT format. If you want to support independent loading, you can refer to the code segment in `case MMCSD_MODE_RAW` within `uboot-2022.10/common/spl/spl_mmc.c`.

![alt text](static/spl-config_4.png)

- File System Method

Modify the SPL to load OpenSBI/U-Boot through the file system.
U-Boot-SPL supports loading OpenSBI and U-Boot via the file system, allowing independent loading of U-Boot and OpenSBI files.
Use make menuconfig to enable the following configuration, as shown in the image below, select the FAT file system to load `opensbi.itb` and `uboot.itb.`

![alt text](static/spl-config_5.png)

###### 2.3.1.4.2 nor

For booting from NOR media, K1 provides booting options of NOR (U-Boot-SPL/U-Boot/OpenSBI) + SSD (bootfs/rootfs) or NOR (U-Boot-SPL/U-Boot/OpenSBI) + eMMC (bootfs/rootfs), with a default preference for trying the SSD first.

The DTS (Device Tree Source) configuration for SPL is as follows:

```sh
//uboot-2022.10/arch/riscv/dts/k1-x_spl.dts
        spi@d420c000 {
                status = "okay";
                pinctrl-names = "default";
                pinctrl-0 = <&pinctrl_qspi>;
                u-boot,dm-spl;

                spi-max-frequency = <15140000>;
                flash@0 {
                        compatible = "jedec,spi-nor";
                        reg = <0>;
                        spi-max-frequency = <26500000>;

                        m25p,fast-read;
                        broken-flash-reset;
                        u-boot,dm-spl;
                        status = "okay";
                };
        };
```

The SPL supports booting from NOR and requires the following configurations to be enabled first; they are typically enabled by default.

1.Execute `make uboot_menuconfig` and select SPL configuration options.

![alt text](static/spl-config_6.png)

2.Enable “Support MTD drivers”, “Support SPI DM drivers in SPL”, “Support SPI drivers”, “Support SPI flash drivers”, “Support for SPI flash MTD drivers in SPL”, “Support loading from mtd device”. Ensure that the “Partition name to use to load U-Boot from” matches the partition names in the partition table.

![alt text](static/spl-config_7.png)

3.Configuration for Block Devices (blk devices)
Execute `make uboot_menuconfig`, then select Device Drivers ---> Fastboot support. Choose Support blk device, which includes support for devices like SSD/EMMC. For example, SSD corresponds to nvme, and EMMC corresponds to mmc.

![alt text](static/spl-config_8.png)

4.For MTD Devices, environmental variables need to be enabled to ensure that SPL can retrieve MTD partition information after booting.
Execute `make uboot_menuconfig`, then select Environment and enable SPI environment loading. The environment offset address must match the environment partition in the partition table, such as `0x80000`.

![alt text](static/spl-config_9.png)

![alt text](static/spl-config_10.png)

5.SPI Flash Driver Configuration needs to match the manufacturer's model of the SPI flash on the hardware (similarly for NAND). In the menuconfig, select the corresponding manufacturer ID. Execute make uboot_menuconfig, then select Device Drivers ---> MTD Support ---> SPI Flash Support. Choose the appropriate driver based on the SPI flash manufacturer of your hardware.

![alt text](static/spl-config_11.png)
If the required driver is not listed, you can add it directly in the code. flash_name can be customized, usually matching the hardware flash name. `0x1f4501` represents the JEDEC ID of the flash, and other parameters can be added based on the specifications of the hardware flash.

```sh
//uboot-2022.10/drivers/mtd/spi/spi-nor-ids.c
const struct flash_info spi_nor_ids[] = {
     { INFO("flash_name",    0x1f4501, 0, 64 * 1024,  16, SECT_4K) },
```

There are mainly two loading methods, as described above.

- Raw Partition Method

For NOR devices, U-Boot/OpenSBI will be obtained according to the MTD partition table. The configuration method is as follows:

![alt text](static/spl-config_12.png)

- Absolute Offset

NOR boot supports loading via the absolute offset of the storage medium but does not support independently loading U-Boot and OpenSBI. U-Boot and OpenSBI must be packaged together, i.e., in FIT format.
Enable the following configuration and input the absolute offset address of the storage medium.

![alt text](static/spl-config_13.png)

###### 2.3.1.4.3 nand

For NAND-based booting, K1 provides booting options using NAND (U-Boot-SPL/U-Boot/OpenSBI) + SSD (bootfs/rootfs) or pure NAND (U-Boot-SPL/U-Boot/OpenSBI/kernel). NAND boot is not enabled by default.

The SPL-DTS configuration is as follows:

```sh
//uboot-2022.10/arch/riscv/dts/k1-x_spl.dts
        spi@d420c000 {
                status = "okay";
                pinctrl-names = "default";
                pinctrl-0 = <&pinctrl_qspi>;
                u-boot,dm-spl;

                spi-max-frequency = <15140000>;
                spi-nand@0 {
                       compatible = "spi-nand";
                       reg = <0>;
                       spi-tx-bus-width = <1>;
                       spi-rx-bus-width = <1>;
                       spi-max-frequency = <6250000>;
                       u-boot,dm-spl;
                       status = "okay";
               };
        };
```

Below is the configuration for the pure NAND boot solution.
Run `make uboot_menuconfig` and select SPL Configuration Options.

![alt text](static/spl-config_14.png)

enable as follow:
`Support MTD drivers`
`Support SPI DM drivers in SPL`
`Support SPI drivers`
`Use standard NAND driver`
`Support simple NAND drivers in SPL`
`Support loading from mtd device`
`Partition name to use to load U-Boot from`should match the partition names in the partition table.

If you enable the opensbi/uboot separate image, you need to enable `Second partition to use to load U-Boot from`, and it must maintain the order of opensbi followed by uboot.

![alt text](static/spl-config_15.png)

For MTD (MTD stands for Memory Technology Device) devices, you need to enable the environment settings to ensure that after the SPL (Secondary Program Loader) boots, it can obtain the MTD partition information from the environment. When running make uboot_menuconfig, choose Environment and enable SPI environment loading. The environment offset address here needs to be consistent with the environment partition in the partition table, such as 0x80000.

The NAND flash driver needs to be adapted to the corresponding model of the SPI flash used on the hardware. The currently supported NAND flash drivers are listed below. If there is no corresponding driver, you can add the manufacturer's JEDEC ID (Joint Electron Devices Engineering Council ID) in the other.c driver.

```sh
~/uboot-2022.10$ ls drivers/mtd/nand/spi/*.c
uboot-2022.10/drivers/mtd/nand/spi/core.c
uboot-2022.10/drivers/mtd/nand/spi/micron.c
uboot-2022.10/drivers/mtd/nand/spi/winbond.c
uboot-2022.10/drivers/mtd/nand/spi/gigadevice.c
uboot-2022.10/drivers/mtd/nand/spi/other.c
uboot-2022.10/drivers/mtd/nand/spi/macronix.c
uboot-2022.10/drivers/mtd/nand/spi/toshiba.c


//uboot-2022.10/drivers/mtd/nand/spi/other.c
 static int other_spinand_detect(struct spinand_device *spinand)
 {
     u8 *id = spinand->id.data;
     int ret = 0;

     /*
      * dosilicon nand flash
      */
     if (id[1] == 0xe5)
        ret = spinand_match_and_init(spinand, dosilicon_spinand_table,
                     ARRAY_SIZE(dosilicon_spinand_table),
                     id[2]);

     /*FORESEE nand flash*/
     if (id[1] == 0xcd)
        ret = spinand_match_and_init(spinand, foresee_spinand_table,
                     ARRAY_SIZE(foresee_spinand_table),
                     id[2]);
     if (ret)
         return ret;

     return 1;
 }
```

##### 2.3.1.5 Booting the Kernel

U-Boot's environment (env) defines multiple boot combinations, allowing developers to customize them according to their needs. The `k1-x_env.txt` file provided with the solution has a higher priority and will override variables in env.bin with the same names.

After FSBL (First Stage Boot Loader) loads and boots OpenSBI -> U-Boot, U-Boot uses the load command to load kernel, device tree blob (DTB), and other image files from the FAT or ext4 file system into memory, and then executes the bootm command to start the kernel.

The specific process for loading and booting the kernel can be found in `uboot-2022.10/board/spacemit/k1-x/k1-x.env`. This file includes boot schemes for eMMC/SD/NOR+BLK and automatically adapts to the corresponding boot media.

- MMC Boot

For SD/eMMC, both fall under the category of mmc_boot.

```sh
//uboot-2022.10/board/spacemit/k1-x/k1-x.env
run mmc_boot;

mmc_boot=echo "Try to boot from ${bootfs_devname}${boot_devnum} ..."; \
          run commonargs; \
          run set_mmc_root; \
          run set_mmc_args; \
          run detect_dtb; \
          run loadknl; \
          run loaddtb; \
          run loadramdisk; \
          bootm ${kernel_addr_r} ${ramdisk_combo} ${dtb_addr};
```

The `rootfs` is passed from U-Boot to the kernel via bootargs, where it is parsed by the kernel or the init script and then mounted.

The field `root=/dev/mmcblk2p6` indicates the partition for the rootfs.

```sh
bootargs=earlycon=sbi earlyprintk console=tty1 console=ttyS0,115200 loglevel=8 clk_ignore_unused swiotlb=65536 rdinit=/init root=/dev/mmcblk2p6 rootwait rootfstype=ext4
```

- nor+blk Boot

```sh
//uboot-2022.10/board/spacemit/k1-x/k1-x.env
run nor_boot;

nor_boot=echo "Try to boot from ${bootfs_devname}${boot_devnum} ..."; \
         run commonargs; \
         run set_nor_root; \
         run set_nor_args; \
         run detect_dtb; \
         run loadknl; \
         run loaddtb; \
         run loadramdisk; \
         bootm ${kernel_addr_r} ${ramdisk_combo} ${dtb_addr};
```

- nand Boot

```sh
//uboot-2022.10/board/spacemit/k1-x/k1-x.env
run nand_boot;

//to be adapted
```

### 2.4 Secure Boot

Secure boot is implemented based on the FIT (FDT Image) format:

1. Pack opensbi, uboot.itb, and the kernel into an FIT image.
2. Enable secure boot configuration in the code.
3. Sign the FIT image using a private key and export the public key to the DTB (Device Tree Blob).

#### 2.4.1 Verification Process

The verification process during boot is shown in the following diagram:

![alt text](static/secure_boot.png)

Signing process:

1. The Boot ROM acts as the root of trust and is immutable.
2. The hash of the Root of Trust Public Key (ROTPK) is burned into the chip's internal eFuse, which can only be programmed once.
3. The hash algorithm used is SHA256.
4. The signing algorithm uses SHA256 + RSA2048.

#### 2.4.2 Configuration

- U-Boot compilation configuration:

```sh
CONFIG_FIT_SIGNATURE=y
CONFIG_SPL_FIT_SIGNATURE=y

CONFIG_SHA256=y
CONFIG_SPL_SHA256=y

CONFIG_RSA=y
CONFIG_SPL_RSA=y
CONFIG_SPL_RSA_VERIFY=y
CONFIG_RSA_VERIFY=y
```

- OpenSBI compilation configuration:

```sh
CONFIG_FIT_SIGNATURE=y
```

- Kernel compilation configuration:

```sh
CONFIG_FIT_SIGNATURE=y
```

#### 2.4.3  Public and Private Keys

Generate private keys and certificates using OpenSSL. Protect the private key and distribute the certificate.

```sh
# build private key without password
openssl genrsa -out prv-rsa.key 2048
# private key parse:
openssl rsa -in prv-rsa.key -text -noout

# build certificate that expired after 365 days
openssl req -batch -new -x509 -days 365 -key prv-rsa.key -out rsa.crt
# certificate parse:
openssl x509 -in rsa.crt -text -noout

# build public key from private key:
openssl rsa -in prv-rsa.key -pubout -out pub-rsa.key
# public key parse:
openssl rsa -in pub-rsa.key -pubin -noout -text
```

#### 2.4.4 Image Signing

- Modify the its file to enable hash and signature configurations.

```sh
/dts-v1/;

/ {
        description = "Configuration to load OpenSBI before U-Boot";
        #address-cells = <2>;
        fit,fdt-list = "of-list";

        images {
                opensbi {
                        description = "OpenSBI fw_dynamic Firmware";
                        type = "firmware";
                        os = "opensbi";
                        arch = "riscv";
                        compression = "none";
                        load = <0x0 0x0>;
                        entry = <0x0 0x0>;
                        data = /incbin/("./fw_dynamic.bin");
                        hash-1 {
                                algo = "sha256";
                        };
                };
        };
        configurations {
                default = "config_1";

                config_1 {
                        description = "opensbi FIT config";
                        firmware = "opensbi";
                        signature {
                                algo = "sha256,rsa2048";
                                key-name-hint = "uboot_key_prv";
                                sign-images = "firmware";
                        };
                };
        };
};
```

- Using a private key and certificate, sign the fit image file.

```sh
# build empty dtb file, for next stage public key file output
printf "/dts-v1/;\n/ {\n};" > pubkey.dts
dtc -I dts -O dtb -o pubkey.dtb pubkey.dts

# build fit image
# input: fit script, folder contain private key and certification
# output: fit image, dtb that contain public key info
mkimage -f uboot_fdt_sign.its -K pubkey.dtb -k key -r u-boot.itb

# parse dtb file that public key info
fdtdump -s pubkey.dtb
```

- Update the public key information to the parent bootloader code.

  - For example, update the public key information corresponding to the private key used for U-Boot signing to the FSBL DTS.

```sh
/ {
        signature {
                key-uboot_key_prv {
                        required = "conf";
                        algo = "sha256,rsa2048";
                        rsa,r-squared = <0x5353bc86 0x7070d595 0xe2ea6280 0xb9887ae1 0xf69bb145 0x161e6675 0x6f9d37dc 0x29646b18 0x0ecc66d1 0x0ef7fa25 0xddc925cf 0xf068e5e4 0x78e5b40b 0x124095c6 0x1282d13c 0x1bdf09d0 0x7ddf7bf4 0xb4e61d0b 0x8d68f15d 0xb77282df 0xb0b371d8 0xd887288d 0x6c2ee06e 0x4124c030 0xbcdb8688 0x13a6ea0a 0xbb8dc9d1 0xd4b8a0fd 0x141c1e45 0x91c77190 0xf2685d1e 0xa44e33eb 0x38a90bdf 0x671b076b 0x0efb5223 0x72762fd2 0xcbf35219 0x833553c7 0x91382847 0xa3806134 0xb785d6f6 0x64ba98d7 0x4f01bc2e 0x78e320dc 0x9233332c 0x8be5ebec 0x60605d78 0xd5e5741c 0x2980546e 0x6332d458 0x73023036 0xb5e64449 0xc3f81911 0xc7d57cad 0xf17d98b1 0x139801a2 0x778632bd 0xfc15d9ca 0x4f5fc152 0xa49e2b4f 0x6f09a6b5 0xecd52030 0x19022428 0x5907c874>;
                        rsa,modulus = <0xaa282eab 0xc7d0a288 0x5eee2ea1 0xd7d11bc5 0xaf57d029 0x4ad6c85f 0xedc802b1 0x227775cc 0x0d57d3de 0xc8e6113c 0xd3c238fd 0x03eecd4c 0x6983e4e0 0xd71eba6b 0xcdcc3c7f 0x6f602163 0x71e25d7e 0xd3ade9b9 0x25c9b950 0x4bf4d0a5 0xa067ca9c 0x64397ed2 0xd07dfa01 0x29102b9c 0x6008c40e 0xc55cc431 0xf3422d16 0xb8ade9d2 0xa8e5d3d1 0x40aca443 0x91603617 0x4159c91f 0xa10e3ef9 0xa21c40c7 0x377dfcc6 0xd831b829 0xd645d1b1 0xb04c534e 0xfd3352ef 0xdfe19a7d 0xf90c4295 0x7e753266 0x398ade75 0x85427a33 0x79412712 0x5dcd236d 0x015d8fb6 0xdde963ad 0xb8730cf5 0x45fc281b 0x1e40a1de 0xcd1d2af6 0x45ce6740 0x42e1e705 0x274af16a 0x50a66381 0xbb815c44 0x5222fe56 0x826e4475 0xd2193598 0x967573fd 0xc814bed6 0x95db8fae 0xe519808f>;
                        rsa,exponent = <0x00000000 0x00010001>;
                        rsa,n0-inverse = <0xfba86191>;
                        rsa,num-bits = <0x00000800>;
                        key-name-hint = "uboot_key_prv";
                };
        };
};
```

## 3. U-Boot Features and Configuration

This chapter primarily covers the functions of U-Boot and common methods for configuring it.

### 3.1  Function Introduction

The main functions of U-Boot include:

- Loading the Boot Kernel

U-Boot loads the kernel image from storage media (such as eMMC, SD, NAND, NOR, SSD, etc.) into a designated memory location and then boots the kernel.

- Fastboot Firmware Update

Using the fastboot tool, images can be flashed to specific partition locations.

- Boot Logo Display

During the boot process, U-Boot displays the boot logo and the boot menu.

- Driver Debugging

Device drivers can be debugged using U-Boot, such as those for MMC, SPI, NAND, NOR, NVMe, etc. U-Boot provides a shell command line interface for testing the functionalities of various drivers. The U-Boot drivers are located in the `drivers/`directory.

### 3.2 Compilation

This section describes the process of building U-Boot image files based on the U-Boot code environment.

- Compilation Configuration

For the initial compilation or when switching to a different configuration, you first need to select the appropriate build configuration. Here, we use the k1 configuration as an example:

```shell
cd ~/uboot-2022.10
make ARCH=riscv k1_defconfig -C ~/uboot-2022.10/
```

To modify the build configuration interactively, you can use the following command:

```shell
make ARCH=riscv menuconfig
```

![a](static/OLIdbiLK4onXqyxOOj8cyBDCn3b.png)

Use the keys "Y" or "N" to enable or disable the relevant feature configurations. Once you have finished configuring and save your changes, they will be updated in the .config file located in the root directory of the U-Boot project.

- Compiling U-Boot

```shell
cd ~/uboot-2022.10
GCC_PREFIX=riscv64-unknown-linux-gnu-
make ARCH=riscv CROSS_COMPILE=${GCC_PREFIX} -C ~/uboot-2022.10 -j4
```

- compilation output

```shell
~/uboot-2022.10$ ls u-boot* -l
u-boot
u-boot.bin           # U-Boot image
u-boot.dtb           # DTB file
u-boot-dtb.bin       # U-Boot image with DTB
u-boot.itb           # Packs u-boot-nodtb.bin and DTB into FIT format
u-boot-nodtb.bin
bootinfo_emmc.bin    # Records SPL location for eMMC boot
bootinfo_sd.bin
bootinfo_spinand.bin
bootinfo_spinor.bin
FSBL.bin             # u-boot-spl.bin with header information, loaded and started by BROM
k1-x_deb1.dtb        # DTB file for the deb1 solution
k1-x_spl.dtb         # SPL DTB file
```

### 3.3 DTS Configuration

U-Boot DTS configuration is located in the directory uboot-2022.10/arch/riscv/dts/. Modify the DTS for the specific solution, such as the deb1 solution.

```shell
~/uboot-2022.10$ ls arch/riscv/dts/k1*.dts -l
arch/riscv/dts/k1-x_deb1.dts
arch/riscv/dts/k1-x_deb2.dts
arch/riscv/dts/k1-x_evb.dts
arch/riscv/dts/k1-x_fpga_1x4.dts
arch/riscv/dts/k1-x_fpga_2x2.dts
arch/riscv/dts/k1-x_fpga.dts
arch/riscv/dts/k1-x_spl.dts
```

## 4. U-Boot Driver Development and Debugging

This section primarily covers the usage and debugging methods of U-Boot drivers. By default, all drivers are already configured.

### 4.1 Booting the Kernel

This subsection describes how to boot the kernel using U-Boot, along with custom configurations and booting procedures for partitions.

- After powering on the development board, immediately press the "s" key on the keyboard to enter the U-Boot shell.
- You can enter fastboot mode by executing `fastboot 0`, and then use a PC's `fastboot stage Image` command to send the image to the development board. (Alternatively, you can use other methods to download images, such as the `fatload` command.)
- Execute `booti` to boot the kernel (or `bootm` to boot an FIT format image).

```shell
#download kernel image
=> fastboot -l 0x40000000 0
Starting download of 50687488 bytes
...
downloading/uploading of 50687488 bytes finished

# PC's command
C:\Users>fastboot stage Z:\k1\output\Image
Sending 'Z:\k1\output\Image' (49499 KB)           OKAY [  1.934s]
Finished. Total time: 3.358s

#After downloading is complete, to exit fastboot mode, press CTRL+C in the U-Boot shell using the keyboard.

#download dtb
=> fastboot -l 0x50000000 0
Starting download of 33261 bytes

downloading/uploading of 33261 bytes finished

# PC's command
C:\Users>fastboot stage Z:\k1\output\k1-x_deb1.dtb
Sending 'Z:\k1\output\k1-x_deb1.dtb' (32 KB)      OKAY [  0.004s]
Finished. Total time: 0.054s
```

excute boot command

```shell
=> booti 0x40000000 - 0x50000000
Moving Image from 0x40000000 to 0x200000, end=3d4f000
## Flattened Device Tree blob at 50000000
   Booting using the fdt blob at 0x50000000
   Using Device Tree in place at 0000000050000000, end 0000000050014896

Starting kernel ...

[    0.000000] Linux version 6.1.15+ ... ...
[    0.000000] OF: fdt: Ignoring memory range 0x0 - 0x200000
[    0.000000] Machine model: spacemit k1-x deb1 board
[    0.000000] earlycon: sbi0 at I/O port 0x0 (options '')
[    0.000000] printk: bootconsole [sbi0] enabled
```

- Booting a FIT Format Image Using the bootm Command

Assuming that partition 5 of the eMMC is a FAT32 filesystem and contains a file named uImage.itb, you can load and boot the kernel using the following commands:

```shell
=> fatls mmc 2:5
sdh@d4281000: 74 clk wait timeout(100)
 50896911   uImage.itb
     4671   env_k1-x.txt

2 file(s), 0 dir(s)

=> fatload mmc 2:5 0x40000000 uImage.itb
50896911 bytes read in 339 ms (143.2 MiB/s)
=> bootm 0x40000000
## Loading kernel from FIT Image at 40000000 ...
Boot from fit configuration k1_deb1
   Using 'conf_2' configuration
   Trying 'kernel' kernel subimage
     Description:  Vanilla Linux kernel
     Type:         Kernel Image
     Compression:  uncompressed
     Data Start:   0x400000e8
     Data Size:    50687488 Bytes = 48.3 MiB
     Architecture: RISC-V
     OS:           Linux
     Load Address: 0x01400000
     Entry Point:  0x01400000
   Verifying Hash Integrity ... OK
## Loading fdt from FIT Image at 40000000 ...
   Using 'conf_2' configuration
   Trying 'fdt_2' fdt subimage
     Description:  Flattened Device Tree blob for k1_deb1
     Type:         Flat Device Tree
     Compression:  uncompressed
     Data Start:   0x43067c90
     Data Size:    68940 Bytes = 67.3 KiB
     Architecture: RISC-V
     Load Address: 0x28000000
   Verifying Hash Integrity ... OK
   Loading fdt from 0x43067c90 to 0x28000000
   Booting using the fdt blob at 0x28000000
   Loading Kernel Image
   Using Device Tree in place at 0000000028000000, end 0000000028013d4b

Starting kernel ...

[    0.000000] Linux version 6.1.15+ ... ...
[    0.000000] OF: fdt: Ignoring memory range 0x0 - 0x1400000
[    0.000000] Machine model: spacemit k1-x deb1 board
[    0.000000] earlycon: sbi0 at I/O port 0x0 (options '')
[    0.000000] printk: bootconsole [sbi0] enabled
```

### 4.2 env

This chapter introduces how to configure the loading of the environment from a specified storage medium during the U-Boot boot stage.

- Run `make menuconfig` and go to the Environment section.

![a](static/CgrNbzNbkot1tvxOXIhcGMrRnvc.png)

![a](static/Od7AbhfLSoHWY9xN8uIcwlAhnhb.png)

The currently supported optional media include MMC (MultiMediaCard) and MTD (Memory Technology Device) devices, which further include SPI NOR (spinor) and SPI NAND (spinand).

The offset address for the environment needs to be determined based on the partition table configuration. For more details, refer to the partition table configuration in the flashing and boot settings section. The default offset is 0x80000.

```shell
(0x80000) Environment address       # Offset address for the spinor environment
(0x80000) Environment offset        # Offset address for the MMC device environment
```

### 4.3 mmc

Both eMMC (embedded MultiMediaCard) and SD cards use the MMC driver, with device index being 2 and 0 respectively.

- Configuration in the config file

Run `make menuconfig`, then navigate to Device Drivers --> MMC Host Controller Support, and enable the following configurations:

![a](static/YnF5beU32ojicYx6xbkcM2pGn2b.png)

- DTS (Device Tree Source) Configuration

```c
//uboot-2022.10/arch/riscv/dts/k1-x.dtsi
         sdhci0: sdh@d4280000 {
             compatible = "spacemit,k1-x-sdhci";
             reg = <0x0 0xd4280000 0x0 0x200>;
             interrupt-parent = <&intc>;
             interrupts = <99>;
             resets = <&reset RESET_SDH_AXI>,
                      <&reset RESET_SDH0>;
             reset-names = "sdh_axi", "sdh0";
             clocks = <&ccu CLK_SDH0>,
                      <&ccu CLK_SDH_AXI>;
             clock-names = "sdh-io", "sdh-core";
             status = "disabled";
         };

         sdhci2: sdh@d4281000 {
             compatible = "spacemit,k1-x-sdhci";
             reg = <0x0 0xd4281000 0x0 0x200>;
             interrupt-parent = <&intc>;
             interrupts = <101>;
             resets = <&reset RESET_SDH_AXI>,
                      <&reset RESET_SDH2>;
             reset-names = "sdh_axi", "sdh2";
             clocks = <&ccu CLK_SDH2>,
                      <&ccu CLK_SDH_AXI>;
             clock-names = "sdh-io", "sdh-core";
             status = "disabled";
         };

//uboot-2022.10/arch/riscv/dts/k1-x_deb1.dts
&sdhci0 {
        pinctrl-names = "default";
        pinctrl-0 = <&pinctrl_mmc1 &gpio80_pmx_func0>;
        bus-width = <4>;
        cd-gpios = <&gpio 80 0>;
        cd-inverted;
        cap-sd-highspeed;
        sdh-phy-module = <0>;
        status = "okay";
};

/* eMMC */
&sdhci2 {
        bus-width = <8>;
        non-removable;
        mmc-hs400-1_8v;
        mmc-hs400-enhanced-strobe;
        sdh-phy-module = <1>;
        status = "okay";
};
```

- Debugging and Verification

The U-Boot shell provides a command-line interface for debugging the MMC driver, which requires enabling the compile-time configuration option `CONFIG_CMD_MMC`.

```shell
=> mmc list
sdh@d4280000: 0 (SD)
sdh@d4281000: 2 (eMMC)
=> mmc dev 2 #Switch to eMMC
switch to partitions #0, OK
mmc2(part 0) is current device

#Read 0x1000 blocks from offset 0 into memory address 0x40000000
=> mmc read 0x40000000 0 0x1000

MMC read: dev # 2, block # 0, count 4096 ... 4096 blocks read: OK

#Write 0x1000 blocks from memory address 0x40000000 to offset 0
=> mmc write 0x40000000 0 0x1000

MMC write: dev # 2, block # 0, count 4096 ... 4096 blocks written: OK

#For other usage options, refer to mmc -h
```

- Common Interfaces

Refer to the interfaces in cmd/mmc.c for common operations related to the MMC driver.

### 4.4 nvme

The nvme driver is primarily used for debugging SSD.

- Configuration in the config file

Run `make menuconfig`, go into Device Driver, and enable the following configurations:

![a](static/OLktbqlRLoreIPxlZ9TcGtwOnff.png)

![a](static/UrVybSqdFo0iTnxZ8QAcKoWAnqc.png)

- DTS (Device Tree Source) Configuration

```c
//uboot-2022.10/arch/riscv/dts/k1-x.dtsi
         pcie1_rc: pcie@ca400000 {
             compatible = "k1x,dwc-pcie";
             reg = <0x0 0xca400000 0x0 0x00001000>, /* dbi */
                   <0x0 0xca700000 0x0 0x0001ff24>, /* atu registers */
                   <0x0 0x90000000 0x0 0x00100000>, /* config space */
                   <0x0 0xd4282bd4 0x0 0x00000008>, /*k1x soc config addr*/
                   <0x0 0xc0c20000 0x0 0x00001000>, /* phy ahb */
                   <0x0 0xc0c10000 0x0 0x00001000>, /* phy addr */
                   <0x0 0xd4282bcc 0x0 0x00000008>, /* conf0 addr */
                   <0x0 0xc0b10000 0x0 0x00001000>; /* phy0 addr */
             reg-names = "dbi", "atu", "config", 
                            "k1x_conf", "phy_ahb", 
                            "phy_addr", "conf0_addr", 
                            "phy0_addr";

             k1x,pcie-port = <1>;
             clocks = <&ccu CLK_PCIE1>;
             clock-names = "pcie-clk";
             resets = <&reset RESET_PCIE1>;
             reset-names = "pcie-reset";

             bus-range = <0x00 0xff>;
             max-link-speed = <2>;
             num-lanes = <2>;
             num-viewport = <8>;
             device_type = "pci";
             #address-cells = <3>;
             #size-cells = <2>;
             ranges = <0x01000000 0x0 0x90100000 
                          0 0x90100000 0x0 0x100000>,
                  <0x02000000 0x0 0x90200000 
                      0 0x90200000 0x0 0x0fe00000>;

             interrupts = <142>, <146>;
             interrupt-parent = <&intc>;
             #interrupt-cells = <1>;
             interrupt-map-mask = <0 0 0 0x7>;
             interrupt-map = <0000 0 0 1 &pcie1_intc 1>, /* int_a */
                     <0000 0 0 2 &pcie1_intc 2>, /* int_b */
                     <0000 0 0 3 &pcie1_intc 3>, /* int_c */
                     <0000 0 0 4 &pcie1_intc 4>; /* int_d */
             linux,pci-domain = <1>;
             status = "disabled";
             pcie1_intc: interrupt-controller@0 {
                 interrupt-controller;
                 reg = <0 0 0 0 0>;
                 #address-cells = <0>;
                 #interrupt-cells = <1>;
             };
         };


//uboot-2022.10/arch/riscv/dts/k1-x_deb1.dts
 &pcie1_rc {
     pinctrl-names = "default";
     pinctrl-0 = <&pinctrl_pcie1_3>;
     status = "okay";
 };
```

- Debugging and Verification

Need to enable configuration `CONFIG_CMD_NVME`, follow these steps for debugging purposes:

```shell
=> nvme scan
=> nvme detail
Blk device 0: Optional Admin Command Support:
        Namespace Management/Attachment: no
        Firmware Commit/Image download: yes
        Format NVM: yes
        Security Send/Receive: yes
Blk device 0: Optional NVM Command Support:
        Reservation: yes
        Save/Select field in the Set/Get features: yes
        Write Zeroes: yes
        Dataset Management: yes
        Write Uncorrectable: yes
Blk device 0: Format NVM Attributes:
        Support Cryptographic Erase: No
        Support erase a particular namespace: Yes
        Support format a particular namespace: Yes
Blk device 0: LBA Format Support:
        LBA Foramt 0 Support: (current)
                Metadata Size: 0
                LBA Data Size: 512
                Relative Performance: Good
Blk device 0: End-to-End DataProtect Capabilities:
        As last eight bytes: No
        As first eight bytes: No
        Support Type3: No
        Support Type2: No
        Support Type1: No
Blk device 0: Metadata capabilities:
        As part of a separate buffer: No
        As part of an extended data LBA: No
=> nvme read/write addr blk_off blk_cnt
```

- Common Interfaces

Refer to the interfaces in `cmd/nvme.c` for common operations related to the nvme driver.

### 4.5 net

- Configuration in the config file

Run `make menuconfig`, go into Device Driver, and enable the following configurations:

![a](static/RCZdbLULLo7I0axEo3rc71BdnZg.png)

![a](static/K5s8bumbzofb0txqmiXc43BCnRg.png)

- DTS (Device Tree Source) Configuration

```c
//uboot-2022.10/arch/riscv/dts/k1-x.dtsi
         eth0: ethernet@cac80000 {
             compatible = "spacemit,k1x-emac";
             reg = <0x00000000 0xCAC80000 0x00000000 0x00000420>;
             ctrl-reg = <0x3e4>;
             dline-reg = <0x3e8>;
             clocks = <&ccu CLK_EMAC0_BUS>;
             clock-names = "emac-clk";
             resets = <&reset RESET_EMAC0>;
             reset-names = "emac-reset";
             status = "disabled";
         };

//uboot-2022.10/arch/riscv/dts/k1-x_deb1.dts
 &eth0 {
     status = "okay";
     pinctrl-names = "default";
     pinctrl-0 = <&pinctrl_gmac0>;

     phy-reset-pin = <110>;

     clk_tuning_enable;
     clk-tuning-by-delayline;
     tx-phase = <90>;
     rx-phase = <73>;

     phy-mode = "rgmii";
     phy-addr = <1>;
     phy-handle = <&rgmii>;

     ref-clock-from-phy;

     mdio {
         #address-cells = <0x1>;
         #size-cells = <0x0>;
         rgmii: phy@0 {
             compatible = "ethernet-phy-id001c.c916";
             device_type = "ethernet-phy";
             reg = <0x1>;
         };
     };
 };
```

- Debugging and Verification

You need to first enable the compilation configuration CONFIG_CMD_NET. Connect the Ethernet cable to the development board’s network port, and ensure that a TFTP server is set up (you can search online for instructions on how to set up a TFTP server; this will not be covered here).

```shell
=> dhcp #After executing dhcp, if an address is returned, it indicates a successful connection to the network server. Any other situation indicates a connection failure.
ethernet@cac80000 Waiting for PHY auto negotiation to complete...... done
emac_adjust_link link:1 speed:1000 duplex:full
BOOTP broadcast 1
BOOTP broadcast 2
BOOTP broadcast 3
BOOTP broadcast 4
BOOTP broadcast 5
BOOTP broadcast 6
BOOTP broadcast 7
DHCP client bound to address 10.0.92.130 (7982 ms)

=> tftpboot 0x40000000 site11/uImage.itb
ethernet@cac80000 Waiting for PHY auto negotiation to complete...... done
emac_adjust_link link:1 speed:1000 duplex:full
Using ethernet@cac80000 device
TFTP from server 10.0.92.134; our IP address is 10.0.92.130
Filename 'site11/uImage.itb'.
Load address: 0x40000000
Loading: ##############################################################
         ########
         1.1 MiB/s
done
Bytes transferred = 66900963 (3fcd3e3 hex)
=>

# booting kernel
=>bootm 0x40000000 
```

- Common Interfaces

You can refer to the code interface in cmd/net.c for guidance.

### 4.6 spi

spi only exposes one hardware interface, so it only supports NAND or NOR flash.

- Configuration in the config file

Run `make menuconfig`, go into Device Driver, and enable the following configurations:

![a](static/DdHBbRJQpoIoopxMuO8cS3GBnXg.png)

![a](static/AH6bbloZ9omZNux2ZCxcXblVnvc.png)

- DTS (Device Tree Source) Configuration

```c
//k1-x.dtsi
/dts-v1/;

/ {
        compatible = "spacemit,k1x", "riscv";
        #address-cells = <2>;
        #size-cells = <2>;

        soc:soc {
                compatible = "simple-bus";
                #address-cells = <2>;
                #size-cells = <2>;
                ranges;

                qspi: spi@d420c000 {
                        compatible = "spacemit,k1x-qspi";
                        #address-cells = <1>;
                        #size-cells = <0>;
                        reg = <0x0 0xd420c000 0x0 0x1000>,
                              <0x0 0xb8000000 0x0 0xd00000>;
                        reg-names = "qspi-base", "qspi-mmap";
                        qspi-sfa1ad = <0xa00000>;
                        qspi-sfa2ad = <0xb00000>;
                        qspi-sfb1ad = <0xc00000>;
                        qspi-sfb2ad = <0xd00000>;
                        clocks = <&ccu CLK_QSPI>,
                                <&ccu CLK_QSPI_BUS>;
                        clock-names = "qspi_clk", "qspi_bus_clk";
                        resets = <&reset RESET_QSPI>,
                                <&reset RESET_QSPI_BUS>;
                        reset-names = "qspi_reset", "qspi_bus_reset";
                        qspi-pmuap-reg = <0xd4282860>;
                        spi-max-frequency = <26500000>;
                        qspi-id = <4>;
                        status = "disabled";
                };
        };
};

//k1-x_deb1.dts
&qspi {
        status = "okay";
        pinctrl-names = "default";
        pinctrl-0 = <&pinctrl_qspi>;
};
```

- Debugging and Verification

To enable the U-Boot shell command `sspi`, you need to configure `CONFIG_CMD_SPI`.

debug command

```c
sspi -h

    "SPI utility command",
    "[<bus>:]<cs>[.<mode>][@<freq>] <bit_len> <dout> - Send and receive bits\n"
    "<bus>     - Identifies the SPI bus\n"
    "<cs>      - Identifies the chip select\n"
    "<mode>    - Identifies the SPI mode to use\n"
    "<freq>    - Identifies the SPI bus frequency in Hz\n"
    "<bit_len> - Number of bits to send (base 10)\n"
    "<dout>    - Hexadecimal string that gets sent"

```

- Common Interfaces

You can refer to the code interface in cmd/spi.c for guidance.

### 4.7 nand

The NAND driver is based on SPI, so it's necessary to first enable the SPI driver functionality.

- Configuration in the config file

Run `make menuconfig`, then navigate to Device Drivers -> MTD Support.

![a](static/Pmlobv86koO6qpxDohMcycGVn4e.png)

If you need to add a new NAND flash, you can do so by adding the JEDEC ID of the NAND flash according to the supported vendor drivers.

```shell
~/uboot-2022.10$ ls drivers/mtd/nand/spi/*.c -l
drivers/mtd/nand/spi/core.c
drivers/mtd/nand/spi/gigadevice.c
drivers/mtd/nand/spi/macronix.c
drivers/mtd/nand/spi/micron.c
drivers/mtd/nand/spi/other.c
drivers/mtd/nand/spi/toshiba.c
drivers/mtd/nand/spi/winbond.c
```

To add a new flash for a Gigadevice chip, follow these steps:

```c
//uboot-2022.10/drivers/mtd/nand/spi/gigadevice.c
 static const struct spinand_info gigadevice_spinand_table[] = {
     SPINAND_INFO("GD5F1GQ4UExxG", 0xd1,
              NAND_MEMORG(1, 2048, 128, 64, 1024, 1, 1, 1),
              NAND_ECCREQ(8, 512),
              SPINAND_INFO_OP_VARIANTS(&gd5fxgq4_read_cache_variants,
                           &write_cache_variants,
                           &update_cache_variants),
              0,
              SPINAND_ECCINFO(&gd5fxgqxxexxg_ooblayout,
                      gd5fxgq4xexxg_ecc_get_status)),
     SPINAND_INFO("GD5F1GQ5UExxG", 0x51,
              NAND_MEMORG(1, 2048, 128, 64, 1024, 1, 1, 1),
              NAND_ECCREQ(4, 512),
              SPINAND_INFO_OP_VARIANTS(&gd5f1gq5_read_cache_variants,
                           &write_cache_variants,
                           &update_cache_variants),
              0,
              SPINAND_ECCINFO(&gd5fxgqxxexxg_ooblayout,
                      gd5fxgq5xexxg_ecc_get_status)),
 };
```

If you need to implement support for NAND Flash of other brands and plan to refer to the Gigadevice driver for this purpose

- DTS (Device Tree Source) Configuration

NAND driver is attached to the SPI driver, so the DTS needs to be configured under the SPI node.

```c
 &qspi {
     status = "okay";
     pinctrl-names = "default";
     pinctrl-0 = <&pinctrl_qspi>;

     spi-nand@0 {
         compatible = "spi-nand";
         reg = <0>;
         spi-tx-bus-width = <1>;
         spi-rx-bus-width = <1>;
         spi-max-frequency = <6250000>;
         u-boot,dm-spl;
         status = "okay";
     };
 };
```

- Debugging and Verification

The NAND driver can be debugged using MTD commands.

```shell
=> mtd
mtd - MTD utils

Usage:
mtd - generic operations on memory technology devices

mtd list
mtd read[.raw][.oob]                  <name> <addr> [<off> [<size>]]
mtd dump[.raw][.oob]                  <name>        [<off> [<size>]]
mtd write[.raw][.oob][.dontskipff]    <name> <addr> [<off> [<size>]]
mtd erase[.dontskipbad]               <name>        [<off> [<size>]]

Specific functions:
mtd bad                               <name>

With:
        <name>: NAND partition/chip name (or corresponding DM device name or OF path)
        <addr>: user address from/to which data will be retrieved/stored
        <off>: offset in <name> in bytes (default: start of the part)
                * must be block-aligned for erase
                * must be page-aligned otherwise
        <size>: length of the operation in bytes (default: the entire device)
                * must be a multiple of a block for erase
                * must be a multiple of a page otherwise (special case: default is a page with dump)

The .dontskipff option forces writing empty pages, don't use it if unsure.

=> mtd list
[RESET]spacemit_reset_set assert=1, id=77
[RESET]spacemit_reset_set assert=1, id=78
clk qspi_bus_clk already disabled
clk qspi_clk already disabled
ccu_mix_set_rate of qspi_clk timeout
[RESET]spacemit_reset_set assert=0, id=77
[RESET]spacemit_reset_set assert=0, id=78
SF: Detected w25q32 with page size 256 Bytes, erase size 64 KiB, total 4 MiB
Could not find a valid device for spi-nand
List of MTD devices:
* nor0
  - device: flash@0
  - parent: spi@d420c000
  - driver: jedec_spi_nor
  - path: /soc/spi@d420c000/flash@0
  - type: NOR flash
  - block size: 0x10000 bytes
  - min I/O: 0x1 bytes
  - 0x000000000000-0x000000400000 : "nor0"
          - 0x0000000a0000-0x000000100000 : "opensbi"
          - 0x000000100000-0x000000300000 : "uboot"
=> mtd read/write partname addr off size
```

- Common Interfaces

You can refer to the code interface in cmd/mtd.c for guidance.

### 4.8 nor

The NOR driver is based on SPI, so it's necessary to first enable the SPI driver functionality.

- Configuration in the config file

Execute `make menuconfig`, navigate to `Device Drivers ---> MTD Support ---> SPI Flash Support`, and enable the following configurations (they are usually enabled by default). This example uses enabling the Winbond NOR flash as an illustration.

![a](static/WkhTbAHpFot5raxYWMWckwBwnsh.png)

![a](static/VTT0bxjO1oobWYxPeficMzw4nfl.png)

Adding a new SPI NOR flash:

- For NOR flashes already supported by the above-mentioned vendors, You can directly enable the corresponding build configurations, such as those for NOR flashes from the Gigadevice manufacturer.
- The JEDEC ID list for SPI flashes is maintained in u-boot-2022.10/drivers/mtd/spi/spi-nor-ids.c. If a specific NOR flash’s JEDEC ID is not in the list, you can add the JEDEC ID yourself. The JEDEC ID corresponds to the manufacturer code of the SPI flash. You can find the manufacturer code (such as 0xfe for Winbond) in the NOR flash datasheet by searching for the keyword manufacturer.
- DTS Configuration

The NOR driver depends on the SPI driver interface. For SPI driver configuration, refer to the SPI subchapter. You need to add a DTS node as follows:

```c
//k1/uboot-2022.10/arch/riscv/dts/k1-x_deb1.dts
 &qspi {
     status = "okay";
     pinctrl-names = "default";
     pinctrl-0 = <&pinctrl_qspi>;

     flash@0 {
         compatible = "jedec,spi-nor";
         reg = <0>;
         spi-max-frequency = <26500000>;
         m25p,fast-read;
         broken-flash-reset;
         status = "okay";
     };
 };
```

- Debugging and Verification

Debugging can be performed using the mtd and `sf` commands in the U-Boot command line. The build configuration needs to have `CONFIG_CMD_MTD=y` and `CONFIG_CMD_SF` enabled.

Reading and writing to the NOR flash using the mtd commands:

```shell
=> mtd list
List of MTD devices:
* nor0
  - device: flash@0
  - parent: spi@d420c000
  - driver: jedec_spi_nor
  - path: /soc/spi@d420c000/flash@0
  - type: NOR flash
  - block size: 0x1000 bytes
  - min I/O: 0x1 bytes
  - 0x000000000000-0x000000400000 : "nor0"
          - 0x0000000a0000-0x0000000e0000 : "opensbi"
          - 0x000000100000-0x000000200000 : "uboot"
=>

=> mtd
mtd - MTD utils

Usage:
mtd - generic operations on memory technology devices

mtd list
mtd read[.raw][.oob]                  <name> <addr> [<off> [<size>]]
mtd dump[.raw][.oob]                  <name>        [<off> [<size>]]
mtd write[.raw][.oob][.dontskipff]    <name> <addr> [<off> [<size>]]
mtd erase[.dontskipbad]               <name>        [<off> [<size>]]

Specific functions:
mtd bad                               <name>

With:
        <name>: NAND partition/chip name (or corresponding DM device name or OF path)
        <addr>: user address from/to which data will be retrieved/stored
        <off>: offset in <name> in bytes (default: start of the part)
                * must be block-aligned for erase
                * must be page-aligned otherwise
        <size>: length of the operation in bytes (default: the entire device)
                * must be a multiple of a block for erase
                * must be a multiple of a page otherwise (special case: default is a page with dump)

The .dontskipff option forces writing empty pages, don't use it if unsure.

=>
=> mtd read uboot 0x40000000
Reading 1048576 byte(s) at offset 0x00000000
=> mtd dump uboot 0 0x10
Reading 16 byte(s) at offset 0x00000000

Dump 16 data bytes from 0x0:
0x00000000:     d0 0d fe ed 00 0d e8 95  00 00 00 38 00 0d e4 44
=>
```

Reading from and writing to NOR Flash using the `sf` command:

```shell
=> sf
sf - SPI flash sub-system

Usage:
sf probe [[bus:]cs] [hz] [mode] - init flash device on given SPI bus
                                  and chip select
sf read addr offset|partition len       - read `len' bytes starting at
                                          `offset' or from start of mtd
                                          `partition'to memory at `addr'
sf write addr offset|partition len      - write `len' bytes from memory
                                          at `addr' to flash at `offset'
                                          or to start of mtd `partition'
sf erase offset|partition [+]len        - erase `len' bytes from `offset'
                                          or from start of mtd `partition'
                                         `+len' round up `len' to block size
sf update addr offset|partition len     - erase and write `len' bytes from memory
                                          at `addr' to flash at `offset'
                                          or to start of mtd `partition'
sf protect lock/unlock sector len       - protect/unprotect 'len' bytes starting
                                          at address 'sector'
=> sf probe
SF: Detected w25q32 with page size 256 Bytes, erase size 4 KiB, total 4 MiB
=> sf read 0x40000000 0 0x10
device 0 offset 0x0, size 0x10
SF: 16 bytes @ 0x0 Read: OK
=>
```

- Common Interfaces

```c
include <spi.h>
#include <spi_flash.h>

struct udevice *new, *bus_dev;
int ret;
static struct spi_flash *flash;

//bus,cs对应spi的bus和cs编号，如0，0
ret = spi_find_bus_and_cs(bus, cs, &bus_dev, &new);
flash = spi_flash_probe(bus, cs, speed, mode);

ret = spi_flash_read(flash, offset, len, buf);
```

### 4.9 hdmi

This section primarily focuses on how to enable the HDMI (High-Definition Multimedia Interface) driver.

- Configuration in the config file

Execute `make uboot_menuconfig`, then go to `Device Drivers -> Graphics support`, and make sure the following configurations are enabled (they should be enabled by default).

![a](static/GeszbbETBojI9KxyCWBcPM7fnHe.png)

![a](static/MXYNbqJwjoNsdhxsBT2clnTSn1e.png)

![a](static/MX60b8b2uoLDLaxHlJlcTyc7nte.png)

![a](static/Sm8hbLmawoxfMdxrMlBcJMVInHd.png)

![a](static/NuSSbshdfon2mWxWZU6cEipvnwf.png)

- DTS (Device Tree Source) Configuration

```c
&dpu {
        status = "okay";
};

&hdmi {
        pinctrl-names = "default";
        pinctrl-0 = <&pinctrl_hdmi_0>;
        status = "okay";
};
```

### 4.10 boot logo

This section primarily describes how to display the boot logo during the U-Boot boot process.

- Configuration in the config file

Run `make menuconfig` and enable the following configurations.

1. First, enable HDMI support in U-Boot, refer to section hidm.
2. Then, enable boot logo support in U-Boot. Go to `Device Drivers -> Graphics support` and enable the following option.

![a](static/FfzObuq4poT5ZYxU17scAzZRnyf.png)

- env Configuration

To add the three necessary environment variables for the bootlogo—splashimage, splashpos, and splashfile—into the U-Boot configuration, you would modify the file located at uboot-2022.10/include/configs/k1-x.h.

```c
//uboot-2022.10/include/configs/k1-x.h
    ... ...
    ... ...
 #define CONFIG_EXTRA_ENV_SETTINGS \
        ... ...
        ... ...
     "splashimage=" __stringify(CONFIG_FASTBOOT_BUF_ADDR) "\0" \
     "splashpos=m,m\0" \
     "splashfile=k1-x.bmp\0" \

        ... ...
        ... ...
     BOOTENV_DEVICE_CONFIG \
     BOOTENV

 #endif /* __CONFIG_H */
```

Among them, `splashimage` represents the address in memory where the bootlogo image is loaded;

`splashpos` represents the position where the image is displayed, "m,m" means the image is displayed in the center of the screen;

`splashfile` refers to the filename of the BMP file that will be displayed; this image needs to be placed in the partition where the bootfs resides.

- Packaging the .bmp image into the bootfs

To package the BMP image that will be displayed into the bootfs:

Place the `k1-x.bmp` file in the `./buildroot-ext/board/spacemit/k1` directory. The filename should match the one specified in `UBOOT_LOGO_FILE` within `buildroot-ext/board/spacemit/k1/prepare_img.sh`, as well as the environment variable splashfile, such as `k1-x.bmp`.

After compilation and packaging, the BMP image will be included in the bootfs.

```shell
//buildroot-ext/board/spacemit/k1/prepare_img.sh
#!/bin/bash

######################## Prepare sub-iamges and pack ####################
#$0 is this file path
#$1 is buildroot output images dir

IMGS_DIR=$1
DEVICE_DIR=$(dirname $0)

... ...

UBOOT_LOGO_FILE="$DEVICE_DIR/k1-x.bmp"

```

- How to Modify the Boot Logo

1. Directly replace the `k1-x.bmp` file in the `buildroot-ext/board/spacemit/k1/` directory, or add a new image according to the description above.

### 4.11 boot menu

This section mainly introduces how to enable the U-Boot bootmenu feature.

- DTS (Device Tree Source) Configuration

Run `make menuconfig`, go to `Command line interface > Boot commands`, and enable the following configuration:

![a](static/BmycbCac2oCtuGxjjpvcUlHunug.png)

Then go to `Boot options > Autoboot options` and enable the following options:

![a](static/UEhPbxaIgoXpv8xBh52cn7Vdndb.png)

- env Configuration

In `buildroot-ext/board/spacemit/k1/env_k1-x.txt`, you need to add bootdelay and bootmenu_delay, for example, `bootdelay=5`, `bootmenu_delay=5`. Here, 5 represents the waiting time for the bootmenu, measured in seconds.

```c
//buildroot-ext/board/spacemit/k1/env_k1-x.txt
bootdelay=5

# Boot menu definitions
boot_default=echo "Current Boot Device: ${boot_device}"
flash_default=echo "Returning to Boot Menu..."
spacemit_flashing_usb=echo "recovery from usb...... "; \
                      spacemit_flashing usb;
spacemit_flashing_mmc=echo "recovery from mmc...... " \
                      spacemit_flashing mmc;
spacemit_flashing_net=echo "recovery from net...... " \
                      spacemit_flashing net;
bootmenu_delay=5
bootmenu_0="-------- Boot Options --------"=run boot_default
bootmenu_1="Boot from Nor"=run nor_boot
bootmenu_2="Boot from Nand"=run nand_boot
bootmenu_3="Boot from MMC"=run try_mmc
bootmenu_4="Autoboot"=run autoboot
bootmenu_5="Show current Boot Device"=run boot_default
bootmenu_6="-------- Flash Options --------"=run flash_default
bootmenu_7="recovery from usb"=run spacemit_flashing_usb
bootmenu_8="recovery from mmc"=run spacemit_flashing_mmc
bootmenu_9="recovery from net"=run spacemit_flashing_net
```

- Enter the bootmenu

Press and hold the Esc key on the keyboard after power-on to enter the bootmenu.

### 4.12 fastboot command

This section mainly introduces the fastboot commands supported by the k1-deb1 solution.

- Configuration in the config file

Run `make menuconfig`, go to `Device Drivers ---> Fastboot support`, and enable the following compilation configurations:

![a](static/LrxMbKM9Eoioc9xJo9bcUrA2nnb.png)

Fastboot depends on the USB driver, so you need to enable USB support:

![a](static/DmeEbYiPqoUuW9xa8u8cw9hlnPg.png)

![a](static/MuMabzRykoQeWHxZk1UcNDZVnDg.png)

- enter the fastboot mode

1.You can enter the U-Boot shell by pressing the 's' key after the system boots up, and then execute `fastboot 0` to enter fastboot mode.

The default fastboot buffer address/size is defined by the macros `CONFIG_FASTBOOT_BUF_ADDR`/`CONFIG_FASTBOOT_BUF_SIZE`.

```shell
#or fastboot -l 0x30000000 -s 0x10000000 0，Specify the buffer address and size.
=> fastboot 0
k1xci_udc: phy_init
k1xci_udc probe
k1xci_udc: pullup 1
-- suspend --
handle setup GET_DESCRIPTOR, 0x80, 0x6 index 0x0 value 0x100 length 0x40
handle setup SET_ADDRESS, 0x0, 0x5 index 0x0 value 0x22 length 0x0
handle setup GET_DESCRIPTOR, 0x80, 0x6 index 0x0 value 0x100 length 0x12
..
```

2.After the device boots into Bianbu OS, send the `adb reboot bootloader` command to restart the system into fastboot mode.(某些方案可能不支持)

- Supported fastboot commands

For configuring the fastboot environment on your computer, please refer to the chapter on installing the computer environment.

```shell
#Native fastboot protocol commands
fastboot devices              # Display available devices
fastboot reboot               # Restart the device
fastboot getvar [version/product/serialno/max-download-size]
fastboot flash partname image # Flash the image to the partname partition
fastboot erase partname       # Erase the partname partition
fastboot stage file           # Download the file to the buffer address

# OEM vendor-specific commands and functionalities
fastboot getvar [mtd-size/blk-size] # Get the size of mtd/blk devices; return NULL if not available
fastboot oem read part              # Read data from part to buffer address
fastboot get_staged file            # Upload data and name it file. Depends on the oem read part command
```

### 4.13 Files System

- fat

```shell
=> fat
  fatinfo fatload fatls fatmkdir fatrm fatsize fatwrite
=> fatls mmc 2:5
 50896911   uImage.itb
     4671   env_k1-x.txt

2 file(s), 0 dir(s)

=> fatload mmc 2:5 0x40000000 uImage.itb #load uImage.itb到0x40000000
50896911 bytes read in 339 ms (143.2 MiB/s)
=>
```

- ext4

similar to fat command.

```shell
=> ext4
  ext4load ext4ls ext4size
=> ext4load
ext4load - load binary file from a Ext4 filesystem

Usage:
ext4load <interface> [<dev[:part]> [addr [filename [bytes [pos]]]]]
    - load binary file 'filename' from 'dev' on 'interface'
      to address 'addr' from ext4 filesystem
=>
```

### 4.14 Common U-Boot Commands

Common Commands/Tools

```shell
printenv  - print environment variables
md        - memory display
mw        - memory write (fill)
fdt       - flattened device tree utility commands



help      - print command description/usage
```

- fdt

The `fdt` command is primarily used to print the contents of DTS (Device Tree Source), such as the DTB (Device Tree Blob) file loaded after U-Boot starts.

```shell
=> fdt
fdt - flattened device tree utility commands

Usage:
fdt addr [-c] [-q] <addr> [<size>]  - Set the [control] fdt location to <addr>
fdt move   <fdt> <newaddr> <length> - Copy the fdt to <addr> and make it active
fdt resize [<extrasize>]            - Resize fdt to size + padding to 4k addr + some optional <extrasize> if needed
fdt print  <path> [<prop>]          - Recursive print starting at <path>
fdt list   <path> [<prop>]          - Print one level starting at <path>
fdt get value <var> <path> <prop> [<index>] - Get <property> and store in <var>
                                      In case of stringlist property, use optional <index>
                                      to select string within the stringlist. Default is 0.
fdt get name <var> <path> <index>   - Get name of node <index> and store in <var>
fdt get addr <var> <path> <prop>    - Get start address of <property> and store in <var>
fdt get size <var> <path> [<prop>]  - Get size of [<property>] or num nodes and store in <var>
fdt set    <path> <prop> [<val>]    - Set <property> [to <val>]
fdt mknode <path> <node>            - Create a new node after <path>
fdt rm     <path> [<prop>]          - Delete the node or <property>
fdt header [get <var> <member>]     - Display header info
                                      get - get header member <member> and store it in <var>
fdt bootcpu <id>                    - Set boot cpuid
fdt memory <addr> <size>            - Add/Update memory node
fdt rsvmem print                    - Show current mem reserves
fdt rsvmem add <addr> <size>        - Add a mem reserve
fdt rsvmem delete <index>           - Delete a mem reserves
fdt chosen [<start> <size>]         - Add/update the /chosen branch in the tree
                                        <start>/<size> - initrd start addr/size
NOTE: Dereference aliases by omitting the leading '/', e.g. fdt print ethernet0.
=>

=> fdt addr $fdtcontroladdr
=> fdt print
/ {
        compatible = "spacemit,k1x", "riscv";
        #address-cells = <0x00000002>;
        #size-cells = <0x00000002>;
        model = "spacemit k1-x deb1 board";
... ...
        memory@0 {
                device_type = "memory";
                reg = <0x00000000 0x00000000 0x00000000 0x80000000>;
        };
        chosen {
                bootargs = "earlycon=sbi console=ttyS0,115200 debug loglevel=8,initcall_debug=1 rdinit=/init.tmp";
                stdout-path = "serial0:115200n8";
        };
};

=> fdt print /chosen
chosen {
        bootargs = "earlycon=sbi console=ttyS0,115200 debug loglevel=8,initcall_debug=1 rdinit=/init.tmp";
        stdout-path = "serial0:115200n8";
};
=>
```

- shell command

U-Boot supports shell-style commands such as `if/fi,` `echo`, etc.

```shell
=> if test ${boot_device} = nand; then echo "nand boot"; else echo "not nand boot";fi
not nand boot
=> printenv boot_device
boot_device=nor
=> if test ${boot_device} = nor; then echo "nor boot"; else echo "not nor boot";fi
nor boot
=>
```

## 5. OpensBI Functionality and Configuration

This section introduces the compilation configuration for OpensBI.

## 5.1 OpensBI Compilation

```shell
cd ~/opensbi/

#Note: The toolchain used for compilation must be provided by spacemit. Toolchains not provided by spacemit may cause compilation issues.
GCC_PREFIX=riscv64-unknown-linux-gnu-

CROSS_COMPILE=${GCC_PREFIX} PLATFORM=generic \
PLATFORM_DEFCONFIG=k1-x_deb1_defconfig \
PLATFORM_RISCV_ISA=rv64gc \
FW_TEXT_START=0x0  \
make
```

### 5.2 Generated Compilation Files

```shell
~/opensbi$ ls build/platform/generic/firmware/ -l
fw_dynamic.bin     # Dynamic image that passes configuration parameters during the jump
fw_dynamic.elf
fw_dynamic.elf.dep
fw_dynamic.itb     # fw_dynamic.bin packed into FIT format
fw_dynamic.its
fw_dynamic.o
fw_jump.bin       # Jump image that only performs a jump
fw_payload.bin    # Payload image that includes the U-Boot image
```

### 5.3 OpensBI Functionality Configuration

You can enable or disable certain features by executing menuconfig:

```shell
make PLATFORM=generic PLATFORM_DEFCONFIG=k1-x_deb1_defconfig menuconfig
```

![a](static/JdMVb4GyioYNhMxUOhHc8R3enEd.png)

## 6. FAQ

This chapter introduces common issues and their solutions, as well as common debugging methods and frequently encountered problems.

### 6.1 Device Not Detected When Using TitanFlash to Flash Firmware

Ensure that the USB cable is connected to the computer, and the serial port prints as shown below:
![alt text](static/flash_tool_3.png)

If the device is still not detected, check the device manager to see if an ADB device exists. If not, refer to the fastboot environment installation chapter in the computer setup section.
![alt text](static/flash_tool_4.png)

### 6.2 Changes to def_config Do Not Take Effect After Updating Code

Execute `make menuconfig` to update the .config file. The changes will take effect only after recompilation.

### 6.3 Merging U-Boot and OpensBI for Loading

The SDK design separates U-Boot and OpensBI loading. Developers can merge the two based on their requirements. Follow these steps.

- SPL Boot Configuration

1.In U-Boot, remove the configuration for the second partition and uncheck `Second partition to use to load U-Boot from.`
2.Change the partition name to opensbi-uboot and recompile U-Boot. (Note: The partition name is opensbi-uboot)

![alt text](static/spl-config_16.png)

- Generate uboot-opensbi.itb

Create the `uboot-opensbi.its` file with the following content:

```sh
/dts-v1/;

/ {
        description = "U-boot FIT image for k1x";
        #address-cells = <2>;
        fit,fdt-list = "of-list";

        images {
                uboot {
                        description = "U-Boot";
                        type = "standalone";
                        os = "U-Boot";
                        arch = "riscv";
                        compression = "none";
                        load = <0x0 0x00200000>;
                        data = /incbin/("./u-boot-nodtb.bin");
                };

                opensbi {
                        description = "OpenSBI fw_dynamic Firmware";
                        type = "firmware";
                        os = "opensbi";
                        arch = "riscv";
                        compression = "none";
                        load = <0x0 0x0>;
                        entry = <0x0 0x0>;
                        data = /incbin/("./fw_dynamic.bin");
                };
                fdt_14 {
                        description = "k1-x_MUSE-Card";
                        type = "flat_dt";
                        compression = "none";
                        data = /incbin/("./uboot/k1-x_MUSE-Card.dtb");
                };
        };

        configurations {
                default = "conf_14";
                conf_14 {
                        description = "k1-x_MUSE-Card";
                        firmware = "opensbi";
                        loadables = "uboot";
                        fdt = "fdt_14";
                };
        };
};
```

- Generate ITB File with mkimage

Place the following files in the same directory:
```uboot-opensbi.its```
```u-boot-nodtb.bin```
`fw_dynamic.bin`
`k1-x_MUSE-Card.dtb`（this is the device tree for the solution; modify according to the actual solution name）

Execute the following command to generate the `uboot-opensbi.itb` file:
`uboot-2022.10/tools/mkimage -f uboot-opensbi.its -r u-boot-opensbi.itb`

- Modify Partition Table

Using `partition_universal.json` as an example, delete the U-Boot partition, change the OpensBI partition name to `opensbi-uboot`, and set the partition size to the combined size of both, as follows:

```sh
~$ cat partition_universal.json 
{
  "version": "1.0",
  "format": "gpt",
  "partitions": [
    {
      "name": "bootinfo",
      "offset": "0",
      "size": "80B",
      "image": "factory/bootinfo_sd.bin"
    },
    {
      "name": "fsbl",
      "offset": "128K",
      "size": "256K",
      "image": "factory/FSBL.bin"
    },
    {
      "name": "env",
      "offset": "384K",
      "size": "64K"
    },
    {
      "name": "opensbi-uboot",
      "offset": "1M",
      "size": "3M",
      "image": "u-boot-opensbi.itb"
    },
    {
      "name": "bootfs",
      "offset": "4M",
      "size": "256M",
      "image": "bootfs.img",
      "compress": "gzip-5"
    },
    {
      "name": "rootfs",
      "size": "-"
    }
  ]
}

```

- Update Flashing Commands

Take emmc as an example

```sh
fastboot stage factory/FSBL.bin
fastboot continue
#sleep to wait for uboot ready
#linux env
sleep 1
#windows env
#timeout /t 1 >null   
fastboot stage u-boot-opensbi.itb
fastboot continue

fastboot flash gpt partition_universal.json
#bootinfo_emmc.bin
fastboot flash bootinfo factory/bootinfo_emmc.bin
fastboot flash fsbl factory/FSBL.bin
fastboot flash env env.bin
fastboot flash opensbi-uboot u-boot-opensbi.itb
fastboot flash bootfs bootfs.img
fastboot flash rootfs rootfs.ext4
```

If using the TitanFlasher tool provided by spacemit, change the name u-boot.itb in the fastboot.yaml file to `u-boot-opensbi.itb`.

### 6.4 Define FSBL Location for eMMC Boot

Only eMMC has distinct boot0 and user regions; NOR, NAND, and SD card boot media have only one storage region. The hardware is configured to load bootinfo and FSBL from the boot region, so bootinfo and FSBL cannot be placed in the user region.

During the flashing process, the bootinfo information for eMMC is fixed in the U-Boot code inside the fastboot_oem_flash_bootinfo function. This is done so that the same partition_universal.json partition table can be used for card booting and other purposes.

If you need to modify the FSBL load offset for eMMC, you can directly modify the following code:

```sh
//uboot-2022.10/drivers/fastboot/fb_mmc.c
516 void fastboot_mmc_flash_write(const char *cmd, void *download_buffer,
517                   u32 download_bytes, char *response)
518 {
        ... ...
        
554     if (strcmp(cmd, "bootinfo") == 0) {
555         printf("flash bootinfo\n");
556         fastboot_oem_flash_bootinfo(cmd, fastboot_buf_addr, download_bytes,
557                                     response, fdev);
558         return;
559     }


//uboot-2022.10/drivers/fastboot/fb_spacemit.c
 816 void fastboot_oem_flash_bootinfo(const char *cmd, void *download_buffer,
 817         u32 download_bytes, char *response, struct flash_dev *fdev)
 818 {

 830     /*fill up emmc bootinfo*/
 831     struct boot_parameter_info *boot_info;
 832     boot_info = (struct boot_parameter_info *)download_buffer;
 833     memset(boot_info, 0, sizeof(boot_info));
 834     boot_info->magic_code = BOOT_INFO_EMMC_MAGICCODE;
 835     boot_info->version_number = BOOT_INFO_EMMC_VERSION;
 836     boot_info->page_size = BOOT_INFO_EMMC_PAGESIZE;
 837     boot_info->block_size = BOOT_INFO_EMMC_BLKSIZE;
 838     boot_info->total_size = BOOT_INFO_EMMC_TOTALSIZE;
 839     boot_info->spl0_offset = BOOT_INFO_EMMC_SPL0_OFFSET;
 840     boot_info->spl1_offset = BOOT_INFO_EMMC_SPL1_OFFSET;
 841     boot_info->spl_size_limit = BOOT_INFO_EMMC_LIMIT;
 842     strcpy(boot_info->flash_type, "eMMC");
 843     boot_info->crc32 = crc32_wd(0, (const uchar *)boot_info, 0x40, CHUNKSZ_CRC32);
 844 
 845     /*flash bootinfo*/

```

For eMMC's boot0, the fastboot flashing service performs special handling for the bootinfo/FSBL partition, writing the image file to the boot0 region. Specifically, this can be referenced in the `uboot-2022.10/drivers/fastboot/fb_mmc.c::fastboot_mmc_flash_write` function, under the `if (strcmp(cmd, "bootinfo") == 0)` and `if (strcmp(cmd, CONFIG_FASTBOOT_MMC_BOOT1_NAME) == 0)` branches.

```sh
//uboot-2022.10/drivers/fastboot/fb_mmc.c
void fastboot_mmc_flash_write(const char *cmd, void *download_buffer,
517                   u32 download_bytes, char *response)
518 {

554     if (strcmp(cmd, "bootinfo") == 0) {
555         printf("flash bootinfo\n");
556         fastboot_oem_flash_bootinfo(cmd, fastboot_buf_addr, download_bytes,
557                                     response, fdev);
558         return;
559     }

563 #ifdef CONFIG_FASTBOOT_MMC_BOOT_SUPPORT
564     if (strcmp(cmd, CONFIG_FASTBOOT_MMC_BOOT1_NAME) == 0) {
565         dev_desc = fastboot_mmc_get_dev(response);
566         if (dev_desc){
567 #ifdef CONFIG_SPACEMIT_FLASH
568             flash_mmc_boot_op(dev_desc, download_buffer, 1,
569                     download_bytes, BOOT_INFO_EMMC_SPL0_OFFSET);
570             fastboot_okay(NULL, response);
571 #else
572             fb_mmc_boot_ops(dev_desc, download_buffer, 1,
573                     download_bytes, response);
574 #endif
575         }
576         return;
577     }
```

### 6.5 Will bootinfo_sd.bin at Address 0 Conflict with the GPT Table?

`bootinfo_sd.bin` is placed at address 0 on the SD card and does not conflict with the GPT table (the GPT table is actually located at offset 0x100 after address 0). It is also not recognized as a partition.

### 6.6 How to Set Up Hidden Partitions

The partition table `partition_universal.json` supports hidden partitions. The hidden tag is used to mark hidden partitions. After flashing and booting, hidden partitions do not appear in the GPT table.

Partition tables with hidden partitions require the use of the TitanFlasher tool to flash the image, or executing fastboot flash gpt `partition_universal.json` followed by using fastboot commands to flash the images for hidden partitions.

Currently, only blk devices such as eMMC and SSD support hidden partitions. The bootinfo partition only supports hidden partitions.

A sample partition table with hidden partitions is as follows. Partitions without the hidden tag are considered visible partitions.

`cat k1/common/flash_config/partition_universal.json`

```sh
{
  "version": "1.0",
  "format": "gpt",
  "partitions": [
    {
      "name": "bootinfo",
      "hidden": true,
      "offset": "0",
      "size": "80B",
      "image": "factory/bootinfo_sd.bin"
    },
    {
      "name": "fsbl",
      "hidden": true,
      "offset": "128K",
      "size": "256K",
      "image": "factory/FSBL.bin"
    },
    {
      "name": "env",
      "hidden": true,
      "offset": "384K",
      "size": "64K",
      "image": "u-boot-env-default.bin"
    },
    {
      "name": "opensbi",
      "offset": "1M",
      "size": "1M",
      "image": "opensbi.itb"
    },
    {
      "name": "uboot",
      "offset": "2M",
      "size": "2M",
      "image": "u-boot.itb"
    },
    {
      "name": "bootfs",
      "offset": "4M",
      "size": "256M",
      "image": "bootfs.img",
      "compress": "gzip-5"
    },
    {
      "name": "rootfs",
      "size": "-"
    }
  ]
}
```

For U-Boot versions prior to v1.0.9, patches need to be applied. Save the following patch file, enter the U-Boot repository, apply the patch, and recompile.
`cat support-hidden-partition.patch`

```sh
From 504a003ad33b2dc90749632b4abe05a8fc3a2f21 Mon Sep 17 00:00:00 2001
From: chris <chris.huang@spacemit.com>
Date: Tue, 23 Jul 2024 20:32:54 +0800
Subject: [PATCH] [add]1.support flash hidden partition to blk dev. not support
 for mtd dev currently.

Change-Id: Ic78817a3c84e6e02829376990649b477f3c9248d
---
 drivers/fastboot/fb_blk.c      | 28 ++++++++++++++++++++++++++-
 drivers/fastboot/fb_mmc.c      | 26 +++++++++++++++++++++++++
 drivers/fastboot/fb_spacemit.c | 35 ++++++++++++++++------------------
 include/fb_spacemit.h          | 10 +++++++++-
 4 files changed, 78 insertions(+), 21 deletions(-)

diff --git a/drivers/fastboot/fb_blk.c b/drivers/fastboot/fb_blk.c
index bb756d3f99..11219bc6a6 100644
--- a/drivers/fastboot/fb_blk.c
+++ b/drivers/fastboot/fb_blk.c
@@ -186,6 +186,8 @@ void fastboot_blk_flash_write(const char *cmd, void *download_buffer,
         static char __maybe_unused part_name_t[20] = "";
         unsigned long __maybe_unused src_len = ~0UL;
         bool gzip_image = false;
+        bool is_hidden_part = false;
+        int part_index = 0;
 
         if (fdev == NULL){
                 fdev = malloc(sizeof(struct flash_dev));
@@ -202,6 +204,15 @@ void fastboot_blk_flash_write(const char *cmd, void *download_buffer,
                 printf("init fdev success\n");
         }
 
+        for (part_index = 0; part_index < MAX_PARTITION_NUM; part_index++){
+                if (fdev->parts_info[part_index].part_name != NULL
+                                && strcmp(cmd, fdev->parts_info[part_index].part_name) == 0){
+                        if (fdev->parts_info[part_index].hidden)
+                                is_hidden_part = true;
+                        break;
+                }
+        }
+
         /*blk device would not flash bootinfo except emmc*/
         if (strcmp(cmd, "bootinfo") == 0) {
                 fastboot_okay(NULL, response);
@@ -213,9 +224,24 @@ void fastboot_blk_flash_write(const char *cmd, void *download_buffer,
                                                 response, fdev);
                 return;
         }
+
+        if (is_hidden_part){
+                /*find available blk dev*/
+                do_get_part_info(&dev_desc, cmd, &info);
+                if (!dev_desc){
+                        fastboot_fail("can not get available blk dev", response);
+                        return;
+                }
+
+                strlcpy((char *)&info.name, cmd, sizeof(info.name));
+                info.size = fdev->parts_info[part_index].part_size / dev_desc->blksz;
+                info.start = fdev->parts_info[part_index].part_offset / dev_desc->blksz;
+                info.blksz = dev_desc->blksz;
+                printf("!!! flash image to hidden partition !!!\n");
+        }
 #endif
 
-        if (fastboot_blk_get_part_info(cmd, &dev_desc, &info, response) < 0)
+        if (!is_hidden_part && fastboot_blk_get_part_info(cmd, &dev_desc, &info, response) < 0)
                 return;
 
         if (check_gzip_format((uchar *)download_buffer, src_len) >= 0) {
diff --git a/drivers/fastboot/fb_mmc.c b/drivers/fastboot/fb_mmc.c
index 26e45f4a60..6b417b4ddc 100644
--- a/drivers/fastboot/fb_mmc.c
+++ b/drivers/fastboot/fb_mmc.c
@@ -528,6 +528,8 @@ void fastboot_mmc_flash_write(const char *cmd, void *download_buffer,
         static char __maybe_unused part_name_t[20] = "";
         unsigned long __maybe_unused src_len = ~0UL;
         bool gzip_image = false;
+        bool is_hidden_part = false;
+        int part_index = 0;
 
         if (fdev == NULL){
                 fdev = malloc(sizeof(struct flash_dev));
@@ -544,6 +546,15 @@ void fastboot_mmc_flash_write(const char *cmd, void *download_buffer,
                 printf("init fdev success\n");
         }
 
+        for (part_index = 0; part_index < MAX_PARTITION_NUM; part_index++){
+                if (fdev->parts_info[part_index].part_name != NULL
+                                && strcmp(cmd, fdev->parts_info[part_index].part_name) == 0){
+                        if (fdev->parts_info[part_index].hidden)
+                                is_hidden_part = true;
+                        break;
+                }
+        }
+
         if (strcmp(cmd, "bootinfo") == 0) {
                 printf("flash bootinfo\n");
                 fastboot_oem_flash_bootinfo(cmd, fastboot_buf_addr, download_bytes,
@@ -659,6 +670,21 @@ void fastboot_mmc_flash_write(const char *cmd, void *download_buffer,
         }
 #endif
 
+#ifdef CONFIG_SPACEMIT_FLASH
+        if (is_hidden_part){
+                /*find available blk dev*/
+                dev_desc = fastboot_mmc_get_dev(response);
+                if (!dev_desc)
+                        return;
+
+                strlcpy((char *)&info.name, cmd, sizeof(info.name));
+                info.size        = fdev->parts_info[part_index].part_size / dev_desc->blksz;
+                info.start = fdev->parts_info[part_index].part_offset / dev_desc->blksz;
+                info.blksz        = dev_desc->blksz;
+                printf("!!! flash image to hidden partition !!!\n");
+        }
+#endif
+
         if (!info.name[0] &&
             fastboot_mmc_get_part_info(cmd, &dev_desc, &info, response) < 0)
                 return;
diff --git a/drivers/fastboot/fb_spacemit.c b/drivers/fastboot/fb_spacemit.c
index ab25ddb1ef..3d2ae02b48 100644
--- a/drivers/fastboot/fb_spacemit.c
+++ b/drivers/fastboot/fb_spacemit.c
@@ -110,12 +110,6 @@ int _clear_env_part(void *download_buffer, u32 download_bytes,
 {
         u32 boot_mode = get_boot_pin_select();
 
-        /* char cmdbuf[64] = {"\0"}; */
-        /* sprintf(cmdbuf, "env export -c -s 0x%lx 0x%lx", (ulong)CONFIG_ENV_SIZE, (ulong)download_buffer); */
-        /* if (run_command(cmdbuf, 0)){ */
-        /*         return -1; */
-        /* } */
-
         switch(boot_mode){
 #ifdef CONFIG_ENV_IS_IN_MMC
         case BOOT_MODE_EMMC:
@@ -147,12 +141,6 @@ int _clear_env_part(void *download_buffer, u32 download_bytes,
                         ret = _fb_mtd_erase(mtd, CONFIG_ENV_SIZE);
                         if (ret)
                                 return -1;
-
-                        /*should not write env to env part*/
-                        /* ret = _fb_mtd_write(mtd, download_buffer, 0, CONFIG_ENV_SIZE, NULL); */
-                        /* if (ret){ */
-                        /*         pr_err("can not write env to mtd flash\n"); */
-                        /* } */
                 }
                 break;
 #endif
@@ -208,7 +196,7 @@ int _write_mtd_partition(struct flash_dev *fdev)
  * @brief transfer the string of size 'K' or 'M' to u32 type.
  *
  * @param reserve_size , the string of size
- * @return int , return the transfer result.
+ * @return int , return the transfer result of KB.
  */
 int transfer_string_to_ul(const char *reserve_size)
 {
@@ -303,6 +291,7 @@ int _parse_flash_config(struct flash_dev *fdev, void *load_flash_addr)
                         const char *node_file = NULL;
                         const char *node_offset = NULL;
                         const char *node_size = NULL;
+                        fdev->parts_info[part_index].hidden = false;
 
                         cJSON *arraypart = cJSON_GetArrayItem(cj_parts, i);
                         cJSON *cj_name = cJSON_GetObjectItem(arraypart, "name");
@@ -311,11 +300,12 @@ int _parse_flash_config(struct flash_dev *fdev, void *load_flash_addr)
                         else
                                 node_part = "";
 
-                        /*only blk dev would not add bootinfo partition*/
-                        if (!parse_mtd_partition){
-                                if (strlen(node_part) > 0 && !strncmp("bootinfo", node_part, 8)){
-                                        pr_info("bootinfo would not add as partition\n");
-                                        continue;
+                        cJSON *cj_hidden = cJSON_GetObjectItem(arraypart, "hidden");
+                        if (cj_hidden){
+                                if ((cj_hidden->type == cJSON_String && strcmp("true", cj_hidden->valuestring) == 0)
+                                                || cj_hidden->type == cJSON_True){
+                                        printf("!!!! patr name:%s would set to hidden part !!!!\n", node_part);
+                                        fdev->parts_info[part_index].hidden = true;
                                 }
                         }
 
@@ -374,19 +364,26 @@ int _parse_flash_config(struct flash_dev *fdev, void *load_flash_addr)
                         if (off > 0)
                                 combine_size = off;
 
+                        /*TODO: support hidden partition for mtd dev*/
                         if (parse_mtd_partition){
                                 /*parse mtd partition*/
                                 if (strlen(combine_str) == 0)
                                         sprintf(combine_str, "%s%s@%dK(%s)", combine_str, node_size, combine_size, node_part);
                                 else
                                         sprintf(combine_str, "%s,%s@%dK(%s)", combine_str, node_size, combine_size, node_part);
-                        }else if (fdev->gptinfo.fastboot_flash_gpt){
+                        }else if (!fdev->parts_info[part_index].hidden && fdev->gptinfo.fastboot_flash_gpt){
                                 /*parse gpt partition*/
                                 if (strlen(node_offset) == 0)
                                         sprintf(combine_str, "%sname=%s,size=%s;", combine_str, node_part, node_size);
                                 else
                                         sprintf(combine_str, "%sname=%s,start=%s,size=%s;", combine_str, node_part, node_offset, node_size);
                         }
+
+                        /*save part offset and size to byte*/
+                        fdev->parts_info[part_index].part_offset = combine_size * 1024;
+                        fdev->parts_info[part_index].part_size = transfer_string_to_ul(node_size) * 1024;
+
+                        /*save as the next part offset*/
                         combine_size += transfer_string_to_ul(node_size);
 
                         /*after finish recovery, it would free the malloc paramenter at func recovery_show_result*/
diff --git a/include/fb_spacemit.h b/include/fb_spacemit.h
index 801895d7d4..3f633d30a1 100644
--- a/include/fb_spacemit.h
+++ b/include/fb_spacemit.h
@@ -58,8 +58,16 @@ struct flash_volume_image {
 struct flash_parts_info {
         char *part_name;
         char *file_name;
-        /*partition size info, such as 128MiB*/
+
+        /*save partition size to string*/
         char *size;
+
+        /*partition size info*/
+        u64 part_size;
+
+        /*partition offset info*/
+        u64 part_offset;
+
         /*use for fsbl, if hidden that gpt would reserve a raw memeory
           for fsbl and the partition is not available.
         */
-- 
2.25.1
```
