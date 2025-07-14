# USB

介绍 USB 的功能和使用方法。

## 模块介绍

USB 全称 Universal Serial Bus（通用串行总线），是一种新兴的并逐渐取代其他接口标准的数据通信方式，由 Intel、Compaq、Digital、IBM、Microsoft、NEC 及 Northern Telecom 等计算机公司和通信公司于1995年联合制定，并逐渐形成了行业标准。

K1 共有三个USB控制器，分别为:
- USB2.0 OTG（USB0）
- USB2.0 Host（USB1）
- USB3.0 DRD（其中 USB2.0 端口为 USB2，SuperSpeed 端口为 USB3）

Linux 中，支持 两种 USB 角色：
- 可以外接 USB 外设的 Host 模式 
- 作为 USB 外设可以接入到其他上位机的 Device 模式

### 功能介绍

#### USB Host

![](static/USB-host.png)

USB Host 角色驱动框架图可以分为以下几个层次：  

- **USB Host Controller Driver：** 这是USB控制器驱动层，负责初始化控制器以及进行底层数据收发操作。  
- **USB Core Services：** 这是核心层，负责抽象出 USB 层次和基于 URB 的传输，并提供接口供上下使用。  
- **USB Class Driver：** 这是 USB 设备功能层，负责实现 USB 设备驱动、USB 功能驱动，对接内核其他框架（如 HID、UVC、Storage 等）。  

#### USB Device

![](static/USB-device.png)

USB Device 角色驱动框架图可以分为以下几个层次：  

- **USB Device Controller Driver：** 这是 USB Device 角色控制器驱动层，负责初始化控制器及进行底层数据收发操作。  
- **UDC Core：** 这是核心层，负责抽象出 USB Device 层次和基于 usb_request 的传输，并提供接口供上下使用。  
- **Composite：** 用于组合多个 USB Device 功能为一个设备，支持用户空间通过 configfs 配置，或者 legacy 驱动硬编码组合好的 Functions。  
- **Function Driver：** 这是 USB Device 功能层，负责实现 USB Device 模式的功能驱动，对接内核其他框架（如存储、V4L2、网络等）。  

这些层次结构共同构成了 Linux 系统中 USB 子系统的框架，确保了 USB 模块系统中的正常运行和数据传输。

### 源码结构介绍

USB2.0 OTG 控制器驱动代码位于 `drivers/usb` 目录下：

```
drivers/usb
|-- phy/
|   |-- phy-k1x-ci-otg.c      # OTG 驱动，用于实现 EHCI Host 和 K1X UDC 两种模式驱动切换。
|   |-- phy/phy-k1x-ci-usb2.c # PHY 驱动。
|-- host/
|   |-- ehci-k1x-ci.c         # EHCI Host 模式平台驱动, 需要和 EHCI Host 驱动组合使用。
|-- gadget/
    |-- udc/
        |-- k1x_udc_core.c    # Device 模式驱动。
```

USB2.0 HOST 控制器驱动代码位于 `drivers/usb` 目录下：

```
drivers/usb
|-- phy/
|    |-- phy-k1x-ci-usb2.c # PHY驱动。
|-- host/
    |-- ehci-k1x-ci.c     # EHCI Host 模式平台驱动, 需要和 EHCI Host 驱动组合使用。
```

USB3.0 DRD 控制器驱动代码位于 `drivers/usb` 目录下：

```
drivers/usb
|-- phy/
|   |-- phy-k1x-ci-usb2.c   # USB3.0复合端口下的USB2.0 PHY驱动。
|-- phy/
|   |-- spacemit/
|       |-- phy-spacemit-k1x-combphy.c # USB3.0 5Gbps PHY驱动。
|-- dwc3/
|   |-- dwc3-spacemit.c    # DWC平台驱动, 需要和DWC3驱动搭配使用。
```

其他一些组件代码路径如下：

```
drivers/
|-- extcon/
|    |-- extcon-k1xci.c   # MicroUSB Pin检测连接器驱动,需搭配OTG驱动、Extcon驱动使用。
|-- usb
|    |-- misc/
|        |-- spacemit_onboard_hub.c # 用于板载USB外设供电配置的帮助驱动。
```

## 关键特性

### USB2.0 OTG

#### 特性

| 特性 | 特性说明 |
| :-----| :----|
| 支持OTG | 支持 Host 和 Device 模式切换，并支持 idpin+vbuspin 检测。 |
| 支持HS,FS Host/Device | High Speed(480Mb/s), Full Speed(12Mb/s) Host/Device 模式 |
| 支持LS Host Only | 支持Low Speed(1.5Mb/s) Host only 模式|
| 支持16 Host Channel|最多支持16 Channel同时传输|
| 支持16 IN + 16 OUT Device端点| 16KB Tx Buffer, 2KB Rx Buffer|
| 支持Remote Wakeup| Host 模式下支持 High Speed, Full Speed, Low Speed Remote Wakeup |

#### 性能参数

| 测试项目 | Tx(MB/s) | Rx(MB/s) |
| :-----| :----| :----: |
| U盘测速(HIKISEMI S560 256GB) | 32.2 | 32.4 |
| U盘模式 Gadget 测速 | 21.8 | 14.8 |

测试方法

