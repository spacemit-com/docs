# 环境准备
以下文档基于ubuntu22.04描述

## 安装依赖
```
sudo apt install build-essential clang flex bison g++ gawk \
gcc-multilib g++-multilib gettext git libncurses-dev libssl-dev \
python3-distutils rsync unzip zlib1g-dev file wget jq device-tree-compiler 

```

# 下载代码
```
git clone https://gitee.com/bianbu-linux/openwrt.git -b bl-v1.0.y
```

# 拉取feeds
首次或想更新包时需要运行
```
cd openwrt
./scripts/feeds update -a 
./scripts/feeds install -a
```

# 固件编译
V=s输出详细日志
## SBC方案
```
cp feeds/spacemit_openwrt_feeds/spacemit_k1_defconfig .config
make -j12 V=s
```
固件位于bin/targets/spacemit/DEVICE_debX/*.zip

## NAS方案
```
cp feeds/spacemit_openwrt_feeds/spacemit_k1_nas_defconfig .config
make -j12 V=s
```
固件位于bin/targets/spacemit/DEVICE_MUSE-N1/*.zip


## 清理
全部清理，会把bin、build_dir、staging_dir、feeds、dl等目录删掉
```
make distclean
```
局部清理，会把编译输出目录bin、build_dir、staging_dir删掉
```
make dirclean
```



## 单包编译
以adb包为例说明
### 编译
```
make package/utils/adb/compile V=s
```
### 清理
```
make package/utils/adb/clean V=s
```


# 烧写
固件*.zip，使用Titan Flasher工具刷写至设备板载存储介质<br />
固件*sdcard.img，使用dd命令写至卡上，设备插卡上电即可实现卡启动

# 支持设备列表

## SBC方案
BPI-F3<br />
MUSE-Pi

## NAS方案
MUSE-N1

# FAQ
