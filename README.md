---
slug: /
sidebar_position: 1
---

# 简介

Bianbu Linux 是 Spacemit Stone 系列芯片的 BSP，即 SDK。包含监管程序接口（OpenSBI）、引导加载程序（U-Boot/UEFI）、Linux 内核、根文件系统（包含各种中间件和库）以及示例等。其目标是为客户提供处理器 Linux 支持，并且可以开发驱动或应用。

## 主要组件

以下是Bianbu Linux的组件：

- OpenSBI
- U-Boot
- Linux Kernel
- Buildroot
- OpenWrt
- onnxruntime (with Hardware Accelerated)
- ai-support: AI demo 
- img-gpu-powervr: GPU DDK
- mesa3d
- QT 5.15 (with GPU enabled)
- k1x-vpu-firmware: Video Process Unit firmware
- k1x-vpu-test: Video Process Unit test program
- k1x-jpu: JPEG Process Unit API
- k1x-cam: CMOS Sensor and ISP API
- mpp: Media Process Platform
- FFmpeg (with Hardware Accelerated)
- GStreamer (with Hardware Accelerated)
- v2d-test: 2D Unit test program
- factorytest: factory test app

更多组件正在适配中。

## 快速指南

- [下载和编译](download_and_build.md)
- [设备管理](device_management.md)
- [方案管理](solution_management.md)

## OpenWrt
- [下载和编译](openwrt_quickstart.md)

## 进阶指南

- [外设驱动](development_guide/peripheral_driver/00-intro.md)
- [启动](development_guide/boot.md)
- [多媒体](development_guide/media.md)