```
# U盘测速：
## host:
fio -name=Tx -ioengine=libaio -direct=1 -iodepth=64 -rw=write -bs=512K -size=1024M -numjobs=1 -group_reporting -filename=/dev/sda
fio -name=Rx -ioengine=libaio -direct=1 -iodepth=64 -rw=read -bs=512K -size=1024M -numjobs=1 -group_reporting -filename=/dev/sda

# U盘模式Gadget：
## device:
gadget-setup msc
## pc:
fio -name=DevRx -ioengine=libaio -direct=1 -iodepth=64 -rw=write -bs=512K -size=100M -numjobs=1 -group_reporting -filename=/dev/sda
fio -name=DevTx -ioengine=libaio -direct=1 -iodepth=64 -rw=read -bs=512K -size=100M -numjobs=1 -group_reporting -filename=/dev/sda
```

### USB2.0 Host

#### 特性

| 特性 | 特性说明 |
| :-----| :----|
| 支持 HS,FS,LS Host | High Speed(480Mb/s), Full Speed(12Mb/s), Low Speed(1.5Mb/s) Host模式 |
| 支持 16 Host Channel|最多支持 16 Channel 同时传输|
| 支持 Remote Wakeup| Host 模式下支持 HighSpeed, FullSpeed, LowSpeed Remote Wakeup |

#### 性能参数

| 测试项目 | Tx(MB/s) | Rx(MB/s) |
| :-----| :----| :----: |
| U 盘测速(HIKISEMI S560 256GB) | 32.2 | 32.4 |

测试方法

```
# U盘测速：
fio -name=Tx -ioengine=libaio -direct=1 -iodepth=64 -rw=write -bs=512K -size=1024M -numjobs=1 -group_reporting -filename=/dev/sda
fio -name=Rx -ioengine=libaio -direct=1 -iodepth=64 -rw=read -bs=512K -size=1024M -numjobs=1 -group_reporting -filename=/dev/sda
```

### USB3.0 DRD

#### 特性

| 特性 | 特性说明 |
| :-----| :----|
| 支持 OTG | 支持 Host 和 Device 模式切换 |
| 支持 SS Host/Device | Super Speed(5Gbps/s) Host/Device 模式 |
| 兼容 HS,FS Host/Device | High Speed(480Mb/s), Full Speed(12Mb/s) Host/Device 模式 |
| 支持 LS Host Only | 支持 Low Speed(1.5Mb/s) Host only 模式|
| 支持 32 Device 端点| 支持动态分配|
| 支持低功耗 | USB2.0 Suspend, USB3.0 U1, U2, U3|
| 支持 Remote Wakeup| Host 模式下支持 SuperSpeed, HighSpeed, FullSpeed, LowSpeed Remote Wakeup |

#### 性能参数

| 测试项目 | Tx(MB/s) | Rx(MB/s) |
| :-----| :----| :----: |
| U盘测速(HIKISEMI S560 256GB)(SuperSpeed) | 345 | 343 |
| U盘测速(HIKISEMI X301 64GB)(HighSpeed) | 27.1 | 30.2 |
| U盘模式Gadget测速(SuperSpeed) | 349 | 328 |

测试方法

```
# U盘测速:
fio -name=Tx -ioengine=libaio -direct=1 -iodepth=64 -rw=write -bs=512K -size=1024M -numjobs=1 -group_reporting -filename=/dev/sda
fio -name=Rx -ioengine=libaio -direct=1 -iodepth=64 -rw=read -bs=512K -size=1024M -numjobs=1 -group_reporting -filename=/dev/sda

# U盘模式Gadget测速(SuperSpeed):
## device:
USB_UDC=c0a00000.dwc3 gadget-setup uas:/dev/nvme0n1p1
## pc:
fio -name=DevRx -rw=write -bs=512k -size=5G -numjobs=1 -iodepth=32 -group_reporting -direct=1 -ioengine=libaio -filename=/dev/sda
fio -name=DevTx -rw=read -bs=512k -size=5G -numjobs=1 -iodepth=32 -group_reporting -direct=1 -ioengine=libaio -filename=/dev/sda
```

## 配置介绍

主要包括 **驱动使能配置** 和 **DTS 配置**

### USB2.0 OTG 配置介绍

#### CONFIG 配置

`CONFIG_K1XCI_USB2_PHY` 为 USB2.0 OTG 的 PHY 提供支持，默认 `Y`。

```
Device Drivers
         -> USB support (USB_SUPPORT [=y])
           -> USB Physical Layer drivers 
             -> K1x ci USB 2.0 PHY Driver (K1XCI_USB2_PHY [=y])
```

`CONFIG_USB_K1X_UDC` 为 USB2.0 OTG 的 Device 功能提供支持，默认 `Y`。

```
Device Drivers
         -> USB support (USB_SUPPORT [=y])
           -> USB Gadget Support (USB_GADGET [=y])
             -> USB Peripheral Controller
               -> Spacemit K1X USB2.0 Device Controller (USB_K1X_UDC [=y]) 
```

`CONFIG_USB_EHCI_K1X` 为 USB2.0 OTG 的 Host 功能提供支持，默认 `Y`。

```
Device Drivers 
         -> USB support (USB_SUPPORT [=y])
           -> EHCI HCD (USB 2.0) support (USB_EHCI_HCD [=y])
             -> EHCI support for Spacemit k1x USB controller (USB_EHCI_K1X [=y])
```

