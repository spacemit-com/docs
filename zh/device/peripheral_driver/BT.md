# BT 
介绍BT的移植和使用方法。

## 模块介绍
K1平台上主要通过外部BT模块来实现BT功能，支持UART，USB等接口的蓝牙模块。
### 功能介绍
![](static/bt.png)

BT框架图可以分为以下几个层次：  

### 源码结构介绍

BT相关的源码可以分为以下几个部分：
1. 蓝牙协议栈，以bluez协议栈为例，代码分为两部分，一部分在内核空间，一部分在用户空间。
2. BT HCI驱动，主要实现HCI层。
3. 平台相关部分，主要实现模组供电以及使能等相关接口，供rfkill驱动调用。
4. 接口驱动，主要实现BT数据传输接口功能，如UART，USB等接口。

BLUEZ协议栈源码在以下目录：
```
drivers/net/bluetooth
|-- af_bluetooth.c
|-- af_bluetooth.o
|-- bnep                #Bluetooth网络封装协议
│   |-- bnep.h
│   |-- core.c
│   |-- Kconfig
│   |-- Makefile
│   |-- netdev.c
│   |-- sock.c
|-- hci_codec.c
|-- hci_codec.h
|-- hci_conn.c
|-- hci_core.c          #Hci core实现
|-- hci_debugfs.c
|-- hci_debugfs.h
|-- hci_event.c
|-- hci_request.c
|-- hci_request.h
|-- hci_sock.c
|-- hci_sync.c
|-- hci_sysfs.c
|-- hidp                #Bluetooth hid实现
│   |-- core.c
│   |-- hidp.h
│   |-- Kconfig
│   |-- Makefile
│   |-- sock.c
|-- iso.c
|-- Kconfig
|-- l2cap_core.c        #l2cap core实现
|-- l2cap_sock.c
|-- lib.c
|-- Makefile
|-- mgmt.c              #mgmt蓝牙管理实现
|-- mgmt_config.c
|-- mgmt_config.h
|-- mgmt_util.c
|-- mgmt_util.h
|-- rfcomm              #rfcomm协议
│   |-- core.c
│   |-- Kconfig
│   |-- Makefile
│   |-- sock.c
│   |-- tty.c
|-- sco.c
|-- selftest.c
|-- selftest.h
|-- smp.c
|-- smp.h
```
HCI驱动的源码在以下目录：
```
drivers/bluetooth
|-- btbcm.c             #broadcom厂商实现
|-- btrtl.c             #realtek厂商实现
|-- btusb.c             #hci uart实现
|-- hci_h4.c            #hci h4实现
|-- hci_h5.c            #hci h5实现
|-- hci_ldisc.c         #蓝牙hci的线路规程

```
平台相关的源码：
```
drivers/soc/spacemit/spacemit-rf
|-- spacemit-pwrseq.c   #WIFI和蓝牙等公共部分实现
|-- spacemit-wlan.c     #WIFI供电，gpio以及时钟相关接口实现
|-- spacemit-bt.c       #BT供电，gpio以及时钟相关接口实现
```

接口相关的源码参考各个接口驱动说明文档。

## 关键特性
### 平台uart接口特性
| 特性 | 特性说明 |
| :-----| :----|
| 支持4线流控 | 最高支持3.6Mbps |
| 支持DMA | 支持DMA传输模式 |

### 模组性能规格
| 模组型号 | 规格 |
| :-----| :----|
| rtl8852bs | 支持Bluetooth 5.2 |
|             | 支持Bluetooth 2.0 UART HCI H4/H5 |
| aic8800d80 | 支持Bluetooth 5.3 |
|             | 支持Bluetooth 2.0 UART HCI H4 |

## 配置介绍
主要包括驱动使能配置和dts配置
### CONFIG配置

协议栈配置
```
Networking support (NET [=y])
        Bluetooth subsystem support (BT [=y])
                Bluetooth Classic (BR/EDR) features (BT_BREDR [=y])
                        RFCOMM protocol support (BT_RFCOMM [=y])
                                RFCOMM TTY support (BT_RFCOMM_TTY [=y])
                        BNEP protocol support (BT_BNEP [=y])
                        HIDP protocol support (BT_HIDP [=y])
                Bluetooth Low Energy (LE) features (BT_LE [=y])
        Export Bluetooth internals in debugfs (BT_DEBUGFS [=y])
```
UART HCI配置
```
Networking support (NET [=y])
        Bluetooth subsystem support (BT [=y])
                Bluetooth device drivers
                        HCI UART driver (BT_HCIUART [=y])
                                UART (H4) protocol support (BT_HCIUART_H4 [=y])
                        Three-wire UART (H5) protocol support (BT_HCIUART_3WIRE [=y])
                        Realtek protocol support (BT_HCIUART_RTL [=y]) 
```
默认支持H4和H5，其中Realtek的蓝牙串口走的是H5协议。

