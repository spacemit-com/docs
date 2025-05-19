# CAN

介绍CAN的配置和调试方式

## 模块介绍  

CAN（Controller Area Network，控制器局域网络）是一种用于控制器和设备之间进行通信的串行通信协议。主要用于汽车工业，工业自动化、医疗设备、航空航天、机器人等多个领域。

### 功能介绍  

![cat](static/can.png)

can控制器实现了基于CAN2.0和CANFD协议的报文收发，包括标准数据帧，标准远程帧，扩展数据帧等。can驱动通过网络设备接口注册为网络设备。在用户层可以通过指定网络工具或接口完成can驱动调用实现报文收发。

### 源码结构介绍

CAN控制器驱动代码在drivers/net/can目录下：  

```c
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

```shell
Symbol: CAN_DEV [=y]
Device Drivers
    -> Network device support (NETDEVICES [=y]) 
  -> CAN Device Drivers (CAN_DEV [=y])
```

在支持平台层can框架后，配置CONFIG_CAN_FLEXCAN为Y，支持k1 can驱动

```shell
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
/*can0*/
&flexcan0 {
	 pinctrl-names = "default";
	 pinctrl-0 = <&pinctrl_can_0>;
	 clock-frequency = <80000000>;
	 status = "okay";
};

/*rcan*/
&r_flexcan {
       pinctrl-names = "default";
       clock-frequency = <80000000>;
       status = "okay";
       pinctrl-0 = <&pinctrl_r_can_0>;
};

```

## 接口描述

### API介绍

can驱动主要实现了发送接收报文的接口
常用：

```c
static int flexcan_open(struct net_device *dev)  
```

开启can设备时调用

```c
static netdev_tx_t flexcan_start_xmit(struct sk_buff *skb, struct net_device *dev) 
```

can设备开始传输时调用
配置can传输时波特率的参数为初始化驱动时保存在驱动私有数据结构体中

### Demo示例

## Debug介绍


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

## 测试介绍

基于k1平台可以外接can收发器进行测试，通讯的另一端一般选择USBCAN分析仪连接电脑模拟can设备，由于通信的另一端设备和用法不确定，这里主要介绍k1 的测试用法。以下将以MUSE Pi开发板为例，基于bianbu-linux系统做demo演示，dts配置请参考dts配置示例章节。

- 基于MUSE Pi连接can设备

![alt text](static/can_image_1.png)

pin脚方向如上图所示，从上往下的绿色箭头，分别为
rcan tx(gpio47, 26 pin接口的8pin)、rcan rx(gpio48, 26 pin接口的10pin);
can0 tx(gpio75, 26 pin接口的23pin)、can0 rx(gpio 76,26 pin接口的24pin)

- pc端安装can软件，以及接入pc can(可以接入两个can外设相互收发)。本次使用的是PEAK的PC can，[PEAK官网](https://www.peak-system.com)
下图所示为rcan的接线，can0的接线类似。

![alt text](static/can_image_2.jpg)

- 查看can设备是否加载成功

```shell
# ifconfig -a
can0      Link encap:UNSPEC  HWaddr 00-00-00-00-00-00-00-00-00-00-00-00-00-00-00-00  
          NOARP  MTU:16  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:10 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
          Interrupt:77 

can1      Link encap:UNSPEC  HWaddr 00-00-00-00-00-00-00-00-00-00-00-00-00-00-00-00  
          inet addr:169.254.185.103  Mask:255.255.0.0
          UP RUNNING NOARP  MTU:72  Metric:1
          RX packets:4226044 errors:1411370 dropped:0 overruns:0 frame:1411370
          TX packets:1428220 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:10 
          RX bytes:50946992 (48.5 MiB)  TX bytes:28564400 (27.2 MiB)
          Interrupt:255 


```

- k1配置can的仲裁域和数据域波特率，两个can设备必须要配置成相同的仲裁、数据波特率才能正常收发数据。

```shell
ip link set can1 up type can bitrate 4000000 sample-point 0.75 dbitrate 8000000 sample-point 0.8 fd on

#接收数据
candump can1
```

- 打开另外一个can设备作为数据发送端(可以是pc can，也可以是开发板的另外一个can设备，这里以另外一个can设备发送数据，PC can可自行验证)

```shell
cansend can1 456##3.8877665544332211aabbccddeeffaabbaabb
```

- 停止can设备

```shell
ifconfig can1 down
```

## FAQ

- 在MUSE-Pi开发版调试rcan，需要关闭以下的dts引脚配置

```dts
diff --git a/arch/riscv/boot/dts/spacemit/k1-x_MUSE-Pi.dts b/arch/riscv/boot/dts/spacemit/k1-x_MUSE-Pi.dts
index 9107d43c3091..a34272ce8318 100644
--- a/arch/riscv/boot/dts/spacemit/k1-x_MUSE-Pi.dts
+++ b/arch/riscv/boot/dts/spacemit/k1-x_MUSE-Pi.dts
@@ -578,12 +578,12 @@ &range GPIO_124 1 (MUX_MODE0 | EDGE_NONE | PULL_UP   | PAD_1V8_DS2)
                &range GPIO_125 3 (MUX_MODE0 | EDGE_NONE | PULL_DOWN | PAD_1V8_DS2)
        >;
 
-       pinctrl_rcpu: pinctrl_rcpu_grp {
-               pinctrl-single,pins = <
-                       K1X_PADCONF(GPIO_47, MUX_MODE1, (EDGE_NONE | PULL_UP | PAD_3V_DS4))     /* r_uart0_tx */
-                       K1X_PADCONF(GPIO_48, MUX_MODE1, (EDGE_NONE | PULL_UP | PAD_3V_DS4))     /* r_uart0_rx */
-               >;
-       };
+       /* pinctrl_rcpu: pinctrl_rcpu_grp { */
+       /*      pinctrl-single,pins = < */
+       /*              K1X_PADCONF(GPIO_47, MUX_MODE1, (EDGE_NONE | PULL_UP | PAD_3V_DS4))     /1* r_uart0_tx *1/ */
+       /*              K1X_PADCONF(GPIO_48, MUX_MODE1, (EDGE_NONE | PULL_UP | PAD_3V_DS4))     /1* r_uart0_rx *1/ */
+       /*      >; */
+       /* }; */
 
        pinctrl_gmac0: gmac0_grp {
                pinctrl-single,pins =<
@@ -1062,7 +1062,7 @@ &vi {
 
 &rcpu {
        pinctrl-names = "default";
-       pinctrl-0 = <&pinctrl_rcpu>;
+       /* pinctrl-0 = <&pinctrl_rcpu>; */
        mboxes = <&mailbox 0>, <&mailbox 1>;
        mbox-names = "vq0", "vq1";
        memory-region = <&rcpu_mem_0>, <&vdev0vring0>, <&vdev0vring1>, <&vdev0buffer>, <&rsc_table>, <&rcpu_mem_snapshots>;

```
