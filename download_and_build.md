---
sidebar_position: 2
---

# 下载和编译

## 开发环境

### 硬件配置

推荐配置：

- CPU：12th Gen Intel(R) Core(TM) i5或以上
- Memory：16GB或以上
- Disk：SSD，256GB或以上

### 操作系统

推荐Ubuntu 20.04或更新LTS版本，其他Linux发行版本没有测试。

### 安装依赖

Ubuntu 16.04 and 18.04:

```shell
sudo apt-get install git build-essential cpio unzip rsync file bc wget python3 libncurses5-dev libssl-dev dosfstools mtools u-boot-tools flex bison python3-pip
sudo pip3 install pyyaml
```

Ubuntu 20.04 or higher:

```shell
sudo apt-get install git build-essential cpio unzip rsync file bc wget python3 python-is-python3 libncurses5-dev libssl-dev dosfstools mtools u-boot-tools flex bison python3-pip
sudo pip3 install pyyaml
```

## 下载

使用repo（版本 >= 2.41）下载完整SDK。如果没有repo，参考[Git Repo 镜像使用帮助](https://mirrors.tuna.tsinghua.edu.cn/help/git-repo/)安装。

Bianbu Linux代码托管在Gitee上，下载前先参考[这篇文档](https://gitee.com/help/articles/4191)设置SSH Keys。

下载代码，例如下载`bl-v1.0.y`分支：

```shell
mkdir ~/bianbu-linux
cd ~/bianbu-linux
repo init -u git@gitee.com:bianbu-linux/manifests.git -b main -m bl-v1.0.y.xml
repo sync
repo start bl-v1.0.y --all
```

推荐提前下载buildroot依赖的第三方软件包，并在团队内部分发，避免主服务器网络拥塞。

```shell
wget -c -r -nv -np -nH -R "index.html*" http://archive.spacemit.com/buildroot/dl
```

目录结构：

```shell
├── bsp-src               # 存放linux kernel、uboot、opensbi源码
│   ├── linux-6.1
│   ├── opensbi
│   └── uboot-2022.10
├── buildroot             # buildroot主目录
├── buildroot-ext         # 客制化扩展，包含board、configs、package、patches子目录
├── Makefile              # 顶层Makefile
├── package-src           # 本地展开的应用或库源码目录
│   ├── ai-support
│   ├── drm-test
│   ├── img-gpu-powervr
│   ├── k1x-cam
│   ├── k1x-jpu
│   ├── k1x-vpu-firmware
│   ├── k1x-vpu-test
│   ├── mesa3d
│   ├── mpp
│   └── v2d-test
└── scripts               # 编译使用到的脚本
```

## 交叉编译

### 首次完整编译

首次编译，建议使用`make envconfig`完整编译。

修改了`buildroot-ext/configs/spacemit_<solution>_defconfig`，要使用`make envconfig`编译。

其他情况，使用`make`编译即可。

```shell
cd ~/bianbu-linux
make envconfig
Available configs in buildroot-ext/configs/:
  1. spacemit_k1_defconfig
  2. spacemit_k1_minimal_defconfig
  3. spacemit_k1_plt_defconfig
  4. spacemit_k1_v2_defconfig


your choice (1-4): 
```

编译Bianbu Linux 1.0版本，输入`1`，然后回车即开始编译。

编译Bianbu Linux 2.0版本，输入`4`。

编译过程可能需要下载一些第三方软件包，具体耗时和网络环境相关。如果提前下载buildroot依赖的第三方软件包，推荐硬件配置编译耗时约为1小时。

编译完成，可以看到：

```shell
Images successfully packed into /home/username/bianbu-linux/output/k1/images/bianbu-linux-k1.zip


Generating sdcard.img...................................
INFO: cmd: "mkdir -p "/home/username/bianbu-linux/output/k1/build/genimage.tmp"" (stderr):
INFO: cmd: "rm -rf "/home/username/bianbu-linux/output/k1/build/genimage.tmp"/*" (stderr):
INFO: cmd: "mkdir -p "/home/username/bianbu-linux/output/k1/images"" (stderr):
INFO: hdimage(sdcard.img): adding partition 'bootinfo' from 'factory/bootinfo_sd.bin' ...
INFO: hdimage(sdcard.img): adding partition 'fsbl' (in MBR) from 'factory/FSBL.bin' ...
INFO: hdimage(sdcard.img): adding partition 'env' (in MBR) from 'env.bin' ...
INFO: hdimage(sdcard.img): adding partition 'opensbi' (in MBR) from 'opensbi.itb' ...
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
Successfully generated at /home/username/work/bianbu-linux/output/k1/images/bianbu-linux-k1-sdcard.img
```

其中`bianbu-linux-k1.zip`适用于Titan Flasher，或者解压后用fastboot刷机；`bianbu-linux-k1-sdcard.img`为sdcard固件，解压后可以用dd命令或者[balenaEtcher](https://etcher.balena.io/)写入sdcard。

> Titan Flasher使用指南：[刷机工具使用指南](https://developer.spacemit.com/#/documentation?token=O6wlwlXcoiBZUikVNh2cczhin5d)

固件默认用户名：`root`，密码：`bianbu`。

### 配置

#### buildroot

配置：

```shell
make menuconfig
```

保存配置，默认保存到`buildroot-ext/configs/spacemit_k1_defconfig`：

```shell
make savedefconfig
```

#### linux

配置：

```shell
make linux-menuconfig
```

保存配置，默认保存到`bsp-src/linux-6.1/arch/riscv/configs/k1_defconfig`：

```shell
make linux-update-defconfig
```

#### u-boot

配置：

```shell
make uboot-menuconfig
```

保存配置，默认保存到`bsp-src/uboot-2022.10/configs/k1_defconfig`：

```shell
make uboot-update-defconfig
```

### 编译指定包

buildroot支持编译指定package，可以`make help`查看指南。

常用命令：

- 删除`<pkg>`的编译目录：`make <pkg>-dirclean`
- 编译`<pkg>`：`make <pkg>`

编译内核：

```shell
make linux-rebuild
```

编译u-boot：

```shell
make uboot-rebuild
```

编译指定包之后，可以单独下载到设备验证，或者编进固件：

```shell
make
```

### 单独编译

交叉编译器下载地址：`http://archive.spacemit.com/toolchain/`，解压即可使用。

设置环境变量：

```shell
export PATH=/path/to/spacemit-toolchain-linux-glibc-x86_64-v0.3.3/bin:$PATH
export CROSS_COMPILE=riscv64-unknown-linux-gnu-
export ARCH=riscv
```

#### 编译opensbi

```shell
cd /path/to/opensbi
make -j$(nproc) PLATFORM_DEFCONFIG=k1_defconfig PLATFORM=generic
```

#### 编译u-boot

```shell
cd /path/to/uboot-2022.10
make k1_defconfig
make -j$(nproc)
```

编译会根据`board/spacemit/k1-x/k1-x.env`生成`u-boot-env-default.bin`，对应分区表`env`分区的镜像。

#### 编译linux

```shell
cd /path/to/linux-6.1
make k1_defconfig
LOCALVERSION="" make -j$(nproc)
```

## 本地编译

将opensbi、u-boot或linux代码下载到Bianbu Desktop，即可本地编译。

### 编译opensbi

```shell
cd /path/to/opensbi
make -j$(nproc) PLATFORM_DEFCONFIG=k1_defconfig PLATFORM=generic
```

将`platform/generic/firmware/fw_dynamic.itb`用fastboot写入opensbi分区即可。

### 编译u-boot

```shell
cd /path/to/uboot-2022.10
make k1_defconfig
make -j$(nproc)
```

将`FSBL.bin`、`u-boot-env-default.bin`和`u-boot.itb`用fastboot写入对应分区即可。

### 编译linux

```shell
cd /path/to/linux-6.1
make k1_defconfig
LOCALVERSION="" make -j$(nproc)
```

将`arch/riscv/boot/Image.gz.itb`和`arch/riscv/boot/dts/spacemit/*.dtb`替换`bootfs`分区对应文件即可。