USB HCI配置
```
Networking support (NET [=y])
        Bluetooth subsystem support (BT [=y])
                Bluetooth device drivers
                        HCI USB driver (BT_HCIBTUSB [=m])
                                Broadcom protocol support (BT_HCIBTUSB_BCM [=y])
                                Realtek protocol support (BT_HCIBTUSB_RTL [=y])  
```
BT_HCIBTUSB_BCM和BT_HCIBTUSB_RTL分别对应Broadcom和Realtek的支持。

AVRCP配置
```
Device Drivers
        Input device support
                Generic input layer (needed for keyboard, mouse, ...) (INPUT [=y])
                        Miscellaneous devices (INPUT_MISC [=y])
                                User level driver support (INPUT_UINPUT [=y])
```
如果要把 AVRCP 的按键值等信息通过 input device 送给用户态程序，则需要打开INPUT_UINPUT。

HOGP配置
```
Device Drivers
        HID bus support (HID_SUPPORT [=y])
                HID bus core support (HID[=y])
                        User-space I/O driver support for HID subsystem (UHID [=y]) 
```
如果要把HoG的KEY_1, KEY_2, KEY_ESC等按键值通过input device送给用户态程序，则需要打开UHID。

平台配置
```
Device Drivers
        SOC (System On Chip) specific Drivers
                Spacemit rfkill driver (SPACEMIT_RFKILL [=y])
```
CONFIG_SPACEMIT_RFKILL 为BT模组提供平台相关支持，默认情况，此选项为Y

### dts配置
#### uart配置

方案上一般 uart2 用于蓝牙，对应uart2，实际对应节点为ttyS1。

```
&uart2 {
        pinctrl-names = "default";
        pinctrl-0 = <&pinctrl_uart2>;
        status = "okay";
};
```
#### uart2 pinctrl配置
蓝牙pinctl配置以实际硬件为准，默认开启流控。
```
pinctrl_uart2: uart2_grp {
        pinctrl-single,pins =<
                K1X_PADCONF(GPIO_21, MUX_MODE1, (EDGE_NONE | PULL_UP | PAD_1V8_DS2))	/* uart2_txd */
                K1X_PADCONF(GPIO_22, MUX_MODE1, (EDGE_NONE | PULL_UP | PAD_1V8_DS2))	/* uart2_rxd */
                K1X_PADCONF(GPIO_23, MUX_MODE1, (EDGE_NONE | PULL_UP | PAD_1V8_DS2))	/* uart2_cts_n */
                K1X_PADCONF(GPIO_24, MUX_MODE1, (EDGE_NONE | PULL_UP | PAD_1V8_DS2))	/* uart2_rts_n */
        >;
};
```

#### 平台部分dts配置

平台完整方案配置如下：
```
rf_pwrseq: rf-pwrseq {
        compatible = "spacemit,rf-pwrseq";
        //vdd-supply = <&ldo_7>;
        //vdd_voltage = <3300000>;
        io-supply = <&dcdc_3>;
        io_voltage = <1800000>;
        pwr-gpios  = <&gpio 67 0>;
        status = "okay";

        wlan_pwrseq: wlan-pwrseq {
                compatible = "spacemit,wlan-pwrseq";
                regon-gpios = <&gpio 116 0>;
                interrupt-parent = <&pinctrl>;
                interrupts = <268>;
                pinctrl-names = "default";
                pinctrl-0 = <&pinctrl_wlan_wakeup>;
        };

        bt_pwrseq: bt-pwrseq {
                compatible = "spacemit,bt-pwrseq";
                reset-gpios     = <&gpio 63 0>;
        };
};
```

目前市面上很多模组都是WIFI和蓝牙二合一，WIFI和蓝牙的供电部分很多都是共用的，共用的部分建议放到rf_pwrseq中进行配置，仅影响蓝牙的部分，放到bt_pwrseq中进行配置。

单蓝牙模组只需要配置bt_pwrseq即可，不需要配置rf_pwrseq，但要使能rf_pwrseq节点。

在打开蓝牙的电源时会先使能共用部分的电源以及gpio状态，平台会维护对应的引用计数，关闭时只有WIFI和蓝牙都关闭后才会真正关闭共用部分的电源以及gpio状态。

