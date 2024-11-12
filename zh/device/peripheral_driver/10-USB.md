# USB

介绍USB的功能和使用方法。

## 模块介绍

USB全称Universal Serial Bus（通用串行总线），是一种新兴的并逐渐取代其他接口标准的数据通信方式，由 Intel、Compaq、Digital、IBM、Microsoft、NEC及Northern Telecom 等计算机公司和通信公司于1995年联合制定，并逐渐形成了行业标准。

K1共有三个USB控制器，分别为USB2.0 OTG（USB0），USB2.0 Host（USB1），USB3.0 DRD（USB2.0 Port记为USB2、SuperSpeed Port记为USB3）。

Linux中，支持两种USB角色，可以外接USB外设的Host模式和作为USB外设可以接入到其他上位机的Device模式。

### 功能介绍

#### USB Host

![](static/USB-host.png)

USB Host角色驱动框架图可以分为以下几个层次：  

- USB Host Controller：这是USB控制器驱动层，负责初始化USB控制器以及底层的数据收发操作，直接控制底层寄存器。  
- USB Core：这是核心层，负责抽象出USB层次和基于URB的传输，并提供接口供上下使用。  
- USB Class：这是USB设备功能层，负责实现USB设备驱动、USB功能驱动，对接内核其他框架（如HID、UVC、Storage等）。  

#### USB Device

![](static/USB-device.png)

USB Device角色驱动框架图可以分为以下几个层次：  

- USB Device Controller：这是USB Device角色控制器驱动层，负责初始化USB控制器以及底层的数据收发操作，直接控制底层寄存器。  
- UDC Core：这是核心层，负责抽象出USB Device层次和基于usb_request的传输，并提供接口供上下使用。  
- Composite: 用于组合多个USB Device功能为一个设备，支持用户空间通过configfs配置，或者legacy驱动硬编码组合好的Functions。  
- Function：这是USB Device功能层，负责实现USB Device模式的功能驱动，对接内核其他框架（如存储、V4L2、网络等）。  

这些层次结构共同构成了Linux系统中USB子系统的框架，确保了USB模块系统中的正常运行和数据传输。

### 源码结构介绍

USB2.0 OTG控制器驱动代码在drivers/usb目录下：

```
drivers/usb
|-- phy/
|   |-- phy-k1x-ci-otg.c      # OTG驱动，用于实现EHCI Host和K1X UDC两种模式驱动切换。
|   |-- phy/phy-k1x-ci-usb2.c # PHY驱动。
|-- host/
|   |-- ehci-k1x-ci.c         # EHCI Host模式平台驱动, 需要和EHCI Host驱动组合使用。
|-- gadget/
    |-- udc/
        |-- k1x_udc_core.c    # Device模式驱动。
```

usb2.0host控制器驱动代码在drivers/usb目录下：

```
drivers/usb
|-- phy/
|    |-- phy-k1x-ci-usb2.c # PHY驱动。
|-- host/
    |-- ehci-k1x-ci.c     # EHCI Host模式平台驱动, 需要和EHCI Host驱动组合使用。
```

usb3.0drd控制器驱动代码在drivers/usb目录下：

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
    |-- extcon-k1xci.c   # ID Pin+Vbus Pin检测连接器驱动,需搭配OTG驱动使用。
|-- usb
    |-- misc/
        |-- spacemit_onboard_hub.c # 用于板载USB外设供电配置的帮助驱动。
