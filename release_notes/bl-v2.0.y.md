---
sidebar_position: 2
---

# Bianbu Linux 2.0更新说明

## v2.0rc7更新说明（开发中）

发布日期：2024-9-30

### 主要更新

- 修复卡量产失败的问题
- 修复rtl8852bs驱动内存越界访问题
- 修复rf驱动gpio设置报错问题
- 修复bootlogo闪烁问题
- 更新sdio delay参数，提升sdio稳定性
- 增加 P1 芯片的 ADC 功能

## v2.0rc6更新说明

发布日期：2024-9-10

### 主要更新

- 【MUSE Book】优化屏幕唤醒时间

## v2.0rc5更新说明

发布日期：2024-9-2

### 主要更新

- 更新rcpu固件

## v2.0rc4更新说明

发布日期：2024-8-29

### 主要更新

- 支持u-boot pwm控制LCD背光
- 支持u-boot 电量计框架
- 支持MIPI DSI屏jd9365dah3
- 支持Type-C芯片husb239
- 修复u-boot efi free memory问题
- 修复u-boot LCD低概率花屏的问题
- 修复关机过程中i2c异常的问题
- 优化内核加载模块速度

### 已知问题

- WiFi功能不稳定

## v2.0rc3更新说明

发布日期：2024-8-10

### 特性

#### 主要组件

- OpenSBI 1.3
- U-Boot 2022.10
- Linux 6.6.36
- buildroot 2023.02.9
- onnxruntime 1.15.1
- img-gpu-powervr 23.2: GPU DDK
- mesa3d 22.3.5
- QT 5.15 (with GPU enabled)
- k1x-vpu-firmware: Video Process Unit firmware
- k1x-vpu-test: Video Process Unit test program
- k1x-jpu: JPEG Process Unit API
- k1x-cam: CMOS Sensor and ISP API
- mpp: Media Process Platform
- FFmpeg 4.4.4 (with Hardware Accelerated)
- GStreamer 1.22.8 (with Hardware Accelerated)
- v2d-test: 2D Unit test program
- factorytest: factory test app

#### 主要驱动

**系统驱动**

- clk
- pinctrl
- timer
- watchdog
- RTC
- DMA
- msgbox
- spinlock
- TCM

**接口驱动**

- USB 2.0/3.0
- PCIe 2.1
- UART
- I2C
- SPI
- PWM
- CAN

**存储驱动**

- MMC (sdcard/eMMC/SDIO)
- SPI NOR
- NVMe SSD

**网络驱动**

- GMAC
- WiFi
- BT

**显示驱动**

- DPU
- GPU
- MIPI DSI
- HDMI 1.4

**多媒体驱动**

- VPU
- JPU
- V2D
- V4L2
- CMOS Sensor
- ISP
- I2S
- HDMI Audio

**功耗管理**

- cpufreq
- thermal
- PMIC

**加密驱动**

- AES
