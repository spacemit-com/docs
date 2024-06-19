---
sidebar_position: 1
---

# 更新说明

## v1.0.3更新说明

发布日期：2024-6-19

### 主要更新

- linux-6.1支持codec芯片es8316
- u-boot支持SPI驱动
- 【BPI-F3】支持风扇降温功能
- 【Muse Pi】支持MIPI-CSI Sensor
- 【Muse Book】支持u-boot阶段显示logo
- 【Muse Book】更新产测工具
- 【Muse Box】支持声卡耳机、Mic检测功能
- 【Muse Box】支持DP显示功能（注：不支持热插拔）
- 修复长时间休眠无法唤醒问题
- 修复低概率创建声卡失败问题
- 修复低概率i2c通信timeout问题
- 修复pcie 2 lane无法自适应支持1 lane问题
- 修复SDIO WiFi低温概率性CRC错误问题
- 【Muse Book】修复获取不到显示器EDID问题

## v1.0更新说明

发布日期：2024-5-30

### 特性

#### 主要组件

- OpenSBI 1.3
- U-Boot 2022.10
- Linux 6.1
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

### 已知问题

- Suspend to ram功能尚不完善