```

## 关键特性

### USB2.0 OTG

#### 特性

| 特性 | 特性说明 |
| :-----| :----|
| 支持OTG | 支持Host和Device模式切换，并支持idpin+vbuspin检测。 |
| 支持HS,FS Host/Device | High Speed(480Mb/s), Full Speed(12Mb/s) Host/Device 模式 |
| 支持LS Host Only | 支持Low Speed(1.5Mb/s) Host only 模式|
| 支持16 Host Channel|最多支持16 Channel同时传输|
| 支持16 IN + 16 OUT Device端点| 16KB Tx Buffer, 2KB Rx Buffer|
| 支持Remote Wakeup| Host模式下支持High Speed, Full Speed, Low Speed Remote Wakeup |

#### 性能参数

| 测试项目 | Tx(MB/s) | Rx(MB/s) |
| :-----| :----| :----: |
| U盘测速(HIKISEMI S560 256GB) | 32.2 | 32.4 |
| U盘模式Gadget测速 | 21.8 | 14.8 |

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
| 支持HS,FS,LS Host | High Speed(480Mb/s), Full Speed(12Mb/s), Low Speed(1.5Mb/s) Host模式 |
| 支持16 Host Channel|最多支持16 Channel同时传输|
| 支持Remote Wakeup| Host模式下支持HighSpeed, FullSpeed, LowSpeed Remote Wakeup |

#### 性能参数

| 测试项目 | Tx(MB/s) | Rx(MB/s) |
| :-----| :----| :----: |
| U盘测速(HIKISEMI S560 256GB) | 32.2 | 32.4 |

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
| 支持OTG | 支持Host和Device模式切换 |
| 支持SS Host/Device | Super Speed(5Gbps/s) Host/Device 模式 |
| 兼容HS,FS Host/Device | High Speed(480Mb/s), Full Speed(12Mb/s) Host/Device 模式 |
| 支持LS Host Only | 支持Low Speed(1.5Mb/s) Host only 模式|
| 支持32 Device端点| 支持动态分配|
| 支持低功耗 | USB2.0 Suspend, USB3.0 U1, U2, U3|
| 支持Remote Wakeup| Host模式下支持SuperSpeed, HighSpeed, FullSpeed, LowSpeed Remote Wakeup |

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

主要包括驱动使能配置和dts配置

### USB2.0 OTG配置介绍

#### CONFIG配置

CONFIG_K1XCI_USB2_PHY为USB2.0 OTG的PHY提供支持，默认Y。

```
Device Drivers
         -> USB support (USB_SUPPORT [=y])
           -> USB Physical Layer drivers 
             -> K1x ci USB 2.0 PHY Driver (K1XCI_USB2_PHY [=y])
```

CONFIG_USB_K1X_UDC为USB2.0 OTG的Device功能提供支持，默认Y。

```
Device Drivers
         -> USB support (USB_SUPPORT [=y])
           -> USB Gadget Support (USB_GADGET [=y])
             -> USB Peripheral Controller
               -> Spacemit K1X USB2.0 Device Controller (USB_K1X_UDC [=y]) 
```

CONFIG_USB_EHCI_K1X为USB2.0 OTG的Host功能提供支持，默认Y。

```
Device Drivers 
         -> USB support (USB_SUPPORT [=y])
           -> EHCI HCD (USB 2.0) support (USB_EHCI_HCD [=y])
             -> EHCI support for Spacemit k1x USB controller (USB_EHCI_K1X [=y])
```

CONFIG_USB_K1XCI_OTG为USB2.0 OTG的OTG角色切换提供支持，默认Y。

```
Device Drivers 
         -> USB support (USB_SUPPORT [=y])
           -> USB Physical Layer drivers 
             -> Spacemit K1-x USB OTG support (USB_K1XCI_OTG [=y])    
```

CONFIG_EXTCON_USB_K1XCI为USB2.0 OTG用于MicroUSB接口的的ID Pin+Vbus Pin检测连接器驱动提供支持，默认Y。

```
Device Drivers
         -> External Connector Class (extcon) support (EXTCON [=y])
           -> Spacemit K1-x USB extcon support (EXTCON_USB_K1XCI [=y])