`CONFIG_USB_K1XCI_OTG` 为 USB2.0 OTG 的 OTG 角色切换提供支持，默认 `Y`。

```
Device Drivers 
         -> USB support (USB_SUPPORT [=y])
           -> USB Physical Layer drivers 
             -> Spacemit K1-x USB OTG support (USB_K1XCI_OTG [=y])    
```

`CONFIG_EXTCON_USB_K1XCI` 为 USB2.0 OTG 用于 MicroUSB 接口的 ID Pin+Vbus Pin 检测连接器驱动提供支持，默认 `Y`。

```
Device Drivers
         -> External Connector Class (extcon) support (EXTCON [=y])
           -> Spacemit K1-x USB extcon support (EXTCON_USB_K1XCI [=y])
```

#### DTS 配置

USB2.0 OTG 支持 4 种配置模式：
- 通常情况下配置为 **以 Device Only 模式工作**。
- 如果支持手动切换 Host，推荐配置为 **以 OTG 模式工作(基于 usb-role-switch)** 并且配置为默认 Device 角色。
- 如果支持自动切换双角色（如 Type-C OTG 接口），推荐配置为 **以 OTG 模式工作(基于 usb-role-switch)**，并接入 Type-C 驱动或者 GPIO 检测。
- 如果基于 K1 平台，且使用了 EXTCON 框架进行 USB 角色控制，也支持配置为 **以 OTG 模式工作（基于 K1 EXTCON）**，此方式通常依赖外部事件（如 ID 检测或 VBUS 状态）来完成角色切换。

##### 以 Device Only 模式工作

USB2.0 OTG 控制器 device 模式对应的设备树节点为 `udc`，作为 device 模式工作时，需要配置 DTS

1. disable `ehci`节点，`otg`节点。
2. enable `usbphy`节点。
3. udc节点的 `spacemit,udc-mode` 属性为 `MV_USB_MODE_UDC` 来选择 device 模式。

方案 DTS 配置如下：

```c
&usbphy {
        status = "okay";
};
&udc {
        spacemit,udc-mode = <MV_USB_MODE_UDC>;
        status = "okay";
};
&ehci { 
        status = "disabled";
};
&otg {
        status = "disabled";
};
```

##### 以 Host Only 模式工作

USB2.0 OTG 控制器 host 模式对应的设备树节点为 `ehci`，作为 host 模式工作时，可以通过 dts 配置:

1. disable `udc` 节点，`otg`节点。
2. `ehci` 节点的 `spacemit,udc-mode` 属性为 `MV_USB_MODE_HOST`（默认值）来选择 host 模式。
3. 如果 host 需要适用 GPIO 控制 vbus 开关，可以使用 `spacemit_onboard_hub` 驱动配置。
4. 可选属性 `spacemit,reset-on-resume`，用于控制系统休眠唤醒后是否 reset 控制器。

```c
&usbphy {
        status = "okay";
};
&udc { 
        status = "disabled";
};
&ehci {
        spacemit,reset-on-resume;
        spacemit,udc-mode = <MV_USB_MODE_HOST>;
        status = "okay";
};
&otg {
        status = "disabled";
};
```

##### 以 OTG 模式工作(基于 usb-role-switch)

此配置模式适合大部分方案，可接入 Type-C 角色检测、GPIO 角色检测、支持用户手动切换等。

需要为 `otg` 节点配置 `usb-role-switch` 属性，以启用对 role-switch 的支持，通常适用于 Type-C 连接器，也支持其他如 GPIO 检测，具体接入方法可参考 Linux 内核文档 usb-connector、Type-C 相关章节。配置后，`/sys/class/usb_role/` 下会出现一个 `mv-otg-role-switch` 节点。

通过启用 `otg` 节点，并且配置 `otg` 节点的 `role-switch-user-control` 属性。

`otg` 节点支持配置 `vbus-gpios` 用于控制角色切换时的 vbus。

`otg` 节点的 `role-switch-default-mode` 属性决定开机后的默认角色，可选 `host`，`peripheral`。

`otg` 节点的 `role-switch-user-control` 属性决定用户是否可以通过 sysfs 的 `/sys/class/usb_role/mv-otg-role-switch/role` 手动控制角色切换。

```c
&usbphy {
        status = "okay";
};

&otg {
        usb-role-switch;
        role-switch-user-control;
        spacemit,reset-on-resume;
        role-switch-default-mode = "host";
        vbus-gpios = <&gpio 123 0>;
        status = "okay";
        /* 可选
        typec_connector {
             ....
        }
        */
};

&udc {
        spacemit,udc-mode = <MV_USB_MODE_OTG>;
        status = "okay";
};

&ehci {
        spacemit,udc-mode = <MV_USB_MODE_OTG>;
        status = "okay";
};

```

##### 以 OTG 模式工作(基于 K1 EXTCON)

此配置只适用于 MicroUSB 接口，且需要支持 VBUS PIN、ID PIN 检测 OTG 自动切换角色的方案。

以 OTG (基于 K1 EXTCON) 模式工作，硬件方案需要进行如下设计：  

