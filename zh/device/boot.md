---
sidebar_position: 3
---

# 启动开发指南

## 1. 概述

### 1.1 编写目的

主要介绍SpacemiT的刷机启动流程和自定义配置，以及uboot、opensbi相关的驱动开发，基本调试方法，方便开发者快速上手或者二次开发。

### 1.2 适用范围

本文档适用于SpacemiT的K1系列SOC。

### 1.3 相关人员

- 刷机启动开发工程师
- 内核开发工程师

### 1.4 文档结构

本文档先介绍刷机启动相关的流程和配置方式，然后再介绍uboot/opensbi的编译配置和驱动开发调试，最后记录一些常见的问题处理

## 2. 刷机启动

本章节介绍刷机和启动相关的配置以及实现原理，并提供自定义刷机、启动的方式。
注意，刷机与启动是相互关联的，如果需要对刷机做自定义配置，启动可能也需要做相关的调整，反之亦然。

### 2.1 固件布局

K1系列SOC常见的固件布局有以下三种

![alt text](static/image_play.png)

- emmc
1.如上图所示，emmc有boot0和user_data_area两个区域。这是它的存储特点决定的，不做多展开。
2.bootinfo_emmc.bin和FSBL.bin存储在boot0区域。user_data_area区域用gpt管理分区表，user区域的fsbl分区里面实际上没有内容。brom启动后会从boot0区域加载bootinfo_emmc.bin和FSBL.bin。

- sdcard
1.类似emmc，但sdcard只有user_data_area区域。bootinfo_sd.bin占用前面80 byte，gpt表在1 blk位置。

- nor+ssd

1.包含nor上固件（fsbl+opensbi+uboot），ssd上固件（bootfs+rootfs）
2.也支持nor+emmc等blk设备，默认已适配。假如没有插入ssd，只有nor+emmc，拨码开关拨向nor启动，则刷机启动为nor+emmc
3.仅支持pcie0接口插入的ssd作为刷机启动介质，如果ssd插入非pcie0接口，则仅作为存储介质。
4.如果由taitanflaser工具刷机，ssd上的分区表实际会包含fsbl/ubot/opensbi，但分区内容没有产生作用。这个是由实际的分区表决定

### 2.2 刷机流程和配置

以下内容包含的刷机、烧写为同一个概念。卡量产本质上也是刷机，只是数据来源于sd卡。
卡量产和卡启动为两个不同的概念，卡启动是将镜像烧写到sdcard里面，从sd卡里面启动系统。

#### 2.2.1 刷机流程

刷机流程实际上是通过fastboot协议，将PC端的镜像传输到机子，然后通过mmc/mtd/nvme等接口写入到对应的存储介质。
完整的刷机流程包括进入uboot fastboot模式，镜像传输，存储介质设备检测，创建gpt表，写入数据到存储介质等内容。

##### 2.2.1.1 uboot fastboot模式

本平台刷机通讯协议是基于fastboot协议，并做了一些自定义扩展，所有实际的刷机行为都是基于uboot fastboot模式下开展。
进入uboot fastboot模式后，通过PC端执行fastboot devices能检测到设备（需要配置PC fastboot环境）

```sh
~$ fastboot devices
c3bc939586f0         Android Fastboot
```

以下介绍进入uboot fastboot模式的三种方法。
1.通过板子上按键组合fel键+reset键进入brom-fastboot，然后在上位机（PC）执行以下命令，板子进入uboot刷机交互界面。brom-fastboot仅用于加载启动uboot fastboot

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

2.对于设备已经启动到os，可通过上位机（PC）运行adb reboot bootloader，板子进入uboot刷机交互界面。（某些固件会去掉adb功能，该方式并不是通用的方式）
3.板子启动时通过串口长按s键进入uboot shell，然后串口执行fastboot 0进入uboot刷机交互界面。

以下介绍各种组合介质的刷机命令，其中brom可根据boot pin切换不同的启动介质(nor/nand/emmc)，这个依赖硬件设计，详情请查看硬件参考设计。

##### 2.2.1.2 emmc刷机

- emmc刷机流程

对于emmc的刷机流程如下，前面部分是从brom-fastboot启动到uboot-fastboot模式，下面其他的介质也一样

```sh
fastboot stage factory/FSBL.bin
fastboot continue
#sleep to wait for uboot ready
#linux环境下
sleep 1
#windows环境下
#timeout /t 1 >null   
fastboot stage u-boot.itb
fastboot continue

fastboot flash gpt partition_universal.json
#bootinfo_emmc.bin内容无实际作用，请参考FAQ 6.4章节。但刷写步骤还需执行
fastboot flash bootinfo factory/bootinfo_emmc.bin
fastboot flash fsbl factory/FSBL.bin
fastboot flash env env.bin
fastboot flash opensbi fw_dynamic.itb
fastboot flash uboot u-boot.itb
fastboot flash bootfs bootfs.img
fastboot flash rootfs rootfs.ext4

```

对于emmc，bootinfo_emmc.bin的内容固化在uboot代码里面，这是为了用同一份partition_universal.json分区表兼容卡启动的制作等。且bootinfo_emmc.bin和fsbl.bin实际上是写入到emmc的boot0分区

- emmc分区表配置

分区表保存在buildroot-ext/board/spacemit/k1/partition_universal.json。其中bootinfo不作为一个显式的分区，用于保存启动介质相关的信息。

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

##### 2.2.1.3 nor+blk设备刷机

- nor+blk刷机流程

k1支持nor+ssd/emmc等blk设备的组合刷机启动，且为自适应的方式，如果同时存在ssd和emmc，默认烧写到ssd

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

#刷spi nor
fastboot flash mtd partition_2M.json
fastboot flash bootinfo factory/bootinfo_spinor.bin
fastboot flash fsbl factory/FSBL.bin
fastboot flash env env.bin
fastboot flash opensbi fw_dynamic.itb
fastboot flash uboot u-boot.itb

