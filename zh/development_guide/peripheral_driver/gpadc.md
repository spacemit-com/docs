介绍gpadc的功能和使用方法。

# 模块介绍
IIO是linux内核中的一个子系统，专门用于处理工业控制、测量设备的数据采集和处理。IIO子系统支持的设备类型众多，包括模数转换器(ADC)、数模转换器(DAC)、加速度计、陀螺仪、惯性测量单元、温度传感器，我们本章节所介绍的gpadc是一个模数转换器，其嵌入到了我们的PMIC芯片中

## 功能介绍
![](static/gpadc.png)  

1. iio core，提供驱动程序和用户空间之间的接口、负责设备枚举、注册和管理  
2. IIO设备驱动程序，用于控制和读取特定IIO设备的代码  
3. IIO缓冲区，用于存储传感器和其他测量设备数据的内存区域  
4. IIO事件处理，用于处理来自传感器和其他测量设备的中断和事件

## 源码结构介绍

```
* IIO core
    drivers/iio/industrialio-core.c  
* IIO设备驱动程序
    drivers/iio/adc/k1x_adc.c
* IIO缓冲区
    drivers/iio/industrialio-buffer.c
* IIO事件处理
    drivers/iio/industrialio-event.c
```
# 关键特性

## 特性
| 特性 |
| :-----|
| 软件支持6路ADC |
| 12bit ADC转换精度，100Hz~50Khz采样率 |

# 配置介绍
主要包括驱动使能配置和dts配置

## CONFIG配置

```
Symbol: SPACEMIT_P1_ADC [=y]
Type  : tristate
Defined at drivers/iio/adc/Kconfig:1444
Prompt: Spacemit P1 adc driver
Depends on: IIO [=y] && MFD_SPACEMIT_PMIC [=y]
Location:
	-> Device Drivers
		-> Industrial I/O support (IIO [=y])
			-> Analog to digital converters
				-> Spacemit P1 adc driver (SPACEMIT_P1_ADC [=y])   
```
## dts配置
我们的gpadc内嵌在pmic中，使能gpadc需要配置两个dts点

1. gpadc channel pinctrl配置
```
pmic_pinctrl: pinctrl {
    compatible = "pmic,pinctrl,spm8821";
    gpio-controller;
    #gpio-cells = <2>;
    spacemit,npins = <6>;

    /* 假如使用channel2 作为adc的输入管脚 */
    gpadc2_pins: gpadc2-pins {
            pins = "PIN2";
            function = "adcin";
    };
};
```

2. adc 驱动使能配置
```
ext_adc: adc {
     compatible = "pmic,adc,spm8821";
};

```

# 接口描述
## 测试介绍
通过动态改变外部采样电压值来进行简单的测试，其软件读取电压值的方法如下：
```
cd /sys/bus/iio/devices/iio:device0
cat in_voltage2_raw
cat in_voltage2_scale

得到的两个节点做乘法运算就是得到的采样电压值(单位为mV)

```
## API介绍
```
struct iio_dev *iio_device_alloc(struct device *parent, int sizeof_priv); --- 申请iio_dev结构体
struct iio_dev *devm_iio_device_alloc(struct device *dev, int sizeof_priv) --- 申请 iio_dev
int iio_device_register(struct iio_dev *indio_dev) -- 注册iio设备
void iio_device_unregister(struct iio_dev *indio_dev) -- 注销iio设备

```

## Debug介绍
### sysfs
cd /sys/bus/iio/devices/iio:device0  --- iio框架目录
in_voltage2_raw  -- 读取的adc硬件寄存器的值
in_voltage2_scale  --- 读取的adc的精度
### debugfs