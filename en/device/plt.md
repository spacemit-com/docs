---
sidebar_position: 9
---

# Product Line Tool

This document describes product line tool.

## Introduction

The product line tool is a system for testing the connectivity of the hardware interface of the board or the whole machine. It is based on bianbu linux and integrated with factorytest application.

The system usually runs on sdcard.

## compilation

```shell
$ cd /path/to/bianbu-linux
$ make envconfig
Available configs in buildroot-ext/configs/:
  1. spacemit_k1_defconfig
  2. spacemit_k1_minimal_defconfig
  3. spacemit_k1_plt_defconfig


your choice (1-3):

```

Select `3` and press Enter to start compiling.

Compiled, you can see：

```shell
Images successfully packed into /path/to/bianbu-linux/output/k1_plt/images/bianbu-linux-k1_plt.zip


Generating sdcard.img...................................
INFO: cmd: "mkdir -p "/path/to/bianbu-linux/output/k1_plt/build/genimage.tmp"" (stderr):
INFO: cmd: "rm -rf "/path/to/bianbu-linux/output/k1_plt/build/genimage.tmp"/*" (stderr):
INFO: cmd: "mkdir -p "/path/to/work/bianbu-linux/output/k1_plt/images"" (stderr):
INFO: hdimage(sdcard.img): adding partition 'bootinfo' from 'factory/bootinfo_sd.bin' ...
INFO: hdimage(sdcard.img): adding partition 'fsbl' (in MBR) from 'factory/FSBL.bin' ...
INFO: hdimage(sdcard.img): adding partition 'env' (in MBR) from 'env.bin' ...
INFO: hdimage(sdcard.img): adding partition 'opensbi' (in MBR) from 'fw_dynamic.itb' ...
INFO: hdimage(sdcard.img): adding partition 'uboot' (in MBR) from 'u-boot.itb' ...
INFO: hdimage(sdcard.img): adding partition 'bootfs' (in MBR) from 'bootfs.img' ...
INFO: hdimage(sdcard.img): adding partition 'rootfs' (in MBR) from 'rootfs.ext4' ...
INFO: hdimage(sdcard.img): adding partition '[MBR]' ...
INFO: hdimage(sdcard.img): adding partition '[GPT header]' ...
INFO: hdimage(sdcard.img): adding partition '[GPT array]' ...
INFO: hdimage(sdcard.img): adding partition '[GPT backup]' ...
INFO: hdimage(sdcard.img): writing GPT
INFO: hdimage(sdcard.img): writing protective MBR
INFO: hdimage(sdcard.img): writing MBR
Successfully generated at /path/to/bianbu-linux/output/k1_plt/images/bianbu-linux-k1_plt-sdcard.img
```

Where `bianbu-linux-k1_plt.zip` is suitable for Titan Flasher, or decompress and brush with fastboot；`bianbu-linux-k1_plt-sdcard.img` is the firmware of sdcard. After decompressing, it can be written to sdcard with dd command or [balenaEtcher](https://etcher.balena.io/).

Firmware default username: `root` password: `bianbu`。

## Customization

### Add board type

Supported board types:

- deb1

The system of production tools usually does not use the dts of the u-boot and kernel directly, but is customized based on it.

Steps to add a new board:

1. Copy u-boot's dts to `buildroot-ext/board/spacemit/k1/plt_dts/u-boot`，such as k1-x_deb2.dts

2. Customization

3. Copy the kernel dts to `buildroot-ext/board/spacemit/k1/plt_dts/kernel`，for example, k1-x_deb2.dts

4. Modify the dtsi path

   ```diff
   -#include "k1-x.dtsi"
   -#include "k1-x_pinctrl.dtsi"
   -#include "lcd/lcd_gx09inx101_mipi.dtsi"
   -#include "k1-x-hdmi.dtsi"
   -#include "k1-x-lcd.dtsi"
   -#include "k1-x-camera-sdk.dtsi"
   +#include "spacemit/k1-x.dtsi"
   +#include "spacemit/k1-x_pinctrl.dtsi"
   +#include "spacemit/lcd/lcd_gx09inx101_mipi.dtsi"
   +#include "spacemit/k1-x-hdmi.dtsi"
   +#include "spacemit/k1-x-lcd.dtsi"
   +#include "spacemit/k1-x-camera-sdk.dtsi"
   ```

5. Changing other configurations

6. Modify `output/k1_plt/.config` to add the dts for the new board type

   ```diff
   -BR2_LINUX_KERNEL_INTREE_DTS_NAME="k1-x_deb1"
   -BR2_LINUX_KERNEL_CUSTOM_DTS_PATH="$(BR2_EXTERNAL_Bianbu_PATH)/board/spacemit/k1/plt_dts/kernel/k1-x_deb1.dts"
   -BR2_TARGET_UBOOT_CUSTOM_DTS_PATH="$(BR2_EXTERNAL_Bianbu_PATH)/board/spacemit/k1/plt_dts/u-boot/k1-x_deb1.dts"
   +BR2_LINUX_KERNEL_INTREE_DTS_NAME="k1-x_deb1 k1-x_deb2"
   +BR2_LINUX_KERNEL_CUSTOM_DTS_PATH="$(BR2_EXTERNAL_Bianbu_PATH)/board/spacemit/k1/plt_dts/kernel/k1-x_deb1.dts $(BR2_EXTERNAL_Bianbu_PATH)/board/spacemit/k1/plt_dts/kernel/k1-x_deb2.dts"
   +BR2_TARGET_UBOOT_CUSTOM_DTS_PATH="$(BR2_EXTERNAL_Bianbu_PATH)/board/spacemit/k1/plt_dts/u-boot/k1-x_deb1.dts $(BR2_EXTERNAL_Bianbu_PATH)/board/spacemit/k1/plt_dts/u-boot/k1-x_deb2.dts"
   ```

7. Recompile u-boot, kernel, and firmware

   ```shell
   make uboot-rebuild
   make linux-rebuild
   make
   ```

8. Saving configuration

   ```shell
   make savedefconfig
   ```

### Customizing rootfs

By customizing rootfs with `buildroot-ext/board/spacemit/k1/plt_overlay`，the files in this directory will be copied to the `output/k1_plt/target` directory before the image is made.
