介绍GPIO的功能和使用方法。

方案 gpio 使用分成 3 步。

1. 方案使用的 gpio 描述
2. 对相应的 pin 进行设置
3. gpio 的使用

假设某方案使用的 gpio 及对应 pin 情况如下表 1，本章基于此例子进行描述。

说明:

1. pin 编号在 linux-6.1\include\dt-bindings\pinctrl\k1-x-pinctrl.h 中定义。
2. pin 配置相同:

表示一组 pin 设置成 gpio 功能时配置相同，即 mux mode、上下拉、边沿检测、驱动能力配置相同。

表 1

## 方案gpio描述

用来描述方案使用的所有gpio。

采用linux gpio框架gpio-ranges属性进行定义。如果某段gpio对应的pin编号也连续，则定义为一组。

上述例子方案dts文件gpio控制器定义

```c
&gpio{
        gpio-ranges = <
                &pinctrl 49  GPIO_49  2
                &pinctrl 58  GPIO_58  1
                &pinctrl 63  GPIO_63  5
                &pinctrl 70  PRI_TDI  4
                &pinctrl 74  GPIO_74  1
                &pinctrl 80  GPIO_80  4
                &pinctrl 90  GPIO_90  3
                &pinctrl 96  DVL0     2
                &pinctrl 110 GPIO_110 1
                &pinctrl 114 GPIO_114 3
                &pinctrl 123 GPIO_123 5
        >;
};
```

## gpio pin配置

将方案使用的gpio对应的pin设置成gpio功能，并进行配置(边沿检测/上下拉/驱动能力)。

采用pinctrl-single,gpio-range属性设置。如果存在某段pin编号连续且配置相同，则配置为一组。

表 1中的gpio对应pinctrl配置如下。pin配置参数参考[pin配置参数](PINCTRL#pin-配置参数)。

```c
&pinctrl {
        pinctrl-single,gpio-range = <
                &range GPIO_49  2 (MUX_MODE0 | EDGE_NONE | PULL_UP   | PAD_3V_DS4)
                &range GPIO_58  1 (MUX_MODE0 | EDGE_NONE | PULL_DOWN | PAD_1V8_DS2)
                &range GPIO_63  2 (MUX_MODE0 | EDGE_NONE | PULL_DOWN | PAD_1V8_DS2)
                &range GPIO_65  1 (MUX_MODE0 | EDGE_NONE | PULL_UP   | PAD_1V8_DS2)
                &range GPIO_66  2 (MUX_MODE0 | EDGE_NONE | PULL_UP   | PAD_3V_DS4)
                &range PRI_TDI  2 (MUX_MODE1 | EDGE_NONE | PULL_UP   | PAD_1V8_DS2)
                &range PRI_TCK  1 (MUX_MODE1 | EDGE_NONE | PULL_DOWN | PAD_1V8_DS2)
                &range PRI_TDO  1 (MUX_MODE1 | EDGE_NONE | PULL_UP   | PAD_1V8_DS2)
                &range GPIO_74  1 (MUX_MODE0 | EDGE_NONE | PULL_UP   | PAD_1V8_DS2)
                &range GPIO_80  1 (MUX_MODE0 | EDGE_NONE | PULL_UP   | PAD_3V_DS4)
                &range GPIO_81  3 (MUX_MODE0 | EDGE_NONE | PULL_UP   | PAD_1V8_DS2)
                &range GPIO_90  1 (MUX_MODE0 | EDGE_NONE | PULL_DOWN | PAD_1V8_DS2)
                &range GPIO_91  2 (MUX_MODE0 | EDGE_NONE | PULL_UP   | PAD_1V8_DS2)
                &range DVL0     2 (MUX_MODE1 | EDGE_NONE | PULL_DOWN | PAD_1V8_DS2)
                &range GPIO_110 1 (MUX_MODE0 | EDGE_NONE | PULL_DOWN | PAD_1V8_DS2)
                &range GPIO_114 1 (MUX_MODE0 | EDGE_NONE | PULL_DOWN | PAD_1V8_DS2)
                &range GPIO_115 1 (MUX_MODE0 | EDGE_NONE | PULL_DOWN | PAD_1V8_DS2)
                &range GPIO_116 1 (MUX_MODE0 | EDGE_NONE | PULL_UP   | PAD_1V8_DS2)
                &range GPIO_123 1 (MUX_MODE0 | EDGE_NONE | PULL_DOWN | PAD_1V8_DS2)
                &range GPIO_124 1 (MUX_MODE0 | EDGE_NONE | PULL_UP   | PAD_1V8_DS2)
                &range GPIO_125 3 (MUX_MODE0 | EDGE_NONE | PULL_DOWN | PAD_1V8_DS2)
        >;
};
```

## gpio 使用

若方案 eth0 使用 gpio 110 作为 phy 的 reset 信号，则 eth0 使用 gpio 110 如下。

```c
&eth0 {
    emac,reset-gpio = <&gpio 110 0>;
};
```
