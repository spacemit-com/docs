介绍USB的功能和使用方法。

K1 共有三个 usb 控制器，分别为 usb2.0otg，usb2.0host，usb3.0drd。

## usb2.0otg

usb2.0otg 控制器支持 device 模式、host 模式和 otg 模式。usb2.0otg 控制器的 phy 为 usbphy 节点，使用控制器的任一模式都需要使能 usbphy 节点。

### 以 device only 模式工作

usb2.0otg 控制器 device 模式对应的设备树节点为 udc，作为 device 模式工作时，需要配置 dts 节点的 spacemit,udc-mode 属性为 MV_USB_MODE_UDC 来选择 device 模式。

方案 dts 配置如下：

```c
&usbphy {
        status = "okay";
};
&udc {
        spacemit,udc-mode = <MV_USB_MODE_UDC>;
        status = "okay";
};
```

### 以 host only 模式工作

usb2.0otg 控制器 host 模式对应的设备树节点为 ehci，作为 host 模式工作时，可以通过 dts 配置 spacemit,udc-mode 属性为 MV_USB_MODE_HOST（默认值）来选择 host 模式。需要 disable udc 节点。

如果 host only 控制器需要控制 vbus 开关，根据需要配置对应的 pinctrl 属性子节点（参考 [PINCTRL](PINCTRL)）。

```c
&usbphy {
        status = "okay";
};
&ehci {
        spacemit,udc-mode = <MV_USB_MODE_HOST>;
        status = "okay";
};
```

### 以 otg 模式工作

以 otg 模式工作，方案需要进行如下设计：

1. USB_ID0 Pin（INPUT） 接入 OTG ID Pin，ID 接地 usb2.0otg 作为 host 工作,  ID 悬空/高 usb2.0otg 作为 device 工作。
2. VBUS_ON0 Pin （INPUT）接入 OTG 接口上 VBUS，当 VBUS 有对外输出或外部输入时，VBUS_ON0 为高。
3. 需要预留一个 OUTPUT GPIO（建议使用 GPIO63）作为 vbus-gpio 用于驱动根据是否处于 host 模式下开关的对外 5v 供电开关。
4. usb2.0otg 切换为 host 模式下时，在 vbus-gpio 输出高以前，VBUS_ON0 不能为高。
5. usb2.0otg 切换为 device 模式下时，端口接入外部 vbus 供电后，VBUS_ON0 被拉高。

dts 需要进行下面的配置：

1. 使用 pinctrl 把 GPIO64 配置为 VBUS_ON0 功能，把 GPIO65 配置为 USB_ID0 功能，用于检测 otg 接口状态。
2. 使能 usbphy、extcon、otg、udc、ehci 节点。
3. 把 dts 中 udc 节点、ehci 节点、otg 节点的 spacemit,udc-mode 属性配置为 MV_USB_MODE_OTG。
4. 在 dts 中需要通过 otg 节点和 udc 节点的 spacemit,extern-attr 配置 vbus 和 idpin 的检测支持，配置为 MV_USB_HAS_VBUS_IDPIN_DETECTION。
5. otg 节点需要配置 vbus-gpio，用于驱动根据进入 host 模式情形下控制对外 5v 供电开关。切换为 host 模式时，在 vbus-gpio 被驱动拉高以前，VBUS_ON0 不能为高。

otg 节点方案 dts 配置示例如下（假设使用 pinctrl 配置采用 k1-x_pinctrl.dtsi 中的 pinctrl_usb0_1 节点），参考 k1-x_evb.dts：

```c
&extcon {
        status = "okay";
};
&otg {
        spacemit,udc-mode = <MV_USB_MODE_OTG>;
        spacemit,extern-attr = <MV_USB_HAS_VBUS_IDPIN_DETECTION>;
        pinctrl-names = "default";
        pinctrl-0 = <&pinctrl_usb0_1>;
        vbus-gpios = <&gpio 63 0>;
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

otg 模式的识别的切换流程如下：

1. 默认情况无连接状态，USB_ID0 为悬空状态，VBUS_ON0 为低，对应 VBUS 无输入无输出。
2. 默认状态下，VBUS_ON0 突然检测为高（USB_ID0 保持不变），说明有主机对 usb2otg 端口供电，驱动会切换进入 device 模式。
3. 默认状态下，USB_ID0 突然检测被拉低（直到 vbus-gpio 被拉高前，VBUS_ON0 不能为高）后，在 vbus-gpio 被驱动拉高以前，VBUS_ON0 不能为高。驱动会操作 vbus-gpio 输出高，随后 usb2otg 端口才能对外提供 VBUS 5V 供电，接着硬件需要拉高 VBUS_ON0（建议 VBUS_ON0 与实际 usb2otg 端口的 VBUS 连接），驱动检测到 VBUS_ON0 拉高后，切换进入 host 模式。

## usb2.0host

usb2.0host 控制器的设备树节点为 ehci1，对应的 phy 节点为 usbphy1，需要使能这两个节点，节点无配置参数。如果对外 vbus 有动态开关电路，需要配置 pinctrl 属性子节点。

## usb3.0drd

usb3.0drd 控制器的设备树节点为 usbdrd3，对应 high-speed utmi phy 节点为 usb2phy，对应 superspeed pipe phy 节点为 combphy，使用 usb3.0drd 控制器时需要使能这两个节点。phy 节点无参数配置。

usb3.0drd 控制器有部分参数通过 dts 的 usbdrd3 节点的子节点 dwc3 节点配置，需要配置以下参数：

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
        };
};
```

### 以 device only 模式工作

usb3.0drd 控制器的角色通过 usbdrd3 节点的子节点 dwc3 的 dr_mode 属性配置，可选 host、peripheral、otg。dr_mode 属性配置为 peripheral 则以 device only 模式工作。

### 以 host only 模式工作

参考 以 device only 模式工作 小节，dr_mode 属性配置为 host 则以 host only 模式工作。

### 以 otg 模式工作

配置 dr_mode 为 otg 模式时，dts 节点中需要配置 usb-role-switch 选项。可以通过 role-switch-default-mode 配置对应的默认角色，可选值为 host、peripheral。

```c
dwc3@c0a00000 {
        dr_mode = "otg";
        usb-role-switch;
        .... 其他参数省略，请参照上面的配置
        role-switch-default-mode = "host";
};
```

配置后，/sys/class/usb_role/下会出现一个 c0a00000.dwc3-role-switch 节点，目前 dwc3 驱动未打开通过 usb_role 框架的 sysfs 切换角色的功能，可以通过 debugfs 进行角色切换：

```c
# 查看控制器当前角色：
cat /sys/kernel/debug/usb/c0a00000.dwc3/mode
# 切换至 host 角色：
echo host > /sys/kernel/debug/usb/c0a00000.dwc3/mode
# 切换至 device 角色：
echo device > /sys/kernel/debug/usb/c0a00000.dwc3/mode
```

以上是支持手动切换控制器角色的配置说明，如果需要支持自动检测 otg 的功能需要配置额外的检测芯片驱动，如 typec 端口控制器驱动接入内核 typec 框架或 pin 脚检测硬件驱动接入 extcon 框架。