1. USB_ID0 Pin（INPUT） 接入 OTG MicroUSB ID Pin。（ID 接地 USB2.0 OTG 作为 host 工作,  ID 悬空/高 USB2.0 OTG 作为 device 工作）。
2. VBUS_ON0 Pin （INPUT）接入 OTG MicroUSB VBUS Pin，当 VBUS 有对外输出或外部输入时，VBUS_ON0 为高。
3. 需要选择一个 Pin 配置为 VBUS 开关（可选 GPIO63 或 GPIO127）配置为 drive_vbus0_iso 功能，用于驱动根据是否处于 host 模式下开关的对外 5V 供电开关。
4. 在 drive_vbus0_iso 输出高以前，VBUS_ON0 不能为高，MicroUSB 也不能对外供电，防止造成硬件损坏。
5. USB2.0 OTG Port 切换为 device 模式下时，端口接入外部 vbus 供电后，VBUS_ON0 需被拉高。

DTS 需要进行下面的配置：

1. 使用 pinctrl 把 GPIO64(另可选GPIO125)配置为 VBUS_ON0 功能，把 GPIO65(另可选GPIO126)配置为USB_ID0功能，用于检测 otg 接口状态。
2. 使能 `usbphy`、`extcon`、`otg`、`udc`、`ehci` 节点。
3. 把 DTS 中 `udc` 节点、`ehci` 节点、`otg` 节点的 `spacemit,udc-mode` 属性配置为 `MV_USB_MODE_OTG`。
4. 在 DTS 中需要通过 `otg` 节点和 `udc` 节点的 `spacemit,extern-attr` 配置 vbus 和 idpin 的检测支持，配置为 `MV_USB_HAS_VBUS_IDPIN_DETECTION`。

OTG 节点方案 DTS 配置示例如下（假设使用 pinctrl 配置采用 `k1-x_pinctrl.dtsi` 中的 `pinctrl_usb0_1` 节点），参考 `k1-x_evb.dts`：

```c
&pinctrl{
   pinctrl_usb0_1: usb0_1_grp {
       pinctrl-single,pins =<
               K1X_PADCONF(GPIO_64, MUX_MODE1, (EDGE_NONE | PULL_DOWN | PAD_1V8_DS2)) /* vbus_on0 */
               K1X_PADCONF(GPIO_65, MUX_MODE1, (EDGE_NONE | PULL_UP   | PAD_1V8_DS2)) /* usb_id0 */
               K1X_PADCONF(GPIO_63, MUX_MODE1, (EDGE_NONE | PULL_DOWN | PAD_1V8_DS2)) /* drive_vbus0_iso */ >;
   };
};
&extcon {
        status = "okay";
};
&otg {
        spacemit,udc-mode = <MV_USB_MODE_OTG>;
        spacemit,extern-attr = <MV_USB_HAS_VBUS_IDPIN_DETECTION>;
        pinctrl-names = "default";
        pinctrl-0 = <&pinctrl_usb0_1>;
        status = "okay";
};
&usbphy {
        status = "okay";
};
&udc {
        spacemit,udc-mode = <MV_USB_MODE_OTG>;
        spacemit,extern-attr = <MV_USB_HAS_VBUS_IDPIN_DETECTION>;
        status = "okay";
};
&ehci {
        spacemit,udc-mode = <MV_USB_MODE_OTG>;
        status = "okay";
};
```

##### USB 休眠唤醒

K1 USB 支持 两种 系统休眠策略：
- reset-resume 策略，保持USB最低功耗
- no-reset 策略

USB2.0 OTG 需要在 `otg` 节点和 `ehci` 节点配置 `spacemit,reset-on-resume` 属性使能 reset-resume。

如果需要支持 USB Remote Wakeup：  
- 需要对 `ehci` 节点，`otg` 节点禁用 `spacemit,reset-on-resume` 属性
- 并且启用 `wakeup-source` 属性
- 此外系统 PMU 需要使能 USB 唤醒的唤醒源，参考下文章节。

```c
&otg {
        /*spacemit,reset-on-resume;*/
        wakeup-source;
        .... 其他参数省略，请参照上面的配置
};
&ehci {
        /*spacemit,reset-on-resume;*/
        wakeup-source;
        .... 其他参数省略，请参照上面的配置
};
```

### USB2.0 HOST 配置介绍

#### CONFIG 配置

`CONFIG_K1XCI_USB2_PHY` 为 USB2.0 HOST 的 PHY 提供支持，默认 `Y`。

```
Device Drivers
         -> USB support (USB_SUPPORT [=y])
           -> USB Physical Layer drivers 
             -> K1x ci USB 2.0 PHY Driver (K1XCI_USB2_PHY [=y])
```

`CONFIG_USB_EHCI_K1X` 为 USB2.0 HOST 的 Host 功能提供支持，默认 `Y`。

```
Device Drivers 
         -> USB support (USB_SUPPORT [=y])
           -> EHCI HCD (USB 2.0) support (USB_EHCI_HCD [=y])
             -> EHCI support for Spacemit k1x USB controller (USB_EHCI_K1X [=y])
```

#### DTS 配置

##### 以 Host Only 模式工作

USB2.0 HOST支持配置为 **以Host Only模式工作**。

USB2.0 HOST 控制器 host 模式对应的设备树节点为 `ehci1`，作为 host 模式工作时，可以通过 DTS 配置:

