介绍camera的配置方法。

和 camera 相关的 dts 配置主要分布在以下几个文件（方案间可能会有细微差别）：

```bash
路径：arch/riscv/boot/dts/spacemit/k1-x-camera-sensor.dtsi
作用：各类sensor的配置信息  

路径：arch/riscv/boot/dts/spacemit/k1-x-camera-sdk.dtsi
作用：ccic、csiphy、isp、vi、cpp的配置信息  

路径：arch/riscv/boot/dts/spacemit/k1-x_pinctrl.dtsi
作用：camera所依赖的pinctrl配置信息

路径：arch/riscv/boot/dts/spacemit/k1-x_xxx.dts
作用：不同方案的board相关配置
```

## pinctrl

采用 pinctrl 章节方法找到 camera 使用的 pins 组，例如 camera0 可以采用 k1-x_pinctrl.dtsi 中定义的 pinctrl_camera0 组，进行 mclk 引脚的配置。

## gpio

查看开发板原理图，找到 mipi csi(0/1/2)硬件接口的复位信号 gpio 和上下电信号 gpio，通常至少会有一组 GPIO。假设 mipi csi0 硬件接口复位 gpio 为 gpio 111，上下电信号 gpio 为 gpio 113，且接入为 camera0：imx135 mipi。（建议 camera ID 和 mipi csi 的编号对应）

方案 dts 中 backsensor 使用 gpio 110 如下。

```c
路径：arch/riscv/boot/dts/spacemit/k1-x_xxx.dts

/* imx315 */
&backsensor {
        pwdn-gpios = <&gpio 113 GPIO_ACTIVE_HIGH>;
        reset-gpios = <&gpio 111 GPIO_ACTIVE_HIGH>;

        status = "okay";
};

路径：arch/riscv/boot/dts/spacemit/k1-x-camera-sensor.dtsi
&soc {
        /* imx315 */
        backsensor: cam_sensor@0 {
                cell-index = <0>;

                status = "okay";
        };
};
```

pwdn-gpios, reset-gpios 跟 sensor 模组的供电配置有关，sensor 驱动中使用这组配置完成 sensor 的上、下电和 reset 操作。不同的 sensor 模组配置可能不一样，bring up 时需要仔细修改。

## dts

### sensor 配置

k1-x-camera-sensor.dtsi 内定义的 sensor 配置如下：

```c
    backsensor: cam_sensor@0 {
            cell-index = <0>;
            twsi-index = <0>;
            dphy-index = <0>;
            compatible = "spacemit,cam-sensor";
            clocks = <&ccu CLK_CAMM0>;
            clock-names = "cam_mclk0";

            /*    该部分内容已经被移动到顶层的dts
                afvdd28-supply = <&ldo_12>;
                avdd28-supply = <&ldo_10>;
                dvdd12-supply = <&ldo_20>;
                iovdd18-supply = <&ldo_11>;
                cam-supply-names = "afvdd28", "avdd28", "dvdd12", "iovdd18";
            */
    
            ......
            status = "okay";
    };
```

cell-index 表示整个 sensor 所在的 device ID，这个 device ID 和上层使用的 sensor device ID 完全匹配。

twsi-index 表示 sensor 使用的 I2C core 的 ID，使用前要确保对应的 i2c bus dts 配置已经开启，具体请参阅 i2c 章节。

dphy-index 表示 sensor 使用的 PHY ID。

clocks/clock-names 表示 sensor 使用的 mclk 的时钟源。
