# RTC

介绍rtc的功能和使用方法。

## 模块介绍

RTC(real-time-clock)简称实时时钟，主要用来计时，产生闹钟等；并维护系统时间；RTC一般有一个备用电池，所以即使系统关机掉电，rtc也能在备份电池的供电下继续正常工作。

### 功能介绍

![](static/rtc.png)  

1. dev/sysfs/proc 即接口层，负责向用户空间提供操作的节点及相关接口
2. rtc-core层为rtc驱动提供一套API，完成设备和驱动的注册等等
3. rtc驱动层，具体负责rtc驱动的实现，如设置时间，设置闹钟等

### 源码结构介绍

```
drivers/rtc/
├── class.c
├── dev.c
├── interface.c
├── Kconfig
├── lib.c
├── Makefile
├── proc.c
├── rtc-core.h
├── rtc-spt-pmic.c
├── sysfs.c
```

## 关键特性

### 特性

| 特性 |
| :-----|
| 支持日历、闹钟、秒计数 |

## 配置介绍

主要包括驱动使能配置和dts配置

### CONFIG配置

```
 CONFIG_RTC_DRV_SPT_PMIC:

 If you say yes here you will get support for the
 RTC of Spacemit spm8xxx PMIC.

 Symbol: RTC_DRV_SPT_PMIC [=y]
 Type  : tristate
 Defined at drivers/rtc/Kconfig:721
 Prompt: Spacemit spm8xxx RTC
 Depends on: RTC_CLASS [=y] && I2C [=y] && MFD_SPACEMIT_PMIC [=y]
 Location:
  -> Device Drivers
   -> Real Time Clock (RTC_CLASS [=y])
    -> Spacemit spm8xxx RTC (RTC_DRV_SPT_PMIC [=y])      
```

### dts配置

```
&i2c8 {
        pinctrl-names = "default";
        pinctrl-0 = <&pinctrl_i2c8>;
        status = "okay";

        spm8821@41 {
                compatible = "spacemit,spm8821";
                reg = <0x41>;
                interrupt-parent = <&intc>;
                interrupts = <64>;
                status = "okay";
    ....

                ext_rtc: rtc {
                        compatible = "pmic,rtc,spm8821";
                };
        };
};
```

## 接口描述

RTC驱动注册生成的字符设备节点/dev/rtcN，应用层使用时只需遵循Linux系统中标准的RTC编程方法即可  

### API介绍

```
#define devm_rtc_register_device(device) \
        __devm_rtc_register_device(THIS_MODULE, device)
int __devm_rtc_register_device(struct module *owner, struct rtc_device *rtc)  -- rtc设备注册

```

## 测试介绍

```
具体可参考：
Documentation/ABI/testing/rtc-cdev

下面用伪代码的方式阐述测试方法：

1. fd = open("/dev/rtcN", xxx)
2. ioctl(fd, RTC_SET_TIME, ...) --> 设置rtc时间
3. ioctl(fd, RTC_RD_TIME, ...)  --> 获得rtc时间
4. ioctl(fd, RTC_ALM_SET, ...)  --> 设置rtc闹钟
5. ioctl(fd, RTC_AIE_ON, ...)  --> 使能rtc
6. ioctl(fd, RTC_ALM_READ, ...)  --> 读取rtc闹钟 
```

## FAQ
