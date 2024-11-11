介绍QSPI的功能和使用方法。
# 模块介绍
QSPI 是soc和外设之间的一种串行接口总线(spi), 支持4x模式。SPI有主、从两种模式，通常一个主设备(master)和一个或多个从设备(slave)连接。主设备选择一个从设备进行通信，完成数据交互。主设备提供时钟，读写操作都由主设备发起。k1 qspi暂时只支持主设备模式。

## 功能介绍
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

## 源码结构介绍
控制器驱动代码在drivers/spi目录下:  
```
|-- spi-k1x-qspi.c              #k1 qspi驱动
```

# 关键特性
## 特性
| 特性 | 特性说明 |
| :-----| :----|
| 通信协议 | 支持SSP/SPI/MicroWire/PSP协议 |
| 通信频率 | 最高频率支持102MHz, 最低频率支持13.25MHz |
| 通信倍数 | x1/x2/x4 |
| 支持外设 | 支持spi-nor和spi-nand |  

## 性能参数
### 通信频率

当前 qspi 控制器最大支持 102MHz，支持的通信频率列表

| 最大频率（MHz） | 分频系数(x)   | 实际频率 |
| --------------- | ------------- | -------- |
| 409             | 4, 5,6,7,8    | 409/x    |
| 307             | 2,3,4,5,6,7,8 | 307/x    |
| 245             | 3,4,5,6,7,8   | 245/x    |
| 223             | 3,4,5,6,7,8   | 223/x    |
| 106             | 2,3,4,5,6,7,8 | 106/x    |
| 495             | 5,6,7,8       | 495/x    |
| 189             | 2,3,4,5,6,7,8 | 189/x    |

### 通信倍速

qspi 通信倍速支持 x1/x2/x4。

测试方法  
可以通过示波器或者逻辑分析测试sck信号频率

# 配置介绍
主要包括驱动使能配置和dts配置
## CONFIG配置

## dts配置
### pinctrl

查看方案原理图，找到 qspi 使用的 pin 组。参考 1.2.2 节，确定 qspi 使用的 pin 组。

假设 qspi 可以直接采用 k1-x_pinctrl.dtsi 中定义 pinctrl_qspi 组。

### spi 设备配置

需要确认 spi 设备类型，qspi 与 spi 设备通信频率和倍速。

#### 设备类型

确认 qspi 下连接的 spi 设备类型，是 spi-nor 还是 spi-nand。

#### 通信频率

qspi 控制器和 spi 设备最大通信速率。  
当前 qspi 控制器支持的通信频率列表见“性能参数”--->"通信频率"中的频率列表


#### 通信倍速

qspi 通信倍速支持 x1/x2/x4。

#### spi 设备 dts

以 spi nor 为例，采用最大通信频率 26.5MHz，发送和接收都采用 x4 通信。

qspi 控制器默认最大通信频率 26.5MHz，最大通信频率为 26.5MHz 时，方案 dts 可以不用配置"k1x,qspi-freq"。

```c
&qspi {
    k1x,qspi-freq = <26500000>;
 
    flash@0 {
                compatible = "jedec,spi-nor";
                reg = <0>;
                spi-max-frequency = <26500000>;
                spi-tx-bus-width = <4>;
                spi-rx-bus-width = <4>;
                m25p,fast-read;
                broken-flash-reset;
                status = "okay";
        };
};
```

### dts

#### spi-nor

综合上述信息，qspi 连接 spi-nor flash，最大通信频率为 26.5MHz，且采用 x4 通信。

方案 dts 配置如下

```c
&qspi {
        pinctrl-names = "default";
        pinctrl-0 = <&pinctrl_qspi>;
        status = "okay";
        k1x,qspi-freq = <26500000>;

        flash@0 {
                compatible = "jedec,spi-nor";
                reg = <0>;
                spi-max-frequency = <26500000>;
                spi-tx-bus-width = <4>;
                spi-rx-bus-width = <4>;
                m25p,fast-read;
                broken-flash-reset;
                status = "okay";
        };
};
```

#### spi-nand

qspi 连接 spi-nand flash，最大通信频率为 26.5MHz，且采用 x4 通信。

方案 dts 配置可以参考 spi-nor，只用修改 flash 设备节点。

```c
&qspi {
        pinctrl-names = "default";
        pinctrl-0 = <&pinctrl_qspi>;
        status = "okay";
        k1x,qspi-freq = <26500000>;

        spinand: spinand@0 {
                compatible = "spi-nand";
                spi-max-frequency = <26500000>;
                reg = <0>;
                spi-tx-bus-width = <4>;
                spi-rx-bus-width = <4>;
                status = "okay";
        };
};
```
# 接口描述
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
## API介绍
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
/sys/kernel/debug/spi-nor/spi4.0
```
|-- capabilities
`-- params
# cat capabilities
Supported read modes by the flash
 1S-1S-1S
  opcode        0x03
  mode cycles   0
  dummy cycles  0
 1S-1S-1S (fast read)
  opcode        0x0b
  mode cycles   0
  dummy cycles  8

Supported page program modes by the flash
 1S-1S-1S
  opcode        0x02
# cat params
name            w25q32
id              ef 40 16 00 00 00
size            4.00 MiB
write size      1
page size       256
address nbytes  3
flags           BROKEN_RESET | HAS_16BIT_SR

opcodes
 read           0x0b
  dummy cycles  8
 erase          0x20
 program        0x02
 8D extension   none

protocols
 read           1S-1S-1S
 write          1S-1S-1S
 register       1S-1S-1S

erase commands
 20 (4.00 KiB) [0]
 d8 (64.0 KiB) [1]
 c7 (4.00 MiB)

sector map
 region (in hex)   | erase mask | flags
 ------------------+------------+----------
 00000000-003fffff |     [01  ] |
```
# FAQ
