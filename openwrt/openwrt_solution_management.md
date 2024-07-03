# 方案管理
本文档介绍进迭OpenWrt SDK的方案管理，目前发行的SDK默认适配k1-sbc、k1-nas方案，    
每个方案支持多个板型，如k1-sbc方案支持k1-x_MUSE-Pi、k1-x_deb1板型，未来会持续更新。




# 方案总览
以开发板方案k1-sbc为例，通常跟以下配置文件有关，后续章节逐一介绍
```
#方案配置
feeds/spacemit_openwrt_feeds/spacemit_k1_defconfig

#方案编译入口
openwrt/target/linux/spacemit/Makefile

#方案定义
openwrt/target/linux/spacemit/k1-sbc/config-6.1
openwrt/target/linux/spacemit/k1-sbc/target.mk
openwrt/target/linux/spacemit/k1-sbc/base-files/

#方案设备树管理
openwrt/target/linux/spacemit/dts/

#方案启动参数
openwrt/target/linux/spacemit/image/env_k1-x.txt

#固件定义
openwrt/target/linux/spacemit/image/k1-sbc.mk
#方案的固件分区
openwrt/target/linux/spacemit/image/partition_tables/partition_2M.json
openwrt/target/linux/spacemit/image/partition_tables/partition_flash.json
openwrt/target/linux/spacemit/image/partition_tables/partition_universal.json

#方案首次启动配置
openwrt/target/linux/spacemit/base-files/etc/uci-defaults/


```

## 方案配置

`feeds/spacemit_openwrt_feeds/spacemit_k1_defconfig`

k1-sbc方案的编译配置，用于指导OpenWrt的编译行为


## 方案Makefile
`openwrt/target/linux/spacemit/Makefile`

SUBTARGETS添加方案名称

```
    ...
 12 SUBTARGETS:=k1-nas k1-sbc
    ...
```


## 方案目录
`openwrt/target/linux/spacemit/k1-sbc`

创建跟方案名称同名目录：k1-sbc，包含以下内容

* openwrt/target/linux/spacemit/k1-sbc/config-6.1   

  config-6.1为方案的kernel配置，编译kernel时合并openwrt/target/linux/generic/config-6.1 与本方案config-6.1，相同的配置选项以本方案为主

* openwrt/target/linux/spacemit/k1-sbc/target.mk   

  定义方案的通用信息，如DEVICE_TYPE:=router（可选router、nas）。   

  此为OpenWrt定义，不同device type默认包含不同软件包


* openwrt/target/linux/spacemit/k1-sbc/base-files/   

  包含要打包进rootfs的配置文件，如下：
```
openwrt/target/linux/spacemit/k1-sbc/base-files$ tree
.
├── etc
│   ├── board.d
│   │   ├── 01_leds
│   │   └── 02_network
│   ├── hostapd.conf
│   ├── hosts
│   ├── init.d
│   │   └── custom_wifi_ap
│   └── inittab
├── lib
│   ├── preinit
│   │   └── 79_move_config
│   └── upgrade
│       └── platform.sh
└── usr
    └── bin
        ├── uas-gadget2-bot.sh
        ├── uas-gadget2.sh
        └── uas-gadget3.sh

8 directories, 11 files
```



## 方案设备树
`openwrt/target/linux/spacemit/dts/`

不同板型的内核设备树，可参考[设备管理](openwrt_device_management.md)添加一个新板型支持


## 方案分区表
`openwrt/target/linux/spacemit/image/partition_tables/`

这里的分区表是多个方案通用。里面包含不同存储介质的分区表配置，    

也可自行添加跟板载存储介质容量一致的分区表，TitanFlasher工具烧写固件时自动匹配

* partition_2M.json，用于nor刷机启动，一般与emmc/ssd等blk设备一起使用
* partition_universal.json，用于blk设备的刷机启动，
* partition_flash.json，用于spacemit Titanflasher工具制作卡量产

修改分区表可能会影响到系统正常启动，详细的修改方式请参考《启动》文档

## 方案启动参数
`openwrt/target/linux/spacemit/image/env_k1-x.txt`