1. `ehci1` 节点的 `spacemit,udc-mode` 属性为 `MV_USB_MODE_HOST`（默认值）来选择 host 模式。
2. 如果 host 需要适用 GPIO 控制 vbus 开关，可以使用 `spacemit_onboard_hub` 驱动配置。
3. 可选属性 `spacemit,reset-on-resume`，用于控制系统休眠唤醒后是否 reset 控制器。

```c
&usbphy1 {
        status = "okay";
};
&ehci1 {
        spacemit,reset-on-resume;
        spacemit,udc-mode = <MV_USB_MODE_HOST>;
        status = "okay";
};
```

##### USB 休眠唤醒

K1 USB支持两种系统休眠策略：
- reset-resume 策略，保持USB最低功耗
- no-reset策略

USB2.0 HOST 控制器需要在 `ehci1` 节点配置 `spacemit,reset-on-resume` 属性使能 reset-resume。

如果需要支持USB Remote Wakeup：  
- 需要对 ehci1 节点禁用 `spacemit,reset-on-resume` 属性
- 并且启用 `wakeup-source` 属性
- 此外系统 PMU 需要使能 USB 唤醒的唤醒源，见下文章节。

```c
&ehci1 {
        /*spacemit,reset-on-resume;*/
        wakeup-source;
        .... 其他参数省略，请参照上面的配置
};
```

### USB3.0 DRD 配置介绍

#### CONFIG 配置

`CONFIG_K1XCI_USB2_PHY` 为 USB3.0 DRD 的 USB2.0 Port 提供 PHY 支持，默认 `Y`。

```
Device Drivers
         -> USB support (USB_SUPPORT [=y])
           -> USB Physical Layer drivers 
             -> K1x ci USB 2.0 PHY Driver (K1XCI_USB2_PHY [=y])
```

`CONFIG_PHY_SPACEMIT_K1X_COMBPHY` 为 USB3.0 DRD 的 SuperSpeed PHY 提供支持，默认 `Y`。

```
Device Drivers 
         -> PHY Subsystem
           -> Spacemit K1-x USB3&PCIE combo PHY driver (PHY_SPACEMIT_K1X_COMBPHY [=y]) 
```

`CONFIG_USB_DWC3_SPACEMIT` 为 SpacemiT USB3.0 DRD 控制器驱动提供平台支持，默认情况下，此选型为 `Y`

```
Device Drivers
         -> USB support (USB_SUPPORT [=y])
           -> DesignWare USB3.0 DRD Core Support (USB_DWC3 [=y])
             -> Spacemit Platforms (USB_DWC3_SPACEMIT [=y])
```

`CONFIG_USB_DWC3_DUAL_ROLE` 为 USB3.0 DRD 控制器提供双模式支持，默认情况下，此选型为 `Y`，实际角色可以由设备树配置。
也可选择配置为单 Host 模式或者单 Device 模式。

```
Device Drivers
         -> USB support (USB_SUPPORT [=y])
           -> DesignWare USB3.0 DRD Core Support (USB_DWC3 [=y]) 
            -> DWC3 Mode Selection (<choice> [=y])
             -> Dual Role mode (USB_DWC3_DUAL_ROLE [=y]) 
```

#### DTS 配置

##### 以 Host Only 模式工作

USB3.0 DRD 控制器的设备树节点为 `usbdrd3`。对应 high-speed utmi phy 节点为 `usb2phy`，对应 superspeed pipe phy 节点为 `combphy`，使用 USB3.0 DRD 控制器时需要使能这两个节点。phy 节点无参数配置。

```
&usb2phy {
        status = "okay";
};
&combphy {
        status = "okay";
};
```

USB3.0 DRD 控制器有部分参数通过 DTS 的 `usbdrd3` 节点的子节点 `dwc3` 节点配置，需要配置部分 quirk 参数如下：

```c
&usbdrd3 {
        status = "okay";
        dwc3@c0a00000 {
                dr_mode = "host";
                phy_type = "utmi";
                snps,hsphy_interface = "utmi";
                snps,dis_enblslpm_quirk;
                snps,dis_u2_susphy_quirk;
                snps,dis_u3_susphy_quirk;
                snps,dis-del-phy-power-chg-quirk;
                snps,dis-tx-ipgap-linecheck-quirk;
                snps,parkmode-disable-ss-quirk;
        };
};
```

如果 Host 需要使用 GPIO 控制 vbus 开关，可以使用 `spacemit_onboard_hub` 驱动配置。

##### 以 Device Only 模式工作

USB3.0 DRD 控制器的角色通过 `usbdrd3` 节点的子节点 `dwc3` 的 `dr_mode` 属性配置，可选 `host` 、 `peripheral` 、 `otg` 。 `dr_mode` 属性配置为 `peripheral` 则以 device only 模式工作。

##### 以 DRD 模式工作

配置 `dr_mode` 为 `otg` 模式时，DTS 节点中需要配置 `usb-role-switch` 布尔属性为真。可以通过 `role-switch-default-mode` 字符串属性配置对应的默认角色，可选值为 `host` 、 `peripheral`。

```c
&usbdrd3 {
dwc3@c0a00000 {
        dr_mode = "otg";
        usb-role-switch;
        .... 其他参数省略，请参照上面的配置
        role-switch-default-mode = "host";
};
};
```

