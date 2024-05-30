---
sidebar_position: 4
---

# 方案管理

本文档介绍SDK如何管理方案（solution），包括方案的配置文件，如何定制方案和如何添加新方案等。

## 方案的配置文件

以默认方案K1为例，通常包含以下配置文件：

```shell
buildroot-ext/board/spacemit/k1/partition_*.json
buildroot-ext/board/spacemit/k1/env_k1-x.txt
buildroot-ext/board/spacemit/k1/k1-x.bmp
buildroot-ext/board/spacemit/k1/dracut.conf
buildroot-ext/board/spacemit/k1/target_overlay
buildroot-ext/configs/spacemit_k1_defconfig
bsp-src/opensbi/platform/generic/configs/k1_defconfig
bsp-src/uboot-2022.10/configs/k1_defconfig
bsp-src/linux-6.1/arch/riscv/configs/k1_defconfig
```

**buildroot-ext/board/spacemit/k1/partition_*.json**

分区配置文件：

- `partition_2M.json`：for 2MB flash
- `partition_universal.json`：for high capacity flash，例如eMMC、sdcard、SSD

**buildroot-ext/board/spacemit/k1/env_k1-x.txt**

u-boot环境变量，可定制启动参数。

**buildroot-ext/board/spacemit/k1/bianbu.bmp**

u-boot启动logo，v1.0alpha2文件名为`k1-x.bmp`：

- 格式：BMP
- 分辨率：小于或等于屏幕分辨率
- 位深度：32

**buildroot-ext/board/spacemit/k1/dracut.conf**

Dracut的配置文件，Dracut是一个制作initramfs镜像的工具。

**buildroot-ext/board/spacemit/k1/target_overlay**

故名思意，改目录是对target目录的overlay。

**buildroot-ext/configs/spacemit_k1_defconfig**

Buildroot的配置文件。

**bsp-src/opensbi/platform/generic/configs/k1_defconfig**

opensbi的配置文件。

**bsp-src/uboot-2022.10/configs/k1_defconfig**

u-boot的配置文件。

**bsp-src/linux-6.1/arch/riscv/configs/k1_defconfig**

kernel的配置文件。

## 定制方案



## 添加新方案