```

#### DTS配置

USB2.0 OTG支持4种配置模式，通常情况下配置为**以Device Only模式工作**。

如果支持手动切换Host，推荐配置为**以OTG模式工作(基于usb-role-switch)**并且配置为默认Device角色。

如果支持自动切换双角色（如Type-C OTG接口），推荐配置为**以OTG模式工作(基于usb-role-switch)**，并接入Type-C驱动或者GPIO检测。

##### 以Device Only 模式工作

USB2.0 OTG 控制器 device 模式对应的设备树节点为 udc，作为 device 模式工作时，需要配置 dts

1. disable ehci节点。
2. enable usbphy节点。
3. udc节点的 `spacemit,udc-mode` 属性为 `MV_USB_MODE_UDC` 来选择 device 模式。

方案 dts 配置如下：

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
```

##### 以Host Only模式工作

USB2.0 OTG 控制器 host 模式对应的设备树节点为 ehci，作为 host 模式工作时，可以通过 dts 配置:

1. disable udc 节点。
2. ehci节点的`spacemit,udc-mode` 属性为 `MV_USB_MODE_HOST`（默认值）来选择 host 模式。
3. 如果host需要适用GPIO控制vbus开关，可以使用spacemit_onboard_hub驱动配置。
4. 可选属性`spacemit,reset-on-resume`，用于控制系统休眠唤醒后是否reset控制器。

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
```

##### 以OTG模式工作(基于usb-role-switch)

此配置模式适合大部分方案，可接入Type—C角色检测、GPIO角色检测、支持用户手动切换等。

需要为 otg 节点配置`usb-role-switch`属性，以启用对role-switch的支持，通常适用于typec连接器，也支持其他如GPIO检测，接入方法可参考Linux内核文档usb-connector、typec相关章节。配置后，/sys/class/usb_role/下会出现一个 mv-otg-role-switch 节点。

通过启用otg节点，并且配置otg节点的`role-switch-user-control`属性。

otg节点支持配置vbus-gpios用于控制角色切换时的vbus。

otg节点的`role-switch-default-mode`属性决定开机后的默认角色，可选host，peripheral。

otg节点的`role-switch-user-control`属性决定用户是否可以通过sysfs的/sys/class/usb_role/mv-otg-role-switch/role手动控制角色切换。

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
        spacemit,udc-mode = <MV_USB_MODE_HOST>;
        status = "okay";
};

```

##### 以OTG模式工作(基于K1 EXTCON)

此配置适用于MicroUSB接口，且需要支持VBUS PIN、ID PIN唤醒的方案。

以 otg(基于K1 EXTCON)模式工作，硬件方案需要进行如下设计：  

1. USB_ID0 Pin（INPUT） 接入 OTG MicroUSB ID Pin。（ID 接地 USB2.0 OTG 作为 host 工作,  ID 悬空/高 USB2.0 OTG 作为 device 工作）。
2. VBUS_ON0 Pin （INPUT）接入 OTG MicroUSB VBUS Pin，当 VBUS 有对外输出或外部输入时，VBUS_ON0 为高。
3. 需要选择一个Pin配置为VBUS开关（可选GPIO63或GPIO127）配置为drive_vbus0_iso功能，用于驱动根据是否处于 host 模式下开关的对外 5v 供电开关。
4. 在drive_vbus0_iso输出高以前，VBUS_ON0 不能为高，MicroUSB也不能对外供电，防止造成硬件损坏。
5. USB2.0 OTG Port 切换为 device 模式下时，端口接入外部 vbus 供电后，VBUS_ON0 需被拉高。

dts 需要进行下面的配置：

1. 使用 pinctrl 把 GPIO64(另可选GPIO125)配置为 VBUS_ON0 功能，把 GPIO65(另可选GPIO126)配置为USB_ID0功能，用于检测 otg 接口状态。
2. 使能 usbphy、extcon、otg、udc、ehci 节点。
3. 把 dts 中 udc 节点、ehci 节点、otg 节点的 spacemit,udc-mode 属性配置为 MV_USB_MODE_OTG。
4. 在 dts 中需要通过 otg 节点和 udc 节点的 spacemit,extern-attr 配置 vbus 和 idpin 的检测支持，配置为 MV_USB_HAS_VBUS_IDPIN_DETECTION。
5. otg 节点需要配置 vbus-gpio，用于驱动根据进入 host 模式情形下控制对外 5v 供电开关。切换为 host 模式时，在 vbus-gpio 被驱动拉高以前，VBUS_ON0 不能为高。