配置后，`/sys/class/usb_role/` 下会出现一个 `c0a00000.dwc3-role-switch` 节点。目前 dwc3 驱动仅支持通过 debugfs 进行角色切换：

```c
# 查看控制器当前角色：
cat /sys/kernel/debug/usb/c0a00000.dwc3/mode
# 切换至 host 角色：
echo host > /sys/kernel/debug/usb/c0a00000.dwc3/mode
# 切换至 device 角色：
echo device > /sys/kernel/debug/usb/c0a00000.dwc3/mode
```

以上是支持手动切换控制器角色的配置说明，如果需要支持自动检测 OTG 的功能需要配置额外的检测芯片驱动，参考内核文档 extcon、typec、usb-connector 相关内容。

如果 Host 需要适用 GPIO 控制 vbus 开关，可以使用 `spacemit_onboard_hub` 驱动配置。

对于 usb3.0 device 的使用场景，建议 role-switch 上报源（如 Type-C 驱动）遵守检测到 device disconnect 时（通常为检测到 vbus 断开，Type-C 则检测到 detach）上报 `USB_ROLE_NONE` 状态，并且在设备树节点为 dwc3@c0a00000 启用 `monitor-vbus` 属性。
配置后控制器将依赖 `USB_ROLE_NONE` 状态做断开检测进行软件重置，得到更好的兼容性，基于 Type-C 上报参考内核 Type-C 文档。

基于GPIO上报的示例如下：

```c
&usbdrd3 {
dwc3@c0a00000 {
        dr_mode = "otg";
        .... 其他参数省略，请参照上面的配置
        monitor-vbus;
        usb-role-switch;
        role-switch-default-mode = "peripheral";
        connector {
                /* Report vbus connection state from MCU */
                compatible = "gpio-usb-b-connector", "usb-b-connector";
                type = "micro";
                label = "Type-C";
                vbus-gpios = <&gpio 78 GPIO_ACTIVE_HIGH>;
        };
};
};
```


##### 以 High-Speed Only 模式工作 / 与 PCIE0 共同工作
USB3.0 DRD控制器物理上有两个 Ports:
- USB2.0 Port 记为 USB2
- SuperSpeed Port 记为 USB3

且 SuperSpeed Port PHY 与 PCIE0 共用，因此启用 USB3.0 DRD 且需要 SuperSpeed 5Gbps 支持时，无法使用PCIE0；仅支持 USB2 Port(480Mbps) 和 PCIE0 共用。

对于方案设计需要拆开 USB2 硬件网络和 USB3/PCIE0 硬件网络, 可以对 DTS 做如下修改：

- 删除 usbdrd3 节点的 phys 和 phy-names 属性
- 启用 dwc3@c0a00000 节点的 maximum-speed 属性并配置为 high-speed

这样会限制 USB3.0 DRD 控制器只启用其 USB2 Port。

方案 DTS 配置示例如下：
```c
&usbdrd3 {
        status = "okay";
        ......(其他配置见上文)
        /* Do not init PIPE3 phy for PCIE0 */
        /delete-property/ phys;
        /delete-property/ phy-names;
        dwc3@c0a00000 {
                maximum-speed = "high-speed";  
                ......（其他配置见上文）
        };
};

&pcie0_rc {
        pinctrl-names = "default";
        pinctrl-0 = <&pinctrl_pcie0_2>;
        status = "okay";
};
```


##### USB休眠唤醒

K1 USB 支持两种系统休眠策略:
- reset-resume策略，保持USB最低功耗
- no-reset策略。

USB3.0 DRD 控制器需要在 `usbdrd3` 节点配置 `reset-on-resume` 属性使能 reset-resume。

如果需要支持 USB Remote Wakeup：  
- 需要对`usbdrd3`节点禁用 `reset-on-resume` 属性
- 并且启用 `wakeup-source` 属性。
- 此外系统 PMU 需要使能 USB 唤醒的唤醒源，见下文章节。

```c
&usbdrd3 {
        /*reset-on-resume;*/
        wakeup-source;
        .... 其他参数省略，请参照上面的配置
};
```

### 其他USB配置介绍

#### 其他USB CONFIG配置

`CONFIG_USB` 为 USB 总线协议提供支持，默认情况，此选项为 `Y`

```
Device Drivers
         -> USB support (USB_SUPPORT [=y])    
```

对于 U 盘、USB 网卡、USB 打印机等配置需要打开，常用的选型默认 `Y`，此处不一一列举。

`CONFIG_USB_ROLE_SWITCH` 为基于 role-switch 的模式切换提供支持（如 Type-C 接口 OTG 可能使用）:

```
Device Drivers
       -> USB support (USB_SUPPORT [=y])
           -> USB Role Switch Support (USB_ROLE_SWITCH [=y]) 
```

`CONFIG_USB_GADGET` 为 USB Device 模式提供支持，默认，此选项为 `Y`

```
Device Drivers
         -> USB support (USB_SUPPORT [=y])
           -> USB Gadget Support (USB_GADGET [=y])
```

`CONFIG_USB_GADGET` 下可选支持 Configfs 配置的 function，如 RNDIS，此处根据实际需求配置，默认常用的已打开。