uboot最高优先级环境变量，这里可以设定bootargs启动参数，loglevel等    

默认的bootargs为：

```
commonargs=setenv bootargs earlycon=${earlycon} earlyprintk console=tty1 console=${console} loglevel=${loglevel} clk_ignore_unused swiotlb=65536 rdinit=${init}
```

可在env_k1-x.txt中重定义earlycon/console/loglevel/init等环境变量
```
# Common parameter
earlycon=sbi
console=ttyS0,115200
init=/init
bootdelay=0
loglevel=8

```

也可以直接重定义整个bootargs
```
bootargs=earlycon=sbi console=ttyS0,115200 loglevel=4

```


## 方案固件
`openwrt/target/linux/spacemit/image/k1-sbc.mk`

制定方案的固件编译，如包含支持的板型、生成sdcard.img、spacemit.zip刷机包等，   

比如不想生成sdcard.img，可去掉IMAGE/pack.zip中的sdcard-img。   

比如对本方案增加MUSE-Pi板型支持，则DEVICE_DTS增加k1-x_MUSE-Pi设备树名称，   

且openwrt/target/linux/spacemit/dts/需有一份k1-x_MUSE-Pi.dts

```
# SPDX-License-Identifier: GPL-2.0-only
#
# Copyright (C) 2024 Spacemit Ltd.

define Device/debX
  DEVICE_VENDOR := Spacemit
  DEVICE_MODEL :=k1-x deb board
  DEVICE_DTS_DIR:= ../dts
  DEVICE_DTS := k1-x_deb1 k1-x_MUSE-Pi 
  SOC := KeyStone
  KERNEL_NAME := Image
  KERNEL_IMG := Image.itb
  KERNEL := kernel-bin | fit none
  IMAGES := pack.zip
  IMAGE/pack.zip := $(KERNEL_IMG) | boot-common | sdcard-img | archive-zip
endef
TARGET_DEVICES += debX

```


## 方案首次启动
`openwrt/target/linux/spacemit/base-files/etc/uci-defaults/`

UCI默认设置提供了一种使用UCI预配置您的镜像的方法。 要在设备首次启动时设置一些系统默认值，   

请在目录此中创建一个脚本。



# 新增方案
以k1-sbc方案为例，需要修改以下内容：

1. 修改openwrt/target/linux/spacemit/Makefile中的SUBTARGETS，添加方案名称，如k1-sbc
2. 新增方案目录，如openwrt/target/linux/spacemit/k1-sbc/，添加如下文件

```
openwrt/target/linux/spacemit/k1-sbc/config-6.1
openwrt/target/linux/spacemit/k1-sbc/target.mk

```
3. 新增固件编译和兼容板型 openwrt/target/linux/spacemit/image/k1-sbc.mk
4. feeds/spacemit_openwrt_feeds/目录下新增编译配置，如k1-sbc方案spacemit_k1_defconfig


## 新增板型
每个方案下至少有一个或多个板型，新增板型支持可参考[设备管理](openwrt_device_management.md)


## uboot/opensbi编译配置
如有需要增加uboot/opensbi新的编译配置，可参考修改以下内容

### uboot

1. uboot源码仓库新增编译配置，如u-boot-2022.10/configs/k1_defconfig
2. 修改openwrt/package/boot/uboot-spacemit/Makefile

```

 50 define Build/Configure
 51     $(MAKE) -C $(LOCAL_SOURCE_DIR) k1_defconfig
 52 endef
 53 
```

### opensbi

1. opensbi源码仓库新增编译配置，如opensbi-1.3/platform/generic/configs/k1_defconfig
2. 修改package/boot/opensbi-spacemit/Makefile

```
 59 define Build/Compile
 60     $(eval $(Package/opensbi_$(BUILD_VARIANT))) \
 61         +$(MAKE_VARS) $(MAKE) -C $(LOCAL_SOURCE_DIR) \
 62         PLATFORM=$(PLAT) PLATFORM_DEFCONFIG=k1_defconfig
 63 endef
 64 

```