otg 节点方案 dts 配置示例如下（假设使用 pinctrl 配置采用 k1-x_pinctrl.dtsi 中的 pinctrl_usb0_1 节点），参考 k1-x_evb.dts：

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

##### USB休眠唤醒

K1 USB支持两种系统休眠策略，一种是reset-resume策略，保持USB最低功耗，一种是no-reset策略。
USB2.0 OTG需要在otg节点和ehci节点配置 spacemit,reset-on-resume 属性使能reset-resume。

如果需要支持USB Remote Wakeup：  
需要对ehci节点，otg节点禁用 spacemit,reset-on-resume 属性，并且启用 wakeup-source 属性。
此外系统pmu需要使能usb唤醒的唤醒源，参考电源管理文档相关章节。

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

### USB2.0 HOST配置介绍

#### CONFIG配置

CONFIG_K1XCI_USB2_PHY为USB2.0 HOST的PHY提供支持，默认Y。

```
Device Drivers
         -> USB support (USB_SUPPORT [=y])
           -> USB Physical Layer drivers 
             -> K1x ci USB 2.0 PHY Driver (K1XCI_USB2_PHY [=y])
```

CONFIG_USB_EHCI_K1X为USB2.0 HOST的Host功能提供支持，默认Y。

```
Device Drivers 
         -> USB support (USB_SUPPORT [=y])
           -> EHCI HCD (USB 2.0) support (USB_EHCI_HCD [=y])
             -> EHCI support for Spacemit k1x USB controller (USB_EHCI_K1X [=y])
```

#### DTS配置

##### 以Host Only模式工作

USB2.0 HOST支持配置为**以Host Only模式工作**。

USB2.0 HOST 控制器 host 模式对应的设备树节点为 ehci1，作为 host 模式工作时，可以通过 dts 配置:

1. ehci1节点的`spacemit,udc-mode` 属性为 `MV_USB_MODE_HOST`（默认值）来选择 host 模式。
2. 如果host需要适用GPIO控制vbus开关，可以使用spacemit_onboard_hub驱动配置。
3. 可选属性`spacemit,reset-on-resume`，用于控制系统休眠唤醒后是否reset控制器。

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

USB2.0 HOST控制器也支持以OTG模式工作(基于usb-role-switch)，具体参考方案dts和USB2.0 OTG配置介绍的相关内容。

##### USB休眠唤醒

K1 USB支持两种系统休眠策略，一种是reset-resume策略，保持USB最低功耗，一种是no-reset策略。
USB2.0 HOST控制器需要在ehci节点配置 spacemit,reset-on-resume 属性使能reset-resume。

如果需要支持USB Remote Wakeup：  
需要对ehci1节点禁用 spacemit,reset-on-resume 属性，并且启用 wakeup-source 属性。
此外系统pmu需要使能usb唤醒的唤醒源，参考电源管理文档相关章节。

```c
&ehci1 {
        /*spacemit,reset-on-resume;*/
        wakeup-source;
        .... 其他参数省略，请参照上面的配置
};
```

### USB3.0 DRD配置介绍

#### CONFIG配置

CONFIG_K1XCI_USB2_PHY为USB3.0 DRD的USB2.0 Port提供PHY支持，默认Y。

```
Device Drivers
         -> USB support (USB_SUPPORT [=y])
           -> USB Physical Layer drivers 
             -> K1x ci USB 2.0 PHY Driver (K1XCI_USB2_PHY [=y])
```