```
Device Drivers
         -> USB support (USB_SUPPORT [=y])
           -> USB Gadget Support (USB_GADGET [=y])
             -> USB Gadget functions configurable through configfs (USB_CONFIGFS [=y])
               -> RNDIS (USB_CONFIGFS_RNDIS [=y])
               -> Function filesystem (FunctionFS) (USB_CONFIGFS_F_FS [=y])
               -> USB Webcam function (USB_CONFIGFS_F_UVC [=y])
               -> ....
```

`CONFIG_SPACEMIT_ONBOARD_USB_HUB` 为板载 USB 外设供电配置的帮助驱动提供支持。

```
Device Drivers 
        -> USB support (USB_SUPPORT [=y])
          -> Spacemit onboard USB hub support (SPACEMIT_ONBOARD_USB_HUB [=y])
```

#### 其他 USB DTS 配置

目前支持通过 `spacemit_onboard_hub` 驱动配置开机自动配置部分 USB 相关上电逻辑，主要用于板载 VBUS 开关、需要上电的 hub 使用。
驱动的 compatible 为 `spacemit,usb3-hub`，支持配置两组 GPIO:

- hub-gpios：用于 hub 上电。
- vbus-gpios：用于对外 vbus 供电。

支持属性：
- `hub_inter_delay_ms`: int, hub-gpios 中的 GPIO 之间的延迟。
- `vbus_inter_delay_ms`: int, vbus-gpios 中 GPIO 之间的延迟。
- `vbus_delay_ms`: int, 配置 hub 上电后多久才能打开 vbus。
- `suspend_power_on`: bool, 系统休眠时是否保留电源。需要支持 USB Remote Wakeup（如键盘鼠标唤醒），此项必须配置。
  
DTS配置示例：

```
usb2hub: usb2hub {
        compatible = "spacemit,usb3-hub";
        hub-gpios = <&gpio 74 0>;
        vbus-gpios = <&gpio 91 0 &gpio 92 0>;
        status = "okay";
};
```

### USB 休眠唤醒配置介绍

#### 供电设计
休眠需要低功耗的场景，建议休眠时关闭 USB 对外 5V VBUS 供电，对于 USB 供电支持 GPIO 控制的方案，可以参考其他 USB DTS 配置中关于 `spacemit_onboard_hub` 驱动的配置说明。  

对于以下场景，休眠时需要保留USB对外5V VBUS（或板载USB外设供电）供电：
- 支持USB Remote Wakeup如USB键盘鼠标唤醒的功能。
- 需要打开摄像头视频流进入休眠，唤醒后恢复上层应用视频流的应用场景。部分摄像头如果休眠断电不支持恢复。
- 对于存在上电后初始化较久的设备（从上电到响应枚举大于2s，如部分4G模组），需要休眠唤醒过程不出现设备断开重连接的行为，建议休眠时不要关闭电源供电。
- 对于适应设备兼容性及需要使用USB对外提供供电的其他场景。

对于以下场景，休眠时需要保持对 SOC 的 USB 模块 1.8V 供电（AVDD18_USB, AVDD18_PCIE）：
- 支持 USB Remote Wakeup 如 USB 键盘鼠标唤醒的功能。
- 未启用 `reset-on-resume`/`spacemit,reset-on-resume`的情况（见各控制器章节）。

#### CONFIG配置
需要使能 `CONFIG_PM_SLEEP`。

#### DTS配置
这里介绍如何使能系统的 USB 唤醒源，各个控制器的 DTS 配置请参考控制器相关章节。
如果需要支持 USB Remote Wakeup 如 USB 键盘鼠标从休眠唤醒系统的功能，需要为设备树下`soc->pmu->power`节点配置 `pmu_wakeup5` bool属性。

DTS示例：
```cpp
&pmu {
	power: power-controller {
		pmu_wakeup5;
	};
};
```

## 接口介绍

### API介绍

#### Host API 介绍

USB host 端接入的设备通常会接入系统其他子系统，如 U 盘存储设备接入存储子系统、USB HID 接入 INPUT 子系统等，请参阅相关的 Linux 内核 API 介绍。

如果需要开发自定义协议的 USB 外设驱动，可参考 Linux 内核 `driver-api/usb/writing_usb_driver` 进行内核态驱动开发或参考 libusb 文档进行用户态驱动开发。

#### Device API 介绍

USB Device支持通过 Configfs 配置，请参考 Linux内核文档 `usb/gadget_configfs`，部分功能需要搭配应用层服务程序使用。

