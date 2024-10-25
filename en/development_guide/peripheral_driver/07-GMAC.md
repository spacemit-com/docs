介绍GMAC的功能和使用方法。

Gmac dts 配置，需要确定以太网使用的 pin 组，phy 复位 gpio，phy 型号及地址。tx phase 和 rx phase 一般采用默认值。

## pinctrl

查看开发板原理图，找到 gmac 使用的 pin 组。

假设 eth0 使用 GPIO00~GPIO14、GPIO45 pin 组，且配置可以采用 k1-x_pinctrl.dtsi 中 pinctrl_gmac0 组配置。

方案 dts 中 eth0 使用 gmac0 pinctrl 如下

```c
&eth0 {
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_gmac0>;
};
```

## gpio

查看开发板原理图，找到以太网 phy 复位信号 gpio，假设 eth0 phy 复位 gpio 为 gpio 110。

方案 dts 中 eth0 使用 gpio 110 如下。

```c
&eth0 {
    emac,reset-gpio = <&gpio 110 0>;
}
```

## phy 配置

### phy 标识

查看开发板原理图，确认以太网 phy 的型号和 phy id。

如以太网 phy RTL8821F-CG，其 phy id 为 001c.c916。

Phy id 信息可以查找 phy spec 或联系 phy 厂商提供。

### phy 地址

查看开发板原理图，确认以太网 phy 的地址，假设为 1。

### phy 配置

根据 5.3.1 和 5.3.2 得到的 phy 标识 id 和 phy 地址，对 phy 进行配置。

方案 dts eth0 配置如下

```c
&eth0 {
    ...
    mdio-bus {
                #address-cells = <0x1>;
                #size-cells = <0x0>;
                rgmii0: phy@0 {
                        compatible = "ethernet-phy-id001c.c916";
                        device_type = "ethernet-phy";
                        reg = <0x1>;
                        phy-mode = "rgmii";
                };
    };
};
```

## tx phase 和 rx phase

Tx-phase 默认值为 90，rx-phase 为 73。

不同的板子 tx-phase 和 rx-phase 可能需要调整。如果 eth0 端口可以 up，但是分配不到 ip 地址，需要联系进迭时空调整 tx-phase 和 rx-phase。

```c
&eth0 {
    tx-phase = <90>;
    rx-phase = <73>;
};
```

## dts 配置

综合开发板以太网硬件信息，配置如下。

```c
&eth0 {
        pinctrl-names = "default";
        pinctrl-0 = <&pinctrl_gmac0>;

        emac,reset-gpio = <&gpio 110 0>;
        emac,reset-active-low;
        emac,reset-delays-us = <0 10000 100000>;

        /* store forward mode */
        tx-threshold = <1518>;
        rx-threshold = <12>;
        tx-ring-num = <128>;
        rx-ring-num = <128>;
        dma-burst-len = <5>;

        ref-clock-from-phy;

        clk-tuning-enable;
        clk-tuning-by-delayline;
        tx-phase = <90>;
        rx-phase = <73>;

        phy-handle = <&rgmii0>;

        status = "okay";

        mdio-bus {
                #address-cells = <0x1>;
                #size-cells = <0x0>;
                rgmii0: phy@0 {
                        compatible = "ethernet-phy-id001c.c916";
                        device_type = "ethernet-phy";
                        reg = <0x1>;
                        phy-mode = "rgmii";
                };
        };
};
```