CONFIG_PHY_SPACEMIT_K1X_COMBPHY为USB3.0 DRD的SuperSpeed PHY提供支持，默认Y。

```
Device Drivers 
         -> PHY Subsystem
           -> Spacemit K1-x USB3&PCIE combo PHY driver (PHY_SPACEMIT_K1X_COMBPHY [=y]) 
```

CONFIG_USB_DWC3_SPACEMIT 为Spacemit USB3.0 DRD控制器驱动提供平台支持，默认情况下，此选型为Y

```
Device Drivers
         -> USB support (USB_SUPPORT [=y])
           -> DesignWare USB3.0 DRD Core Support (USB_DWC3 [=y])
             -> Spacemit Platforms (USB_DWC3_SPACEMIT [=y])
```

CONFIG_USB_DWC3_DUAL_ROLE为USB3.0 DRD控制器提供双模式支持，默认情况下，此选型为Y，实际角色可以由设备树配置。
也可选择配置为单Host模式或者单Device模式。

```
Device Drivers
         -> USB support (USB_SUPPORT [=y])
           -> DesignWare USB3.0 DRD Core Support (USB_DWC3 [=y]) 
            -> DWC3 Mode Selection (<choice> [=y])
             -> Dual Role mode (USB_DWC3_DUAL_ROLE [=y]) 
```

#### DTS配置

##### 以Host Only模式工作

USB3.0 DRD控制器的设备树节点为 usbdrd3.对应 high-speed utmi phy 节点为 usb2phy，对应 superspeed pipe phy 节点为 combphy，使用 USB3.0 DRD 控制器时需要使能这两个节点。phy 节点无参数配置。

```
&usb2phy {
        status = "okay";
};
&combphy {
        status = "okay";
};
```

USB3.0 DRD控制器有部分参数通过 dts 的 usbdrd3 节点的子节点 dwc3 节点配置，需要配置部分quirk参数如下：

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

如果host需要适用GPIO控制vbus开关，可以使用spacemit_onboard_hub驱动配置。

##### 以Device Only模式工作

USB3.0 DRD 控制器的角色通过 usbdrd3 节点的子节点 dwc3 的 dr_mode 属性配置，可选 host、peripheral、otg。dr_mode 属性配置为 peripheral 则以 device only 模式工作。

##### 以DRD模式工作

配置 dr_mode 为 otg 模式时，dts 节点中需要配置 usb-role-switch 布尔属性为真。可以通过 role-switch-default-mode 字符串属性配置对应的默认角色，可选值为 host、peripheral。

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

配置后，/sys/class/usb_role/下会出现一个 c0a00000.dwc3-role-switch 节点。目前 dwc3 驱动仅支持通过 debugfs 进行角色切换：

```c
# 查看控制器当前角色：
cat /sys/kernel/debug/usb/c0a00000.dwc3/mode
# 切换至 host 角色：
echo host > /sys/kernel/debug/usb/c0a00000.dwc3/mode
# 切换至 device 角色：
echo device > /sys/kernel/debug/usb/c0a00000.dwc3/mode
```

以上是支持手动切换控制器角色的配置说明，如果需要支持自动检测 otg 的功能需要配置额外的检测芯片驱动，参考内核文档extcon、typec、usb-connector相关内容。

如果host需要适用GPIO控制vbus开关，可以使用spacemit_onboard_hub驱动配置。

##### USB休眠唤醒

K1 USB支持两种系统休眠策略，一种是reset-resume策略，保持USB最低功耗，一种是no-reset策略。
USB3.0 DRD控制器需要在usbdrd3节点配置reset-on-resume属性使能reset-resume。

如果需要支持USB Remote Wakeup：  
需要对usbdrd3节点禁用 reset-on-resume 属性，并且启用 wakeup-source 属性。
此外系统pmu需要使能usb唤醒的唤醒源，参考电源管理文档相关章节。

```c
&usbdrd3 {
        /*reset-on-resume;*/
        wakeup-source;
        .... 其他参数省略，请参照上面的配置
};
```

