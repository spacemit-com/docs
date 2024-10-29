---
sidebar_position: 2
---

# Download and Build

## Development Environment

### Hardware Configuration

Recommended configuration:

- CPU: 12th Gen Intel(R) Core(TM) i5 or above
- Memory: 16GB or above
- Disk: SSD, 256GB or above

### Operating System

Recommended Ubuntu 20.04 or newer LTS version, other Linux distributions have not been tested.

### Install Dependencies

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

## Download

Use repo (version >= 2.41) to download the complete SDK. If you don't have repo, refer to [Git Repo Mirror Usage Help](https://mirrors.tuna.tsinghua.edu.cn/help/git-repo/) for installation.

Bianbu Linux code is hosted on Gitee. Before downloading, refer to [this document](https://gitee.com/help/articles/4191) to set up SSH Keys.

To download the code, for example, to download the `bl-v1.0.y` branch:

```shell
mkdir ~/bianbu-linux
cd ~/bianbu-linux
repo init -u git@gitee.com:bianbu-linux/manifests.git -b main -m bl-v1.0.y.xml
repo sync
repo start bl-v1.0.y --all
```

Or to download the `bl-v2.0.y` branch:

```shell
mkdir ~/bianbu-linux-2.0
cd ~/bianbu-linux-2.0
repo init -u git@gitee.com:bianbu-linux/manifests.git -b main -m bl-v2.0.y.xml
repo sync
repo start bl-v2.0.y --all
```

It is recommended to download the third-party software packages required by buildroot in advance and distribute them within the team to avoid network congestion on the main server.

```shell
wget -c -r -nv -np -nH -R "index.html*" http://archive.spacemit.com/buildroot/dl
```

Directory structure:

```shell
├── bsp-src               # Stores linux kernel、uboot、opensbi source
│   ├── linux-6.1
│   ├── opensbi
│   └── uboot-2022.10
├── buildroot             # buildroot main directory
├── buildroot-ext         # Custom extensions, including board, configs, package, patches subdirectories
├── Makefile              # Top-level Makefile
├── package-src           # Directory for locally expanded application or library source code
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
└── scripts               # Scripts used for compilation
```

## Cross Compilation

### First Complete Compilation

For the first compilation, it is recommended to use `make envconfig` for a complete build.

If you have modified `buildroot-ext/configs/spacemit_<solution>_defconfig`, use `make envconfig` to compile.

In other cases, use `make` to compile.

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

To compile Bianbu Linux 1.0 version, enter `1` and press Enter to start compiling.

To compile Bianbu Linux 2.0 version, enter `4`.

The compilation process may require downloading some third-party software packages, and the time required depends on the network environment. If you have downloaded the third-party software packages required by buildroot in advance, the recommended hardware configuration compilation time is about 1 hour.

After the compilation is complete, you will see:

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

Among them, `bianbu-linux-k1.zip` is suitable for Titan Flasher, or you can unzip it and use fastboot to flash; `bianbu-linux-k1-sdcard.img` is the sdcard firmware, which can be written to the sdcard using the dd command or [balenaEtcher](https://etcher.balena.io/) after unzipping.

> Titan Flasher User Guide: [Flashing Tool User Guide](https://developer.spacemit.com/#/documentation?token=O6wlwlXcoiBZUikVNh2cczhin5d)

The default username for the firmware is `root`, and the password is `bianbu`.

### Configuration

#### buildroot

Config:

```shell
make menuconfig
```

Save the configuration, which is saved by default to `buildroot-ext/configs/spacemit_k1_defconfig`:

```shell
make savedefconfig
```

#### linux

Config:

```shell
make linux-menuconfig
```

Save the configuration, which is saved by default to `bsp-src/linux-6.1/arch/riscv/configs/k1_defconfig`:

```shell
make linux-update-defconfig
```

#### u-boot

Config:

```shell
make uboot-menuconfig
```

Save the configuration, which is saved by default to `bsp-src/uboot-2022.10/configs/k1_defconfig`:

```shell
make uboot-update-defconfig
```

### Compile Specific Packages

buildroot supports compiling specific packages. You can view the guide with `make help`.

Common commands:

- Delete the build directory of `<pkg>`: `make <pkg>-dirclean`
- Compile `<pkg>`: `make <pkg>`

To compile the kernel:

```shell
make linux-rebuild
```

To compile u-boot:

```shell
make uboot-rebuild
```

After compiling the specified package, you can download it to the device for verification or compile it into the firmware:

```shell
make
```

### Standalone Compilation

The cross-compiler download URL is `http://archive.spacemit.com/toolchain/`, and it can be used after decompression.

Set environment variables:

```shell
export PATH=/path/to/spacemit-toolchain-linux-glibc-x86_64-v0.3.3/bin:$PATH
export CROSS_COMPILE=riscv64-unknown-linux-gnu-
export ARCH=riscv
```

#### Compile opensbi

```shell
cd /path/to/opensbi
make -j$(nproc) PLATFORM_DEFCONFIG=k1_defconfig PLATFORM=generic
```

#### Compile u-boot

```shell
cd /path/to/uboot-2022.10
make k1_defconfig
make -j$(nproc)
```

The compilation will generate `u-boot-env-default.bin` based on `board/spacemit/k1-x/k1-x.env`, corresponding to the image of the env partition in the partition table.

#### Compile linux

```shell
cd /path/to/linux-6.1
make k1_defconfig
LOCALVERSION="" make -j$(nproc)
```