rf_pwrseq：
- vdd-supply是配置模组的供电，具体按实际硬件配置。
- vdd_voltage用于设定模组供电的电压。
- io-supply是配置模组io的供电，具体按实际硬件配置。
- io_voltage用于设定模组io供电的电压。
- pwr-gpios是模组使能脚，配置后会默认拉高，支持多个gpio的配置。
- clock是模组共用的时钟配置。
- power-on-delay-ms是设置模组上电后的延时，默认是100ms。

bt_pwrseq：
- reset-gpios是蓝牙的使能脚，使能蓝牙对应的rfkill时会将该gpio拉高。
- clock是蓝牙的时钟配置。
- power-on-delay-ms是设置蓝牙上电后的延时，默认是10ms。

## 接口描述
### UART hciattach

hciattach 是 BlueZ 为 UART 接口蓝牙控制器提供的初始化工具，USB 接口的蓝牙直接忽略。

如果是Realtek Bluetooth UART，需要使用rtk_hciattach目录下源码编译生成的rtk_hciattach。

```
rtk_hciattach -n -s 115200 ttyS1 rtk_h5 &
```
如果是AIC8800 Bluetooth UART, 使用hciattach如下命令进行初始化
```
hciattach -s 1500000 /dev/ttyS1 any 1500000 flow nosleep
```
### BT基本功能测试接口介绍
使用bluetoothctl与bluetoothd服务进行交互

首先确保bluetoothd服务正常运行，输入bluetoothctl进入命令行：
```
[bluetooth]# power on
[bluetooth]# Changing power on succeeded
[bluetooth]# scan on
[bluetooth]# SetDiscoveryFilter success
[bluetooth]# Discovery started
[bluetooth]# [CHG] Controller 5C:8A:AE:67:62:04 Discovering: yes
[bluetooth]# [NEW] Device 45:DC:1E:BC:2C:77 45-DC-1E-BC-2C-77
[bluetooth]# [NEW] Device 4C:30:B8:02:7F:7A 4C-30-B8-02-7F-7A
[bluetooth]# [NEW] Device DC:28:67:9A:70:8E DC-28-67-9A-70-8E
[bluetooth]# [NEW] Device 58:FB:F1:17:D4:19 58-FB-F1-17-D4-19
[bluetooth]# [NEW] Device 84:7B:57:FB:20:8D 84-7B-57-FB-20-8D
[bluetooth]# [CHG] Device 84:7B:57:FB:20:8D TxPower: 0x000c (12)
[bluetooth]# [CHG] Device 84:7B:57:FB:20:8D Name: LT-ZHENGHONG
[bluetooth]# [CHG] Device 84:7B:57:FB:20:8D Alias: LT-ZHENGHONG
[bluetooth]# [CHG] Device 84:7B:57:FB:20:8D UUIDs: 0000110c-0000-1000-8000-00805f9b34fb
[bluetooth]# [CHG] Device 84:7B:57:FB:20:8D UUIDs: 0000110a-0000-1000-8000-00805f9b34fb
[bluetooth]# [CHG] Device 84:7B:57:FB:20:8D UUIDs: 0000110e-0000-1000-8000-00805f9b34fb
[bluetooth]# [CHG] Device 84:7B:57:FB:20:8D UUIDs: 0000110b-0000-1000-8000-00805f9b34fb
[bluetooth]# [CHG] Device 84:7B:57:FB:20:8D UUIDs: 0000111f-0000-1000-8000-00805f9b34fb
[bluetooth]# [CHG] Device 84:7B:57:FB:20:8D UUIDs: 0000111e-0000-1000-8000-00805f9b34fb
[bluetooth]#
[bluetooth]# pair 84:7B:57:FB:20:8D
Attempting to pair with 84:7B:57:FB:20:8D
[CHG] Device 84:7B:57:FB:20:8D Connected: yes
[LT-ZHENGHONG]# Request confirmation
[LT-ZHENGHONG]#   1;39m[agent] Confirm passkey 947781 (yes/no): yes
[DEL] Device 58:FB:F1:17:D4:19 58-FB-F1-17-D4-19
[bluetooth]# info 84:7B:57:FB:20:8D
Device 84:7B:57:FB:20:8D (public)
        Name: LT-ZHENGHONG
        Alias: LT-ZHENGHONG
        Class: 0x002a010c (2752780)
        Icon: computer
        Paired: no
        Bonded: no
        Trusted: no
        Blocked: no
        Connected: yes
        LegacyPairing: no
        UUID: A/V Remote Control Target (0000110c-0000-1000-8000-00805f9b34fb)
        UUID: Audio Source              (0000110a-0000-1000-8000-00805f9b34fb)
        UUID: A/V Remote Control        (0000110e-0000-1000-8000-00805f9b34fb)
        UUID: Audio Sink                (0000110b-0000-1000-8000-00805f9b34fb)
        UUID: Handsfree Audio Gateway   (0000111f-0000-1000-8000-00805f9b34fb)
        UUID: Handsfree                 (0000111e-0000-1000-8000-00805f9b34fb)
        RSSI: 0xffffffae (-82)
        TxPower: 0x000c (12)
[LT-ZHENGHONG]# [DEL] Device DC:28:67:9A:70:8E DC-28-67-9A-70-8E
[LT-ZHENGHONG]# [DEL] Device 45:DC:1E:BC:2C:77 45-DC-1E-BC-2C-77
[LT-ZHENGHONG]# [DEL] Device 53:84:3E:02:79:84 53-84-3E-02-79-84
[LT-ZHENGHONG]# [CHG] Device 84:7B:57:FB:20:8D Bonded: yes
[LT-ZHENGHONG]# info 84:7B:57:FB:20:8D
Device 84:7B:57:FB:20:8D (public)
        Name: LT-ZHENGHONG
        Alias: LT-ZHENGHONG
        Class: 0x002a010c (2752780)
        Icon: computer
        Paired: no
        Bonded: yes
        Trusted: no
        Blocked: no
        Connected: yes
        LegacyPairing: no
        UUID: A/V Remote Control Target (0000110c-0000-1000-8000-00805f9b34fb)
        UUID: Audio Source              (0000110a-0000-1000-8000-00805f9b34fb)
        UUID: A/V Remote Control        (0000110e-0000-1000-8000-00805f9b34fb)
        UUID: Audio Sink                (0000110b-0000-1000-8000-00805f9b34fb)
        UUID: Handsfree Audio Gateway   (0000111f-0000-1000-8000-00805f9b34fb)
        UUID: Handsfree                 (0000111e-0000-1000-8000-00805f9b34fb)
        RSSI: 0xffffffae (-82)
        TxPower: 0x000c (12)

```