### 其他USB配置介绍

#### 其他USB CONFIG配置

CONFIG_USB为USB总线协议提供支持，默认情况，此选项为Y

```
Device Drivers
         -> USB support (USB_SUPPORT [=y])    
```

对于U盘、USB网卡、USB打印机等配置需要打开，常用的选型默认Y，此处不一一列举。

CONFIG_USB_ROLE_SWITCH为基于role-switch的模式切换提供支持（如Type-C接口OTG可能使用）:

```
Device Drivers
       -> USB support (USB_SUPPORT [=y])
           -> USB Role Switch Support (USB_ROLE_SWITCH [=y]) 
```

CONFIG_USB_GADGET为USB Device模式提供支持，默认，此选项为Y

```
Device Drivers
         -> USB support (USB_SUPPORT [=y])
           -> USB Gadget Support (USB_GADGET [=y])
```

CONFIG_USB_GADGET下可选支持Configfs配置的function，如RNDIS，此处根据实际需求配置，默认常用的已打开。

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

CONFIG_SPACEMIT_ONBOARD_USB_HUB为板载USB外设供电配置的帮助驱动提供支持。

```
Device Drivers 
        -> USB support (USB_SUPPORT [=y])
          -> Spacemit onboard USB hub support (SPACEMIT_ONBOARD_USB_HUB [=y])
```

#### 其他USB DTS配置

目前支持通过spacemit_onboard_hub驱动配置开机自动配置部分USB相关上电逻辑，主要用于板载VBUS开关、需要上电的hub使用。
驱动的compatible为"spacemit,usb3-hub"，支持配置两组GPIO:

- hub-gpios：用于hub上电。
- vbus-gpios：用于对外vbus供电。
支持属性：
- hub_inter_delay_ms: int, hub-gpios中的gpio之间的延迟。
- vbus_inter_delay_ms: int, vbus-gpios中gpio之间的延迟。
- vbus_delay_ms: int, 配置hub上电后多久才能打开vbus。
- suspend_power_on: bool, 系统休眠时是否保留电源。需要支持USB Remote Wakeup（如键盘鼠标唤醒），此项必须配置。
  
DTS配置示例：

```
usb2hub: usb2hub {
        compatible = "spacemit,usb3-hub";
        hub-gpios = <&gpio 74 0>;
        vbus-gpios = <&gpio 91 0 &gpio 92 0>;
        status = "okay";
};
```

### USB休眠唤醒配置介绍

1. 休眠需要低功耗的场景，建议休眠时关闭USB对外5VVBUS供电，对于USB供电支持GPIO控制的方案，可以参考spacemit_onboard_hub驱动的配置说明。  
2. 对于以下场景，建议休眠时保留USB对外5VVBUS供电，对于支持GPIO控制VBUS的方案，可以参
考6.4。
   - 需要打开摄像头视频流进入休眠，唤醒后恢复上层应用视频流的应用场景。部分摄像头如果休
眠断电不支持恢复。
   - 对于上电后初始化较久的设备（从上电到枚举大于3s），需要休眠唤醒过程不出现设备断开重
连接的行为，建议休眠时不要关闭电源供电。
   - 对于适应设备兼容性及需要使用USB对外提供供电的其他场景。
3. 关于如何启用系统对USB RemoteWakeup支持，参见电源管理的standby小节。

## 接口介绍

### API介绍

#### Host API介绍

USB host设备通常会接入系统其他子系统，如U盘存储设备接入存储子系统、USB HID接入INPUT子系统等，请参阅相关的Linux内核API介绍。

#### Device API介绍

USB Device支持通过Configfs配置，请参考Linux内核文档usb/gadget_configfs。

