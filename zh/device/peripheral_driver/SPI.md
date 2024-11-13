# SPI

介绍SPI的功能和使用方法。

## 模块介绍

SPI 是soc和外设之间的一种串行接口总线(spi), 只支持1x模式。SPI有主、从两种模式，通常一个主设备(master)和一个或多个从设备(slave)连接。主设备选择一个从设备进行通信，完成数据交互。主设备提供时钟，读写操作都由主设备发起。k1 spi暂时只支持主设备模式。

### 功能介绍

![](static/linux_spi.png)  

Linux spi驱动框架分为三部分: spi core、spi控制器驱动和spi设备驱动。  
spi core主要作用:

- spi总线和spi_master类注册  
- spi控制器添加和删除  
- spi设备添加和删除  
- spi设备驱动注册与注销  

spi 控制器驱动:

- spi master控制器驱动，对spi master控制器进行操作

spi 设备驱动

- spi device驱动

### 源码结构介绍

控制器驱动代码在drivers/spi目录下:  

```
|-- spi-k1x.c              #k1 spi驱动
```

## 关键特性

### 特性

| 特性 | 特性说明 |
| :-----| :----|
| 通信协议 | 支持SSP/SPI/MicroWire/PSP协议 |
| 通信频率 | 最高频率支持53MHz, 最低频率支持6.3kbps |
| 通信倍数 | x1 | 
| 支持外设 | 支持spi-nor和spi-nand | 

### 性能参数

#### 通信频率

最高频率支持53MHz, 最低频率支持6.3kbps

#### 通信倍速

spi 通信倍速支持 x1。

测试方法  
可以通过示波器或者逻辑分析测试sck信号频率

## 配置介绍

主要包括驱动使能配置和dts配置

### CONFIG配置

CONFIG_SPI 为SPI总线协议提供支持，默认情况，此选项为Y
```
Device Drivers
        SPI support (SPI [=y])
```

CONFIG_SPI_K1X 为K1 spi控制器驱动提供支持，默认情况下，此选型为Y
```
Device Drivers
        SPI support (SPI [=y])
                K1X SPI Controller (SPI_K1X [=y])

```

### dts配置

#### pinctrl

查看方案原理图，找到 spi 使用的 pin 组。参考pinctrl章节，确定 spi 使用的 pin 组。

假设 spi3 可以直接采用 k1-x_pinctrl.dtsi 中定义 pinctrl_ssp3_0 组。

#### spi 设备配置

需要确认 spi 设备类型，spi 与 spi 设备通信频率。

##### 设备类型

确认 spi 下连接的 spi 设备类型，是 spi-nor 还是 spi-nand。

##### 通信频率

spi 控制器和 spi 设备最大通信速率。  

##### 通信倍速

qspi 通信倍速支持 x1。

##### spi 设备 dts

以 spi nor 为例，采用最大通信频率 26MHz，发送和接收都采用 x1 通信。

#### dts示例
```c
&spi3 {
        pinctrl-names = "default";
        pinctrl-0 = <&pinctrl_ssp3_0>;
        k1x,ssp-disable-dma;
        status = "okay";
        k1x,ssp-clock-rate = <26000000>;

        flash@0 {
                compatible = "jedec,spi-nor";
                reg = <0>;
                spi-max-frequency = <26000000>;
                m25p,fast-read;
                broken-flash-reset;
                status = "okay";
        };
};
```

## 接口介绍

### API介绍

设备驱动注册与注销

```
int __spi_register_driver(struct module *owner, struct spi_driver *sdrv);  
void spi_unregister_driver(struct spi_driver *sdrv);
```

数据传输API

- 初始化spi_message

```
void spi_message_init(struct spi_message *m);
```

- 添加spi_transfer到spi_message的transfer列表

```
void spi_message_add_tail(struct spi_transfer *t, struct spi_message *m);
```

- write数据

```
int spi_write(struct spi_device *spi, const void *buf, size_t len);
```

- read数据

```
int spi_read(struct spi_device *spi, void *buf, size_t len);
```

- 同步传输spi_message

```
int spi_sync(struct spi_device *spi, struct spi_message *message);
```

## Debug介绍

### sysfs

查看系统spi总线设备和驱动信息
/sys/bus/spi

```
|-- devices                 //spi总线上的设备
|-- drivers                 //spi总线上注册的设备驱动
|-- drivers_autoprobe
|-- drivers_probe
`-- uevent
```

### debugfs

用于查看系统中spi设备信息
/sys/kernel/debug/spi-nor/spi3.0

## 测试介绍

### spi-nand/nor读写速率测试

打开CONFIG_MTD_TESTS

```
Device Drivers
         Memory Technology Device (MTD) support (MTD [=y])
                MTD tests support (DANGEROUS) (MTD_TESTS [=m])   
```

测试命令

```
insmod mtd_speedtest.ko dev=0   #0表示spi-nand/nor的mtd设备号
```

## FAQ