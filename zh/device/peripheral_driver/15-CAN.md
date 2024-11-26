# CAN

介绍CAN的配置和调试方式

## 模块介绍  

CAN（Controller Area Network，控制器局域网络）是一种用于控制器和设备之间进行通信的串行通信协议。主要用于汽车工业，工业自动化、医疗设备、航空航天、机器人等多个领域。

### 功能介绍  

![](static/can.png)

can控制器实现了基于CAN2.0和CANFD协议的报文收发，包括标准数据帧，标准远程帧，扩展数据帧等。can驱动通过网络设备接口注册为网络设备。在用户层可以通过指定网络工具或接口完成can驱动调用实现报文收发。

### 源码结构介绍

CAN控制器驱动代码在drivers/net/can目录下：  

```  
drivers/net/can  
|--dev.c                     #内核can框架代码，包含计算波特率参数，注册can设备等
|--flexcan/                #k1 can驱动
 |--flexcan-core.c
 |--flexcan.h
```  

## 关键特性  

### 特性

| 特性 | 特性说明 |
| :-----| :----|
| 支持CANFD | 支持CANFD协议，兼容CAN2.0 |
| 支持最大64B数据 | CANFD协议支持8，16，32，64B数据传输 |

### 性能参数

支持最高8M数据域波特率

## 配置介绍

主要包括驱动使能配置和dts配置

### CONFIG配置

CONFIG_CAN_DEV
此为内核平台can框架提供支持，支持k1 can驱动情况下，应为Y

```
Symbol: CAN_DEV [=y]
Device Drivers
    -> Network device support (NETDEVICES [=y]) 
  -> CAN Device Drivers (CAN_DEV [=y])
```

在支持平台层can框架后，配置CONFIG_CAN_FLEXCAN为Y，支持k1 can驱动

```
Symbol: CAN_FLEXCAN [=y]
    -> CAN device drivers with Netlink support (CAN_NETLINK [=y])
  -> Support for Freescale FLEXCAN based chips (CAN_FLEXCAN [=y])
```

### dts配置

在k1平台，can控制器部分不包含收发器，控制器对外的接口为TX和RX

#### pinctrl

可查看linux仓库的arch/riscv/boot/dts/spacemit/k1-x_pinctrl.dtsi，参考已配置好的can节点配置，如下：

```dts
    pinctrl_can_0: can_0_grp {
        pinctrl-single,pins = <
            K1X_PADCONF(GPIO_75, MUX_MODE3, (EDGE_NONE | PULL_UP | PAD_3V_DS4))     /* can_tx0 */
            K1X_PADCONF(GPIO_76, MUX_MODE3, (EDGE_NONE | PULL_UP | PAD_3V_DS4))     /* can_rx0 */
     >;
    };
```

#### dtsi配置示例

dtsi中配置can控制器基地址和时钟复位资源，正常情况无需改动

```dts
    flexcan0: fdcan@d4028000 {
        compatible = "spacemit,k1x-flexcan";
        reg = <0x0 0xd4028000 0x0 0x4000>;
        interrupts = <16>;
        interrupt-parent = <&intc>;
        clocks = <&ccu CLK_CAN0>,<&ccu CLK_CAN0_BUS>;
        clock-names = "per","ipg";
        resets = <&reset RESET_CAN0>;
        fsl,clk-source = <0>;
        status = "disabled";
    };
```

#### dts配置示例

dts完整配置，如下所示
可选择配置时钟频率为20M，40M，80M以支持不同波特率

```dts
&flexcan0 {
 pinctrl-names = "default";
 pinctrl-0 = <&pinctrl_can_0>;
 clock-frequency = <80000000>;
 status = "okay";
};
```

## 接口描述

### API介绍

can驱动主要实现了发送接收报文的接口
常用：

```
static int flexcan_open(struct net_device *dev)  
```

开启can设备时调用

```
static netdev_tx_t flexcan_start_xmit(struct sk_buff *skb, struct net_device *dev) 
```

can设备开始传输时调用
配置can传输时波特率的参数为初始化驱动时保存在驱动私有数据结构体中

### Demo示例

## Debug介绍

测试方法详情见测试介绍章节

## 测试介绍

基于k1平台可以外接can收发器进行测试，通讯的另一端一般选择USBCAN分析仪连接电脑模拟can设备，由于通信的另一端设备和用法不确定，这里主要介绍k1段的测试用法  
1.查看can设备是否加载成功  
ifconfig -a  
2.k1配置can的仲裁域和数据域波特率  
ip link set can0 type can bitrate 125000 dbitrate 250000 berr-reporting on fd on  
3.打开can设备(同时另一端准备接收)  
ip link set can0 up  
4.k1端发送报文  
cansend格式：cansend can-dev id#data  
eg：cansend can0 123##3.11223344556677881122334455667788aabbccdd  
5.k1端接收报文(另一端发送)  
candump can0

## FAQ