#刷block设备
#刷机工具会根据实际的分区表烧写镜像，所以对于nor+blk,blk设备的fsbl/uboot/等分区并没有实际作用
fastboot flash gpt partition_universal.json
fastboot flash bootfs bootfs.img
fastboot flash rootfs rootfs.ext4
```

- nor+blk分区表配置

nor分区表，如buildroot-ext/board/spacemit/k1/partition_2M.json。对于nand/nor设备，分区表都是以partiton_xM.json命名，且需要根据实际的flash容量重命名，否则会导致刷机时找不到对应的分区表。
nor/nand的分区表会向最小容量兼容，比如nor的介质容量为8MB，刷机包里面只有partition_2M.json，则会匹配到partition_2M.json的分区表。

对于分区表配置，mtd存储介质(nand/nor flash)会以partition_2M.json等容量的形式表示，blk设备(包括emmc/sd/ssd等)都以partition_universal.json命名。

Nor flash分区表修改：
1.分区起始地址和size默认以64KB对齐（对应erasesize 为64KB）。
2.如果起始地址和size需要更改成4KB对齐，则要开启uboot的编译配置`CONFIG_SPI_FLASH_USE_4K_SECTORS`

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

ssd分区表。对于blk设备的分区表，都是partition_universal.json。此时bootinfo、fsbl、env、opensbi、uboot等分区以及里面的数据不会影响到正常的启动。

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

##### 2.2.1.4 nand 刷机启动

- nand刷机流程

k1支持nand刷机启动。但由于nand与nor共用同一个spi接口，只能选择其中一个方案，默认支持nor启动。
如需支持nand刷机启动，请参考刷机启动中的nand启动配置章节，配置nand启动。

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

fastboot flash mtd partition_64M.json
fastboot flash bootinfo factory/bootinfo_spinand.bin
fastboot flash fsbl factory/FSBL.bin
fastboot flash env env.bin
fastboot flash opensbi fw_dynamic.itb
fastboot flash uboot u-boot.itb

fastboot flash user-bootfs bootfs.img
fastboot flash user-rootfs rootfs.img
```

nand上文件系统管理依赖ubifs，其中bootfs和rootfs分区制作方法如下，

```sh
#制作bootfs.img
mkfs.ubifs -F -m 2048 -e 124KiB -c 8124 -x zlib -o output/bootfs.img -d input_bootfs/

#制作rootfs.img
mkfs.ubifs -F -m 2048 -e 124KiB -c 8124 -x zlib -o output/rootfs.img -d input_rootfs/

#input_bootfs和input_rootfs应该存放bootfs和rootfs中的文件
#例如对于bootfs,应该放入如下文件Image.itb、env_k1-x.txt、k1-x.bmp

#不同的nand需要修改对应的参数，以下是一些参数说明，具体可查看mkfs.ubifs的用法
-m 2048：设置最小输入/输出单元大小为 2048 字节（2 KiB），要与NAND Flash 的页大小相匹配。
-e 124KiB：设置逻辑擦除块的大小为 124 KiB（千字节），这应小于实际的物理擦除块的大小，以留出空间用于 UBI 的管理结构。
-c 8124：设置最大逻辑擦除块的数量为 8124，这个数字限制了文件系统的最大大小，基于逻辑擦除块的大小和数量计算。
-x zlib：设置压缩类型为 zlib，这将在文件系统上对数据进行 zlib 压缩。
-o output/ubifs.img：指定输出文件名，将生成的 UBIFS 文件系统镜像保存为 output/ubifs.img。
-d input/：指定源目录为 input/，mkfs.ubifs 将会把这个目录下的文件和文件结构创建成 UBIFS 文件系统镜像。
```

同时，本平台也支持nand+blk设备启动方式，刷机命令可参考nor+blk设备刷机章节。

- nand分区表配置

如256MB容量的nand，分区表为partition_256M.json

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

##### 2.2.1.5 卡启动

卡启动是指将镜像写入到sd卡中，设备插入该sd卡后，上电启动将从sd卡加载启动系统。

K1平台支持卡启动，且默认先try sd卡，启动失败后再从pin select选中的存储介质加载启动系统。

本平台不支持通过fastboot协议把镜像烧写到sd卡上。需要通过TitanFlash工具或者dd命令制作卡启动。

以下介绍如何通过刷机工具制作卡启动镜像：

