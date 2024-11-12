---
sidebar_position: 6
slug: /development_guide/peripheral_driver
---

# 外设驱动

介绍文档编写目的、使用范围和相关人员。

## 编写目的

本文档介绍SpacemiT K1 CPU各接口和外设的板级配置、CONFIG配置、测试、调试接口和常见问题等信息，方便开发人员进行二次开发。

## 使用范围

适用于SpacemiT K1 CPU。

## 相关人员

- 驱动工程师
- 系统工程师

## 功能介绍

外设驱动（或设备驱动）是控制硬件设备与操作系统之间的接口。Linux系统外设驱动是一个模块化的组件，它充当了硬件和操作系统之间的桥梁，负责对硬件进行初始化、配置、管理、数据传输和错误处理等工作。

SpacemiT K1包含了各种丰富的IO能力，集成多套PCIe，USB，GMAC、SPI等接口，提供了全面的外设连接选型，该文档包含K1涉及到的高速扩展接口驱动、音视频接口驱动、工业扩展接口驱动、存储接口驱动等使用说明文档。

- 高速扩展接口驱动：PCIe、USB、GMAC等 
- 音视频接口驱动：DSI、HDMI、CSI等
- 工业扩展接口驱动：CAN-FD、UART、I2C、SPI、PWM等
- 存储接口驱动：SDHC、SPI-Flash等

## 快速索引
高速扩展接口驱动
- [GMAC](09-GMAC.md)  
- [USB](10-USB.md)  
- [PCIe](11-PCIe.md)   

音视频接口驱动
- [Display](12-Display.md)  
- [V2D](13-V2D.md)  
- [Audio](17-Audio.md)

工业扩展接口驱动
- [PWM](03-PWM.md)  
- [IR-RX](04-IR-RX.md)
- [UART](05-UART.md)
- [I2C](06-I2C.md)  
- [QSPI](07-QSPI.md) 
- [gpadc](gpadc.md)

存储接口驱动
- [SDHC](08-SDHC.md)

系统基础驱动
- [PINCTRL](01-PINCTRL.md) 
- [GPIO](02-GPIO.md)    
- [Clock](16-Clock.md)  
- [DMA](21-DMA.md)
- [RTC](rtc.md)

功耗子系统驱动
- [thremal](thermal.md)  
- [Cpufreq](15-Cpufreq.md)  
- [Stanby](\bianbu-linux-docs\zh\device\standby.md)

其他
- [CRYPTO](18-CRYPTO.md)