此外 SpacemiT 提供了[bianbu-linux/usb-gadget工具](https://gitee.com/bianbu-linux/usb-gadget)，其中有使用 Configfs 配置 USB Device 的脚本可供使用和参考，请参阅对应页面的帮助文档。

如果需要开发自定义协议的 USB Device 模式驱动，可基于 FunctionFS 开发用户态驱动，可参考 Linux 内核文档`usb/functionfs` 和 Linux 内核源码目录 `tools/usb/ffs-aio-example` 案例。

## Debug介绍

### 通用USB Host Debug介绍

#### sysfs

查看 USB 设备信息

```
ls /sys/bus/usb/devices/
1-0:1.0  1-1.1:1.0  1-1.3      1-1.4:1.0  2-1.1      2-1.1:1.2  2-1.5:1.0  usb1
...
```

sysfs 下的 USB 路径命名如下：

```
<bus>-<port[.port[.port]]>:<config>.<interface>
```

其中对于 Device 层级的 sysfs 目录，可以查询到对应设备的一些信息，选取常用介绍如下：

```
idProduct, idVendor: USB设备的PID和VID。
product: 产品名称字符串。
speed: 如480为USB2.0 high-speed, 5000为USB3.0 SuperSpeed。
```

更多内容可参考 Linux 内核 `ABI/stable/sysfs-bus-usb`, `ABI/testing/sysfs-bus-usb` 等文档。

#### debugfs

查询 USB 的设备信息

```
cat /sys/kernel/debug/usb/devices

T:  Bus=01 Lev=00 Prnt=00 Port=00 Cnt=00 Dev#=  1 Spd=480  MxCh= 1
B:  Alloc=  0/800 us ( 0%), #Int=  0, #Iso=  0
D:  Ver= 2.00 Cls=09(hub  ) Sub=00 Prot=01 MxPS=64 #Cfgs=  1
P:  Vendor=1d6b ProdID=0002 Rev= 6.06
S:  Manufacturer=Linux 6.6.36+ ehci_hcd
S:  Product=Spacemit EHCI
S:  SerialNumber=mv-ehci1
C:* #Ifs= 1 Cfg#= 1 Atr=e0 MxPwr=  0mA
I:* If#= 0 Alt= 0 #EPs= 1 Cls=09(hub  ) Sub=00 Prot=00 Driver=hub
E:  Ad=81(I) Atr=03(Int.) MxPS=   4 Ivl=256ms
......
```

### USB2.0 OTG Debug介绍

Device 模式下的 Debug 信息：暂不支持。

Host 模式下的 Debug 信息：

```
# cd /sys/kernel/debug/usb/ehci/mv-ehci/
bandwidth: 可以查看控制器当前分配的带宽。
periodic: 可以查看当前周期性传输的Debug信息。
register: dump ehci控制器寄存器。
```

OTG 的 Debug 信息：
如果 DTS 中配置了相关属性，可以在以下节点查看到 USB2.0 OTG Port 的当前角色信息，支持手动切换角色。

```
cat /sys/class/usb_role/mv-otg-role-switch/role
device

echo host > /sys/class/usb_role/mv-otg-role-switch/role
cat /sys/class/usb_role/mv-otg-role-switch/role
host
```

### USB2.0 HOST Debug介绍

Host 模式下的 Debug 信息：

```
# cd /sys/kernel/debug/usb/ehci/mv-ehci1/
bandwidth: 可以查看控制器当前分配的带宽。
periodic: 可以查看当前周期性传输的Debug信息。
register: dump ehci控制器寄存器。
```

### USB3.0 DRD Debug介绍

Device 模式下的 Debug 信息：

```
# cd /sys/kernel/debug/usb/c0a00000.dwc3
link_state: 查看Device模式下时的链路状态。
```

Host 模式下的 Debug 信息：

```
# cd /sys/kernel/debug/usb/xhci/xhci-hcd.0.auto
# 查看USB3.0 USB2.0 Port端口信息
cat ports/port01/portsc
Powered Connected Enabled Link:U0 PortSpeed:3 Change: Wake:
# 查看USB3.0的SS Port信息
cat ports/port02/portsc
Powered Connected Enabled Link:U3 PortSpeed:4 Change: Wake: WDE WOE
```

DRD 的 Debug 信息：

```
cat /sys/kernel/debug/usb/c0a00000.dwc3/mode
device
# 手动切换数据角色(需要DTS配置dr_mode=otg)
echo host > /sys/kernel/debug/usb/c0a00000.dwc3/mode
cat /sys/kernel/debug/usb/c0a00000.dwc3/mode
host
```

### 其他 Debug 介绍

目前支持通过 `spacemit_onboard_hub` 驱动配置开机自动配置部分 USB 相关上电逻辑，也提供了部分 debug 支持：
路径在 USB 的 debugfs 目录下，名称为 `spacemit_onboard_hub` 的 DTS 路径名称, 如 `usb2hub`。

```
# cd /sys/kernel/debug/usb/usb2hub/
hub_on: hub-gpios的开关情况。可写入 0/1 控制。
vbus_on: vbus-gpios的开关情况。可写入 0/1 控制。
suspend_power_on: 控制系统休眠时是否关闭电源，由DTS配置默认值。
```

## 测试介绍

USB设备识别可以通过应用层工具 `lsusb` 查看，还可以使用 `lsub -tv` 查看树形详细信息。
```
$ lsusb
Bus 003 Device 002: ID 2109:0817 VIA Labs, Inc. USB3.0 Hub
Bus 003 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
.....
```

USB 设备描述符可以通过应用层工具 `lsusb -v` 查看。
```
$ lsusb -v -s 001:001

Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Device Descriptor:
  bLength                18
  bDescriptorType         1
  bcdUSB               2.00
  bDeviceClass            9 Hub
.....
```

针对 USB 外设可以通过第三方工具完成性能和功能测试，例如：
- USB 存储的读写测试可以使用 FIO 工具，目前 bianbu-linux 上已集成 FIO
- 鼠标键盘功能验证可以通过查看 input 子系统（可选用evtest、getevent等工具）
- 网卡功能可以使用 ping 命令、iperf3 等测试。

## FAQ