- 将tf卡插入读卡器并接入电脑usb口。
- 电脑端打开titanflash刷机工具(安装方式请参考刷机工具使用章节），点击研发工具->卡启动，选择SD卡，刷机包。点击执行。
- 烧写成功后，将tf卡插入设备，上电后设备从卡启动。

![alt text](static/flash_tool_1.png)

##### 2.2.1.6 卡量产

卡量产是指sd卡中写入刷机相关的镜像文件，设备上电启动后先从卡启动，检测到时卡量产后，将sd卡中的镜像文件，按照分区表的位置写入到pin select选中的存储介质

K1支持卡量产烧录。通过titanflash工具把sd卡制作成量产卡，将sd卡插入设备，上电启动后， 会将镜像烧写到emmc/nor/nand等存储介质中。

卡量产的制作步骤如下：

- 将tf卡插入读卡器并接入电脑usb口。
- 打开titanflash工具，选择量产工具->制作量产卡，选择烧录量产卡，点击执行。
- 完成量产卡制作后，将sd卡插入设备，上电启动后会自动进入卡烧录。
- 完成烧录后需要拔下sd卡，设备端上电启动后能正常启动。（如果不拔下sd卡，重复上电会重复开始卡烧录）

![alt text](static/flash_tool_2.png)

#### 2.2.2 刷机工具

本章节简单介绍刷机工具相关的内容

##### 2.2.2.1 刷机工具使用

对于镜像的烧写，可以选择安装titanflash工具，或者使用fastboot工具刷机。
固件生成方式可参考文档[下载和编译](https://bianbu-linux.spacemit.com/source)。

- titanflash刷机工具针对完整刷机包，适用于一般的开发者。
刷机工具安装及使用方式，可参考[这篇文档](https://developer.spacemit.com/documentation?token=O6wlwlXcoiBZUikVNh2cczhin5d)

- fastboot工具针对单个分区的镜像烧写，适用于具备一定能力的开发者，单分区的镜像烧写错误可能会导致系统启动异常，需要谨慎使用，烧写流程请参考刷机流程章节（fastboot环境安装可参考网上[链接](https://www.jb51.net/article/271550.htm)，或者[这篇](https://blog.csdn.net/qq_34459334/article/details/140128714)）。

##### 2.2.2.2 刷机介质选择

- 硬件提供boot download sel switch拨盘切换，可参考如MUSE Pi用户使用指南文档中的[Boot Download Sel&JTAG Sel](https://developer.spacemit.com/documentation?token=ZugWwIVmkiGNAik55hzc4C3Ln6d)章节。
- 其他方案，请参考该方案的硬件用户使用指南
- 对于不同的启动介质，刷机方式已做成自动适配。

### 2.3 启动配置

本章节介绍emmc, sd, nor, nand启动的配置，以及自定义启动配置相关的注意事项。
刷机启动，需要配合正确的分区表等配置以及正确的启动配置。

#### 2.3.1 启动流程

本章节介绍平台芯片的启动流程，并且提供定制化启动的修改方。如下框图，
启动流程主要分成brom->fsbl->opensbi->uboot->kernel等阶段。bootinfo提供fsbl所在偏移、大小、签名等信息

![alt text](static/boot_proc.png)

根据硬件的boot pin select，软件上会从sd或者emmc或者nor等介质加载启动下一级镜像，对于不同的启动介质，启动流程都与上图的一样。具体的boot pin select在刷机配置中的刷机介质选择章节做介绍。

K1上电启动后，会先try boot from sd，启动失败后再根据boot pin select，从emmc或者nand或者nor等存储介质加载uboot和opensbi的镜像。

以下小节将根据启动顺序，分别介绍bootinfo/fsbl/opensbi/uboot/kernel等启动配置

##### 2.3.1.1 brom

brom是一段预编译好程序固化在SoC中，出厂即无法改变。芯片上电或复位会运行brom，通过不同的boot pin会指导brom从对应的存储介质加载bootinfo和fsbl。
Boot pin请参考刷机介质选择章节，这里不做展开。

##### 2.3.1.2 bootinfo

brom启动后会从对应存储介质的物理0地址偏移获取booinfo信息（对于emmc会从boot0区域的0地址加载）。bootinfo包含fsbl、fsbl1的位置以及大小，crc值等信息。
其中，bootinfo*描述文件在如下目录的*.json

```sh
uboot-2022.10$ ls board/spacemit/k1-x/configs/
bootinfo_emmc.json  bootinfo_sd.json  bootinfo_spinand.json  bootinfo_spinor.json
```

json文件举例说明，如下记录了各种存储介质的默认的fsbl/fsbl1位置， fsbl1为备份数据。如需修改偏移地址，可修改bootinfo_*json文件再重新编译uboot。

fsbl的位置尽量保持原厂设置，如果需要自定义，需要考虑不同存储介质的特性，如nand需要按sector对齐等。emmc的fsbl只支持在boot分区加载。

```sh
emmc:boot0:0x200, boot1:0x0
sd:0x20000/0x80000
nor:0x20000/0x70000
nand:0x20000/0x80000
```

编译uboot时在其根目录下生成对应的*.bin文件等bootinfo镜像文件，生成方式可参考uboot-2022.10/board/spacemit/k1-x/config.mk

```sh
uboot-2022.10$ ls -l
bootinfo_emmc.bin
bootinfo_sd.bin
bootinfo_spinand.bin
bootinfo_spinor.bin
```

##### 2.3.1.3 fsbl

brom获取bootinfo后，会从指定的offset加载fsbl。
fsbl文件是由4K头信息+uboot-spl.bin组合而成，头信息会包含uboot-spl的一些必要信息，如crc校验值等
fsbl启动后，会先初始化ddr，然后从分区表加载opensbi、uboot到内存的指定位置，再运行opensbi，接着opensbi会启动uboot。(fsbl如何初始化ddr的内容不在本文档的描述范围内)

##### 2.3.1.4 opensbi和uboot加载启动

不同存储介质对opensbi和uboot的加载方式是多样化的，以下按存储介质分类去描述spl加载启动uboot/opensbi的配置。
k1平台支持加载独立分区的opensbi和uboot，也支持加载一个镜像(uboot-opensbi.itb)的方式。默认以分区名加载。
加载启动一个镜像(uboot-opensbi.itb)的方式请参考FAQ部分。

###### 2.3.1.4.1 emmc和sdcard

emmc与sd都是用到mmc驱动，uboot的代码流程上会根据启动的boot pin select选择emmc或sd。两者的启动配置类似。
opensbi和uboot有不同的image格式，根据存放的位置，可以分为裸分区、文件、fit格式等。
K1平台默认都会先try boot from sd，失败后再try其他介质(emmc/nor/nand等其中一个)。

spl-dts需要开启emmc/sd的配置

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

以下针对emmc/sd的不同启动方式做介绍，默认以分区名加载。

- 裸分区方式

1.fsbl加载启动uboot/opensbi流程(以emmc为例，对于sd/nor等介质类似，限于篇幅不做赘述)

执行make uboot_menuconfig，进入spl的编译配置(所有spl相关的配置的入口，下面的不做赘述)

![alt text](static/spl-config_1.png)

选中以分区表的方式加载，这里默认支持uboot/opensbi独立加载。spl会先找名称为opensbi、uboot的分区，如果找不到该分区名，则尝试分别以分区号1和2加载raw数据。

![alt text](static/spl-config_2.png)

![alt text](static/spl-config_3.png)

spl加载好opensbi和uboot后，会启动opensbi，并传入uboot的内存地址和dtb信息，opensbi根据地址和dtb启动uboot

- 绝对偏移

通过绝对偏移地址加载opensbi/uboot
开启以下配置，指定emmc的偏移地址。这时候会按照emmc的绝对偏移地址加载uboot/opensbi镜像。
该种方式不支持opensbi和uboot分开，也就是opensbi和uboot要打包成一个文件，即fit格式。
若想要支持独立加载，可参考uboot-2022.10/common/spl/spl_mmc.c中的case MMCSD_MODE_RAW部分代码

![alt text](static/spl-config_4.png)

- 文件系统方式

更改spl通过文件系统加载opensbi/uboot
uboot-spl支持通过文件系统的方式加载opensbi和uboot，该方式支持独立加载uboot、opensbi文件
make menuconfig打开以下配置，如图选中fat文件系统，加载opensbi.itb、uboot.itb

![alt text](static/spl-config_5.png)

###### 2.3.1.4.2 nor

对于nor介质启动，k1会提供nor(u-boot-spl/uboot/opensbi)+ssd(bootfs/rootfs)或者nor(u-boot-spl/uboot/opensbi)+emmc(bootfs/rootfs)的启动方式，默认先try ssd，

spl的dts配置如下

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

spl支持nor加载启动，需要先开启以下配置，默认已开启
1.执行make uboot_menuconfig，选择SPL configuration options

![alt text](static/spl-config_6.png)

2.开启“Support MTD drivers”、“Support SPI DM drivers in SPL”、“Support SPI drivers”、“Support SPI flash drivers”、“Support for SPI flash MTD drivers in SPL”、“Support loading from mtd device”，
“Partition name to use to load U-Boot from”与分区表中的分区名保持一致。
![alt text](static/spl-config_7.png)

3.blk设备配置
执行make uboot_menuconfig，选择Device Drivers --->Fastboot support
选择Support blk device，这里支持是ssd/emmc等blk设备，如ssd对应的是nvme，emmc对应是mmc。
![alt text](static/spl-config_8.png)

4.对于mtd设备，需要开启env，以确保spl启动后能从env获取mtd分区信息
执行make uboot_menuconfig，选择Environment，选择开启spi的env加载。这里的env偏移地址需要与分区表的env分区保持一致，如0x80000。
![alt text](static/spl-config_9.png)

![alt text](static/spl-config_10.png)

5.spi flash驱动需要适配硬件上的spi flash对应厂商的型号(下面的nand也同理)
在menuconfig上选择对应的厂商manufactureID
执行make uboot_menuconfig，选择Device Drivers--->MTD Support--->SPI Flash Support
根据硬件的spi flash厂商，选择对应的驱动程序。
![alt text](static/spl-config_11.png)

如果驱动里面都没有，可以在代码上直接添加。flash_name可以自定义，一般为硬件flash名称，0x1f4501为该flash的jedecid，其他参数可以根据该硬件的flash添加。

```sh
//uboot-2022.10/drivers/mtd/spi/spi-nor-ids.c
const struct flash_info spi_nor_ids[] = {
     { INFO("flash_name",    0x1f4501, 0, 64 * 1024,  16, SECT_4K) },
```

加载方式主要有两种，如下

- 裸分区方式

对于nor设备，会根据mtd分区表获取uboot/opensbi。配置方式如下
![alt text](static/spl-config_12.png)

- 绝对偏移

nor启动支持以存储介质的绝对偏移加载启动，但不支持独立加载uboot、opensbi的方式，需把uboot和opensbi打包到一起，即fit格式。
开启以下配置，输入存储介质的绝对偏移地址

![alt text](static/spl-config_13.png)

###### 2.3.1.4.3 nand

对于nand介质启动，K1会提供nand(u-boot-spl/uboot/opensbi)+ssd(bootfs/rootfs)或者纯nand(u-boot-spl/uboot/opensbi/kernel)启动， 默认不开启nand启动。

spl-dts配置如下

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

下面将介绍纯nand启动方案配置。
执行make uboot_menuconfig，选择SPL configuration options

![alt text](static/spl-config_14.png)

开启
Support MTD drivers
Support SPI DM drivers in SPL
Support SPI drivers
Use standard NAND driver
Support simple NAND drivers in SPL
Support loading from mtd device，
Partition name to use to load U-Boot from与分区表中的分区名保持一致。
如果开启opensbi/uboot独立镜像。则需要开启Second partition to use to load U-Boot from，且必须要保持opensbi/uboot的先后顺序。

![alt text](static/spl-config_15.png)

对于mtd设备，需要开启env，以确保spl启动后能从env获取mtd分区信息
执行make uboot_menuconfig，选择Environment，选择开启spi的env加载。这里的env偏移地址需要与分区表的env分区保持一致，如0x80000。

nand flash驱动需要适配硬件上的spi flash对应厂商的型号。目前已支持的nand flash如下驱动所示。如果没有对应的驱动，可以在other.c驱动中添加厂商的jedecid

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

##### 2.3.1.5 启动kernel

uboot的env定义了多种启动组合，开发者可以根据需要自行定制。其中方案自带的k1-x_env.txt优先级更高，会覆盖env.bin相同的变量。
fsbl加载启动opensbi->uboot后，uboot通过文件系统load命令从fat或ext4文件系统里面加载kernel、dtb等镜像文件到内存，然后执行bootm命令启动kernel。
 具体加载启动kernel的流程可以参考uboot-2022.10/board/spacemit/k1-x/k1-x.env，里面已包含emmc/sd/nor+blk等启动方案，且能自动适配对应的启动介质。

- mmc启动

对于sd/emmc，都属于mmc_boot

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

rootfs由uboot传递的bootargs到kernel，由kernel或者init脚本解析并挂载rootfs
如下字段root=/dev/mmcblk2p6即为rootfs分区

```sh
bootargs=earlycon=sbi earlyprintk console=tty1 console=ttyS0,115200 loglevel=8 clk_ignore_unused swiotlb=65536 rdinit=/init root=/dev/mmcblk2p6 rootwait rootfstype=ext4
```

- nor+blk启动

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

- nand启动

```sh
//uboot-2022.10/board/spacemit/k1-x/k1-x.env
run nand_boot;

//待适配
```

### 2.4 安全启动

安全启动基于fit image格式实现：

1. opensbi、uboot.itb、kernel打包成fit image
2. 打开代码中安全启动配置；
3. 使用私钥对fit image做签名，并导出公钥到dtb中；

#### 2.4.1 验签流程

启动验签流程如下图所示：

![alt text](static/secure_boot.png)

签名过程：

1. Boot ROM做为信任根，不可更改；
2. Hash of ROTPK烧录到芯片内部Efuse，只能烧录一次；
3. 哈希算法为SHA256；
4. 签名算法使用SHA256+RSA2048；

#### 2.4.2 配置

- uboot编译配置

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

- opensbi编译配置

```sh
CONFIG_FIT_SIGNATURE=y
```

- kernel编译配置

```sh
CONFIG_FIT_SIGNATURE=y
```

#### 2.4.3 公私钥

使用openssl生成私钥与证书，保护好私钥，发布证书。

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

#### 2.4.4 镜像签名

- 修改its文件，打开hash与签名配置

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

- 使用私钥和证书，对fit image文件签名

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

- 更新public key info到上一级引导代码

  - 如更新uboot签名所用私钥对应的公钥信息到FSBL dts中去

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

## 3. uboot功能与配置

本章节主要介绍uboot的功能以及常用的配置方法。

### 3.1 功能介绍

uboot的主要功能有以下几点：

- 加载启动内核

uboot从存储介质(emmc/sd/nand/nor/ssd等)，加载内核镜像到内存指定位置，并启动内核。

- fastboot刷机功能

通过fastboot工具，烧写镜像到指定的分区位置。

- 开机logo

uboot启动阶段显示启动logo以及boot menu。

- 驱动调试

基于uboot调试设备驱动，如mmc/spi/nand/nor/nvme等驱动。uboot提供shell命令行对各个驱动进行功能调试。
uboot驱动在drivers/目录下。

### 3.2 编译

本章节介绍基于uboot代码环境，编译生成uboot的镜像文件。

- 编译配置

首次编译，或者需要重新选择其他方案，则需要先选择编译配置，这里以k1为例：

```shell
cd ~/uboot-2022.10
make ARCH=riscv k1_defconfig -C ~/uboot-2022.10/
```

可视化更改编译配置：

```shell
make ARCH=riscv menuconfig
```

![a](static/OLIdbiLK4onXqyxOOj8cyBDCn3b.png)

通过键盘"Y"/"N"以开启/关闭相关的功能配置。保存后会更新到 uboot 根目录的.config 文件。

- 编译 uboot

```shell
cd ~/uboot-2022.10
GCC_PREFIX=riscv64-unknown-linux-gnu-
make ARCH=riscv CROSS_COMPILE=${GCC_PREFIX} -C ~/uboot-2022.10 -j4
```

- 编译产物

```shell
~/uboot-2022.10$ ls u-boot* -l
u-boot
u-boot.bin           # uboot镜像
u-boot.dtb           # dtb文件
u-boot-dtb.bin       # 带dtb的uboot镜像
u-boot.itb           # 将u-boot-nodtb.bin和dtb打包成fit格式
u-boot-nodtb.bin
bootinfo_emmc.bin    # 用于emmc启动时记录spl位置的信息
bootinfo_sd.bin
bootinfo_spinand.bin
bootinfo_spinor.bin
FSBL.bin             # u-boot-spl.bin加上头信息。由brom加载启动
k1-x_deb1.dtb        # 方案deb1的dtb文件
k1-x_spl.dtb         # spl的dtb文件
```

### 3.3 dts配置

uboot dts 配置在目录 uboot-2022.10/arch/riscv/dts/，根据不同的方案修改该方案的 dts，如 deb1 方案。

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

## 4. uboot驱动开发调试

本章节主要介绍 uboot 的驱动使用和调试方法，默认情况下所有的驱动都已经做好配置。

### 4.1 boot kernel

本小节介绍 uboot 启动 kernel，以及分区的自定义配置和启动。

- 开发板上电启动后，立即按下键盘上的"s"键，进入 uboot shell
- 可通过执行 fastboot 0 进入 fastboot mode，通过电脑端的 fastboot stage Image 发送镜像到开发板。(或者其他下载镜像的方式，如 fatload 等命令)
- 执行 booti 启动 kernel(或者 bootm 启动 fit 格式镜像)

```shell
#下载kernel镜像
=> fastboot -l 0x40000000 0
Starting download of 50687488 bytes
...
downloading/uploading of 50687488 bytes finished

#电脑端执行命令
C:\Users>fastboot stage Z:\k1\output\Image
Sending 'Z:\k1\output\Image' (49499 KB)           OKAY [  1.934s]
Finished. Total time: 3.358s

#下载完成后，在uboot shell中，通过键盘输入CTRL+C退出fastboot mode。


#下载dtb
=> fastboot -l 0x50000000 0
Starting download of 33261 bytes

downloading/uploading of 33261 bytes finished

#电脑端执行命令
C:\Users>fastboot stage Z:\k1\output\k1-x_deb1.dtb
Sending 'Z:\k1\output\k1-x_deb1.dtb' (32 KB)      OKAY [  0.004s]
Finished. Total time: 0.054s
```

执行启动命令

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

- 通过 bootm 命令启动 fit 格式镜像

假设 emmc 中分区 5 为 fat32 文件系统。且里面保存 uImage.itb 文件，通过以下命令加载启动 kernel。

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

本章节介绍如何配置在 uboot 启动阶段，从指定存储介质加载 env。

- 执行 make menuconfig，进入 Environment，

![alt text](static/CgrNbzNbkot1tvxOXIhcGMrRnvc.png)

![atl text](static/Od7AbhfLSoHWY9xN8uIcwlAhnhb.png)

目前支持可选的介质为 mmc，mtd 设备(其中 mtd 设备包括 spinor，spinand)。

env 的偏移地址需要根据分区表的配置来确定，具体可以参考刷机启动设置章节的分区表配置，默认是 0x80000。

```shell
(0x80000) Environment address       #spinor的env偏移地址
(0x80000) Environment offset        #mmc设备的env偏移地址
```

### 4.3 mmc

emmc 和 sd 卡都是使用到 mmc 驱动，dev number 分别为 2、0。

- config 配置

执行 make menuconfig，进入 Device Drivers--->MMC Host controller Support，开启以下配置

![alt text](static/YnF5beU32ojicYx6xbkcM2pGn2b.png)

- dts 配置

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

- 调试验证

uboot shell 提供命令行调试 mmc 驱动，需要开启编译配置项 CONFIG_CMD_MMC

```shell
=> mmc list
sdh@d4280000: 0 (SD)
sdh@d4281000: 2 (eMMC)
=> mmc dev 2 #切换到emmc
switch to partitions #0, OK
mmc2(part 0) is current device

#read 0偏移的0x1000个blk_cnt到内存0x40000000
=> mmc read 0x40000000 0 0x1000

MMC read: dev # 2, block # 0, count 4096 ... 4096 blocks read: OK

#从内存地址0x40000000 写到0x1000个blk_cnt到0偏移
=> mmc write 0x40000000 0 0x1000

MMC write: dev # 2, block # 0, count 4096 ... 4096 blocks written: OK

#其他用法可参考mmc -h
```

- 常用接口

参考 cmd/mmc.c 中的接口

### 4.4 nvme

nvme 驱动主要用于调试 ssd 硬盘。

- config 配置

执行 make menuconfig，进入 Device Driver，开启以下配置，

![a](static/OLktbqlRLoreIPxlZ9TcGtwOnff.png)

![a](static/UrVybSqdFo0iTnxZ8QAcKoWAnqc.png)

- dts 配置

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

- 调试验证

需要开启编译配置 CONFIG_CMD_NVME，调试方法如下：

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

- 常用接口

参考 cmd/nvme.c 中的代码接口

### 4.5 net

- config 配置

执行 make menuconfig，开启以下配置，

![a](static/RCZdbLULLo7I0axEo3rc71BdnZg.png)

![a](static/K5s8bumbzofb0txqmiXc43BCnRg.png)

- dts 配置

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

- 调试验证

需要先开启编译配置 CONFIG_CMD_NET，网线接上开发板的网口，且已经准备好 tftp 服务器(tftp 服务器的搭建方法可搜索网上资料，这里不做介绍)

```shell
=> dhcp #执行dhcp后，如果返回地址，表示与网络服务器联通。其他情况为连接失败
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

#启动kernel
=>bootm 0x40000000 
```

- 常用接口

参考 cmd/net.c 中的代码接口

### 4.6 spi

spi 只引出一个硬件接口，所以只支持 nand 或者 nor flash。

- config 配置

执行 make menuconfig，进入 Device Drivers，开启以下配置

![a](static/DdHBbRJQpoIoopxMuO8cS3GBnXg.png)

![a](static/AH6bbloZ9omZNux2ZCxcXblVnvc.png)

- dts 配置

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

- 调试验证

开启 uboot shell 命令 sspi 的配置，CONFIG_CMD_SPI，

调试命令：

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

- 常用接口

参考 cmd/spi.c 里面的接口

### 4.7 nand

nand 驱动是基于 spi，所以需要先开启 spi 驱动功能。

- config 配置

执行 make menuconfig，进入 Device Drivers --->MTD Support

![a](static/Pmlobv86koO6qpxDohMcycGVn4e.png)

若需要新增一个 nand flash，可以根据已支持的厂商驱动，添加该 nand flash 的 jedec id。

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

如在 gigadevice 添加新的 flash

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

如果为其他品牌的 nand flash，可参考 gigadevice 的驱动重新实现。

- dts 配置

nand 驱动是挂在 spi 驱动下，所以 dts 需要配置在 spi 节点下面。

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

- 调试验证

nand 驱动可以基于 mtd 命令调试

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

- 常用接口

参考 cmd/mtd.c 的代码接口

### 4.8 nor

nor 驱动是基于 spi 驱动下，所以需要先开启 spi 驱动功能。

- config 配置

执行 make menuconfig，进入 Device Drivers --->MTD Support --->SPI Flash Support，将以下配置打开(默认情况下已开启)。示例以开启 winbond 的 nor flash 为例。

![a](static/WkhTbAHpFot5raxYWMWckwBwnsh.png)

![a](static/VTT0bxjO1oobWYxPeficMzw4nfl.png)

添加一个新的 spi nor flash:

- 对于上图已支持的厂商 nor flash，可以直接开启对应的编译配置，如 gigadevice 厂商的 flash。
- spi flash 的 jedec id 列表在 uboot-2022.10/drivers/mtd/spi/spi-nor-ids.c 里面维护。如果列表里面没有特定的 nor flash jedec id，则可以自行添加 jedec id 到列表中（jedec id 为 spi flash 对应的厂商代号，可根据 nor flash 的 datasheet 查找 manufac 关键字，如 winbond 为 0xfe）
- dts 配置

nor 驱动依赖 spi 驱动接口，spi 驱动请参考 spi 子章节。需要添加 dts 节点，如下：

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

- 调试验证

可基于 uboot 命令行的 mtd/sf 命令调试。编译配置需要开启 CONFIG_CMD_MTD=y，CONFIG_CMD_SF

基于 mtd 命令读写 nor flash：

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

基于 sf 命令读写 nor flash

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

- 常用接口

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

本小节主要介绍如何开启 hdmi 驱动。

- config 配置

执行 make uboot_menuconfig，即进入 Device Drivers -> Graphics support，将以下配置打开(默认情况下已开启)。

![a](static/GeszbbETBojI9KxyCWBcPM7fnHe.png)

![a](static/MXYNbqJwjoNsdhxsBT2clnTSn1e.png)

![a](static/MX60b8b2uoLDLaxHlJlcTyc7nte.png)

![a](static/Sm8hbLmawoxfMdxrMlBcJMVInHd.png)

![a](static/NuSSbshdfon2mWxWZU6cEipvnwf.png)

- dts 配置

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

本小节主要介绍如何在 uboot 启动阶段显示 bootlogo。

- config 配置

执行 make menuconfig，将以下配置打开。

1. 首先打开 uboot 下 hdmi 支持，参考 hdmi小节。
2. 再打开 uboot 下 bootlogo 支持，进入 Device Drivers -> Graphics support，开启以下选项。

![a](static/FfzObuq4poT5ZYxU17scAzZRnyf.png)

- env 配置

在 uboot-2022.10\include\configs 目录下的 k1-x.h 增加 bootlogo 所需的 3 个 env 变量：splashimage、splashpos、splashfile。

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

其中 splashimage 代表 bootlogo 图片加载到内存的地址;

splashpos 代表图片显示的位置，“m,m”代表图片显示在屏幕正中间;

splashfile 指将要显示的 bmp 文件名，这个图片需要放在在 bootfs 所在的分区。

- 打包.bmp 图片进 bootfs

将要显示的 bmp 图片打包进 bootfs：

将 k1-x.bmp 文件放在./buildroot-ext/board/spacemit/k1 目录下，文件名要和 buildroot-ext/board/spacemit/k1/prepare_img.sh 中的 UBOOT_LOGO_FILE 和以及环境变量 splashfile 保持一致，如 k1-x.bmp。

在编译打包后，bmp 图片将被打包进 bootfs。

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

- 如何修改 bootlogo

1. 直接替换 buildroot-ext/board/spacemit/k1/目录中的 k1-x.bmp，或根据上述描述新增图片

### 4.11 boot menu

本小节主要介绍开启 uboot 的 bootmenu 功能。

- config 配置

执行 make menuconfig，进入 Command line interface > Boot commands，将以下配置打开

![a](static/BmycbCac2oCtuGxjjpvcUlHunug.png)

再进入 Boot options > Autoboot options 开启以下选项

![a](static/UEhPbxaIgoXpv8xBh52cn7Vdndb.png)

- env 配置

buildroot-ext/board/spacemit/k1/env_k1-x.txt 中需要添加 bootdelay 和 bootmenu_delay，例如 bootdelay=5，bootmenu_delay=5,其中 5 代表 bootmenu 的等待时间，单位为秒。

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

- 进入 bootmenu

上电后按住键盘 Esc 键进入 bootmenu

### 4.12 fastboot command

本小节主要介绍 k1-deb1 方案支持的 fastboot command。

- 编译配置

执行 make menuconfig，进入 Device Drivers --->Fastboot support，开启以下编译配置

![a](static/LrxMbKM9Eoioc9xJo9bcUrA2nnb.png)

fastboot 依赖 usb 驱动，需要开启 usb 的配置的 USB support。

![a](static/DmeEbYiPqoUuW9xa8u8cw9hlnPg.png)

![a](static/MuMabzRykoQeWHxZk1UcNDZVnDg.png)

- 进入 fastboot mode

1.可以通过系统启动后按"s"键进入 uboot shell，执行 fastboot 0 进入 fastboot 模式。

系统默认的 fastboot buffer addr/size 为宏定义 CONFIG_FASTBOOT_BUF_ADDR/CONFIG_FASTBOOT_BUF_SIZE

```shell
#或者fastboot -l 0x30000000 -s 0x10000000 0，指定buff addr/size
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

2.设备启动到 bianbu os 后，发送 adb reboot bootloader 命令，系统将重启进入 fastboot 模式(某些方案可能不支持)

- 支持的 fastboot command

电脑端的 fastboot 环境配置，请参考电脑环境安装章节。

```shell
#fastboot原生协议命令
fastboot devices              #显示可用的设备
fastboot reboot               #重启设备
fastboot getvar [version/product/serialno/max-download-size]
fastboot flash partname image #烧写image镜像到partname分区
fastboot erase partname       #擦除partname分区
fastboot stage file           #下载文件file到内存的buff addr

#oem厂商自定义的命令和功能
fastboot getvar [mtd-size/blk-size] #获取mtd/blk设备size，没有则返回NULL
fastboot oem read part              #读取part中的数据到buff addr
fastboot get_staged file       #上传数据并命名为file。依赖oem read part命令
```

### 4.13 文件系统

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

类似 fat 指令

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

### 4.14 常用uboot命令

常用命令/工具

```shell
printenv  - print environment variables
md        - memory display
mw        - memory write (fill)
fdt       - flattened device tree utility commands



help      - print command description/usage
```

- fdt

fdt 命令主要用于打印 dts 内容，如 uboot 启动后所加载的 dtb 文件

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

- shell 命令

uboot 支持 shell 风格的命令，如 if/fi，echo 等。

```shell
=> if test ${boot_device} = nand; then echo "nand boot"; else echo "not nand boot";fi
not nand boot
=> printenv boot_device
boot_device=nor
=> if test ${boot_device} = nor; then echo "nor boot"; else echo "not nor boot";fi
nor boot
=>
```

## 5. opensbi功能与配置

本章节介绍opensbi的编译配置

### 5.1 opensbi编译

```shell
cd ~/opensbi/

#注意，这里的编译工具链需要为spacemit提供，非spacemit提供的可能会引起编译异常
GCC_PREFIX=riscv64-unknown-linux-gnu-

CROSS_COMPILE=${GCC_PREFIX} PLATFORM=generic \
PLATFORM_DEFCONFIG=k1-x_deb1_defconfig \
PLATFORM_RISCV_ISA=rv64gc \
FW_TEXT_START=0x0  \
make
```

### 5.2 生成的编译文件

```shell
~/opensbi$ ls build/platform/generic/firmware/ -l
fw_dynamic.bin     #dynamic镜像跳转会传递配置参数
fw_dynamic.elf
fw_dynamic.elf.dep
fw_dynamic.itb     #将fw_dynamic.bin打包成fit格式
fw_dynamic.its
fw_dynamic.o
fw_jump.bin       #jump镜像仅做跳转
fw_payload.bin    #payload镜像会包含uboot镜像
```

### 5.3 opensbi功能配置

可以通过执行 menuconfig，开启或关闭某些功能

```shell
make PLATFORM=generic PLATFORM_DEFCONFIG=k1-x_deb1_defconfig menuconfig
```

![a](static/JdMVb4GyioYNhMxUOhHc8R3enEd.png)

## 6. FAQ

本章节介绍常见问题以及解决方式，或者常用的调试手段以及容易出错的问题记录

### 6.1 用titanflash烧写固件时，没有检测到设备

确保usb线已接入电脑，且串口打印如下所示：
![alt text](static/flash_tool_3.png)

如果还是没有检测到设备，检查设备管理器是否存在adb设备，如果没有则参考电脑环境安装的fastboot环境安装章节
![alt text](static/flash_tool_4.png)

### 6.2 更新代码后有涉及到def_config的改动，编译的时候没有生效

需要执行make menuconfig更新到.config，编译才生效。

### 6.3 uboot和opensbi合并加载

SDK设计把uboot和opensbi分开加载，开发者也可以根据需求把两者合并。需要以下几个步骤。

- spl启动配置

1.uboot取消second分区的配置，取消选中"Second partition to use to load U-Boot from"
2.更改分区名为opensbi-uboot，重新编译uboot（注意，分区名为opensbi-uboot）

![alt text](static/spl-config_16.png)

- 生成uboot-opensbi.itb

创建uboot-opensbi.its文件，内容如下

```dts
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

- mkimage生成itb文件

把如下文件放在同一目录，
```uboot-opensbi.its```
```u-boot-nodtb.bin```
`fw_dynamic.bin`
`k1-x_MUSE-Card.dtb`（此为方案设备树，应根据实际方案名修改）
执行以下命令，可生成`uboot-opensbi.itb`文件
`uboot-2022.10/tools/mkimage -f uboot-opensbi.its -r u-boot-opensbi.itb`

- 更改分区表

以partition_universal.json为例子，删掉uboot分区，修改opensbi分区名为opensbi-uboot，分区size最好为两者的合集，如下

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

- 更新刷机命令

以emmc为例子

```sh
fastboot stage factory/FSBL.bin
fastboot continue
#sleep to wait for uboot ready
#linux环境下
sleep 1
#windows环境下
#timeout /t 1 >null   
fastboot stage u-boot-opensbi.itb
fastboot continue

fastboot flash gpt partition_universal.json
#bootinfo_emmc.bin内容无作用，请参考3.1.3章节。但刷写步骤还需执行
fastboot flash bootinfo factory/bootinfo_emmc.bin
fastboot flash fsbl factory/FSBL.bin
fastboot flash env env.bin
fastboot flash opensbi-uboot u-boot-opensbi.itb
fastboot flash bootfs bootfs.img
fastboot flash rootfs rootfs.ext4
```

如果使用spacemit提供的titanflasher工具，则需将刷机包中的fastboot.yaml文件中的u-boot.itb名称为**u-boot-opensbi.itb**。

### 6.4 emmc启动定义fsbl的位置

只有emmc会有boot0、user区域的区别，nor、nand、sd卡等启动介质只有一个存储区域。硬件设定从boot区域加载bootinfo、fsbl，所以bootinfo/fsbl不能放到user区域
刷机时，emmc的bootinfo信息是固定在uboot代码里面的fastboot_oem_flash_bootinfo函数。目的是为了同一份partition_universal.json分区表可以用于制作卡启动等作用。
如果需要修改emmc的fsbl加载偏移，可直接修改以下代码：

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

对于emmc的boot0，fastboot刷机服务会对bootinfo/fsbl分区做特殊处理，将镜像文件写到boot0区域。具体可参考`uboot-2022.10/drivers/fastboot/fb_mmc.c::fastboot_mmc_flash_write`函数中的`if (strcmp(cmd, "bootinfo") == 0)`和`if (strcmp(cmd, CONFIG_FASTBOOT_MMC_BOOT1_NAME) == 0)`分支

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

### 6.5 sdcard上的bootinfo在0地址是否会与gpt表冲突

bootinfo_sd.bin是放在sd卡的0地址，不会与GPT表冲突（GPT表实际放在0地址偏移0x100之后），也不作为一个分区来识别

### 6.6 如何设置隐藏分区

分区表partition_universal.json支持隐藏分区的功能，hidden标签用来标注隐藏分区，刷机启动后隐藏分区不存在gpt表中。
存在隐藏分区的分区表，需要用titanflaser工具烧写镜像，或者执行fastboot flash gpt partition_universal.json后，才能使用fastboot命令烧写隐藏分区的镜像。
目前仅支持blk设备的隐藏分区功能，如emmc、ssd等。
bootinfo分区仅支持隐藏分区。

带隐藏分区的分区表参考如下：不添加hidden标签默认为显示分区。
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

v1.0.9版本之前的uboot仓库需要合入补丁，保存如下补丁文件，进入uboot仓库，打入补丁，重新编译
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
