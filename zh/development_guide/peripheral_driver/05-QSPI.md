介绍QSPI的功能和使用方法。

## pinctrl

查看方案原理图，找到 qsoi 使用的 pin 组。参考 1.2.2 节，确定 qspi 使用的 pin 组。

假设 qspi 可以直接采用 k1-x_pinctrl.dtsi 中定义 pinctrl_qspi 组。

## spi 设备配置

需要确认 spi 设备类型，qspi 与 spi 设备通信频率和倍速。

### 设备类型

确认 qspi 下连接的 spi 设备类型，是 spi-nor 还是 spi-nand。

### 通信频率

qspi 控制器和 spi 设备最大通信速率。

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

### spi 设备 dts

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