### API介绍
平台部分将BT上下电封装在rfkill子系统中，可以直接使用rfkill操作蓝牙的供电。
```
# rfkill list
0: spacemit-bt: Bluetooth
        Soft blocked: no
        Hard blocked: no
1: phy0: Wireless LAN
        Soft blocked: no
        Hard blocked: no
2: hci0: Bluetooth
        Soft blocked: no
        Hard blocked: no

# rfkill block blutooth
# rfkill list
0: spacemit-bt: Bluetooth
        Soft blocked: yes
        Hard blocked: no
1: phy0: Wireless LAN
        Soft blocked: no
        Hard blocked: no
2: hci0: Bluetooth
        Soft blocked: yes
        Hard blocked: no
```
其中spacemit-bt是平台注册的rfkill设备，hci0是蓝牙协议栈注册的rfkill设备。

平台初始化时只需要主动打开spacemit-bt对应的蓝牙设备即可。
```
cat /sys/class/rfkill/rfkill0/type
bluetooth
cat /sys/class/rfkill/rfkill0/name
spacemit-bt
echo 1 > /sys/class/rfkill/rfkill0/state
```
操作rfkill时需要确认type和name是否为spacemit-bt的蓝牙设备。

### Debug介绍
#### sysfs
sysfs下可以查询对应rfkill的状态信息。
```
cat /sys/class/rfkill/rfkill0/state
1
```
sysfs下可以查询对应uart的信息。
```
/sys/devices/platform/soc/d4017100.uart
```
#### debugfs
debugfs下可以查询蓝牙协议栈相关组件的信息
```
/sys/kernel/debug/bluetooth# ls
hci0  l2cap  rfcomm  rfcomm_dlc  sco
```
## FAQ
问题1

现象：hciattach初始化失败。

打印：
```
Realtek Bluetooth :Realtek Bluetooth init uart with init speed:115200, type:HCI UART H5
Realtek Bluetooth :Realtek hciattach version 3.1.4796cb2.20230921-183414
Realtek Bluetooth :Use epoll
Realtek Bluetooth WARN: Writev partially, ret 0
Realtek Bluetooth WARN: OP_H5_SYNC Transmission timeout
Realtek Bluetooth WARN: Writev partial, 0
Realtek Bluetooth WARN: OP_H5_SYNC Transmission timeout
Realtek Bluetooth WARN: Writev partial, 0
Realtek Bluetooth WARN: OP_H5_SYNC Transmission timeout
Realtek Bluetooth WARN: Writev partial, 0
```
解决办法
1. 确认蓝牙的供电是否正常。