此外SpacemiT提供了[bianbu-linux/usb-gadget工具](https://gitee.com/bianbu-linux/usb-gadget)，其中有使用Configfs配置USB Device的脚本可供使用和参考，请参阅对应页面的帮助文档。

## Debug介绍

### 通用USB Host Debug介绍

#### sysfs

查看USB设备信息

```
ls /sys/bus/usb/devices/
1-0:1.0  1-1.1:1.0  1-1.3      1-1.4:1.0  2-1.1      2-1.1:1.2  2-1.5:1.0  usb1
...
```

sysfs 下的 usb 路径命名如下：

```
<bus>-<port[.port[.port]]>:<config>.<interface>
```

其中对于Device层级的sysfs目录，可以查询到对应设备的一些信息，选取常用介绍如下：

```
idProduct, idVendor: USB设备的PID和VID。
product: 产品名称字符串。
speed: 如480为USB2.0 high-speed, 5000为USB3.0 SuperSpeed。
```

更多内容可参考Linux内核 ABI/stable/sysfs-bus-usb, ABI/testing/sysfs-bus-usb等文档。

#### debugfs

查询USB的设备信息

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

Device模式下的Debug信息：暂不支持。

Host模式下的Debug信息：

```
# cd /sys/kernel/debug/usb/ehci/mv-ehci/
bandwidth: 可以查看控制器当前分配的带宽。
periodic: 可以查看当前周期性传输的Debug信息。
register: dump ehci控制器寄存器。
```

OTG的Debug信息：
如果DTS中配置了相关属性，可以在以下节点查看到 USB2.0 OTG Port的当前角色信息，支持手动切换角色。

```
cat /sys/class/usb_role/mv-otg-role-switch/role
device

echo host > /sys/class/usb_role/mv-otg-role-switch/role
cat /sys/class/usb_role/mv-otg-role-switch/role
host
```

### USB2.0 HOST Debug介绍

Host模式下的Debug信息：

```
# cd /sys/kernel/debug/usb/ehci/mv-ehci1/
bandwidth: 可以查看控制器当前分配的带宽。
periodic: 可以查看当前周期性传输的Debug信息。
register: dump ehci控制器寄存器。
```

### USB3.0 DRD Debug介绍

Device模式下的Debug信息：

```
# cd /sys/kernel/debug/usb/c0a00000.dwc3
link_state: 查看Device模式下时的链路状态。
```

Host模式下的Debug信息：

```
# cd /sys/kernel/debug/usb/xhci/xhci-hcd.0.auto
# 查看USB3.0 USB2.0 Port端口信息
cat ports/port01/portsc
Powered Connected Enabled Link:U0 PortSpeed:3 Change: Wake:
# 查看USB3.0的SS Port信息
cat ports/port02/portsc
Powered Connected Enabled Link:U3 PortSpeed:4 Change: Wake: WDE WOE
```

DRD的Debug信息：

```
cat /sys/kernel/debug/usb/c0a00000.dwc3/mode
device
# 手动切换数据角色(需要DTS配置dr_mode=otg)
echo host > /sys/kernel/debug/usb/c0a00000.dwc3/mode
cat /sys/kernel/debug/usb/c0a00000.dwc3/mode
host
```

### 其他 Debug 介绍

目前支持通过spacemit_onboard_hub驱动配置开机自动配置部分USB相关上电逻辑，也提供了部分debug支持：
路径在usb的debugfs目录下，名称为spacemit_onboard_hub的dts路径名称如usb2hub。

```
# cd /sys/kernel/debug/usb/usb2hub/
hub_on: R/W，hub-gpios的开关情况。可写入 0/1 控制。
vbus_on: R/W, vbus-gpios的开关情况。可写入 0/1 控制。
suspend_power_on: R/W, 控制系统休眠时是否关闭电源，由DTS配置默认值。
```

## 测试介绍

USB可以通过第三方工具完成性能和功能测试，eg：fio用于USB存储测试，目前bianbu-linux上已集成fio工具。
鼠标键盘功能可以通过查看input子系统，网卡功能可以使用iperf3等。

## FAQ
