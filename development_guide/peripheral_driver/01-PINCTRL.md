介绍PIN的功能和使用方法。

## pin 配置参数

对 pin id、复用功能和属性进行定义。

详细定义`linux-6.1/include/dt-bindings/pinctrl/k1-x-pinctrl.h`。

### pin id

即 pin 编号。

K1 pin 编号范围1~147，对应宏定义 `GPIO_00 ~ GPIO_127`。

### pin 功能

k1 pin 支持复用选择。

k1 pin 复用功能列表见[K1 Pin Multiplex.xls]。

pin 的复用功能号为 0~7，分别定义为 `MUX_MODE0 ~ MUX_MODE7`。

### pin 属性

pin 的属性包括边沿检测、上下拉和驱动能力。

#### 边沿检测

采用功能 pin 唤醒系统时，设置产生唤醒事件的信号检测方式。

支持如下四种模式：

- 边沿检测关闭：`EDGE_NONE`
- 上升沿检测：`EDGE_RISE`
- 下降沿检测：`EDGE_FALL`
- 上升和下降沿：`EDGE_BOTH`

#### 上下拉

支持如下三种模式：

- 上下拉禁止：`PULL_DIS`
- 上拉：`PULL_UP`
- 下拉：`PULL_DOWN`

#### 驱动能力

1. pin 电压为 1.8v

分为 4 级，值越大，驱动能力越强。

- PAD_1V8_DS0
- PAD_1V8_DS1
- PAD_1V8_DS2
- PAD_1V8_DS3

2. pin 电压为 3.3v

分为 7 级，值越大，驱动能力越强

- PAD_3V_DS0
- PAD_3V_DS1
- PAD_3V_DS2
- PAD_3V_DS3
- PAD_3V_DS4
- PAD_3V_DS5
- PAD_3V_DS6
- PAD_3V_DS7

## pin 配置定义

### 单个 pin 配置

选定 pin 功能，设置 pin 的边沿检测，上下拉和驱动能力。

采用宏 K1X_PADCONF 进行设置, 格式为 pin_id, mux_mode, pin_config。

举例: 将 pin GPIO_00 设置为 gmac0 rxdv 功能，且关闭边沿检测，关闭上下拉，驱动能力设置为 2(1.8v)。

查看 k1 pin 功能复用列表 [K1 Pin Multiplex.xls]，GPIO_00 要设置成 gmac0 rxdv 功能，需要设置功能模式为 1, 即 MUX_MODE1。

设置如下:

```c
K1X_PADCONF(GPIO_00,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_rxdv */
```

### 定义一组 pin

对控制器(如 gmac、pcie、usb 和 emmc 等)使用的功能 pin 组进行配置。

默认的功能 pin 组定义，`linux-6.1/arch/riscv/boot/dts/spacemit/k1-x_pinctrl.dtsi`。

1. 功能 pin 组是否在 k1-x_pinctrl.dtsi 有定义，如果已定义且满足配置，直接使用；如果未定义或配置不满足，则按照第 2 步进行设置；
2. 设置控制器使用的 pin 组

以 eth0 为例，假设开发板 eth0 pin 组使用 GPIO00~GPIO14、GPIO45，且 tx 需要使能上拉。

k1-x_pinctrl.dtsi 中 gmac0 pins 默认定义

```c
pinctrl_gmac0: gmac0_grp {
        pinctrl-single,pins =<
            K1X_PADCONF(GPIO_00,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_rxdv */
            K1X_PADCONF(GPIO_01,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_rx_d0 */
            K1X_PADCONF(GPIO_02,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_rx_d1 */
            K1X_PADCONF(GPIO_03,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_rx_clk */
            K1X_PADCONF(GPIO_04,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_rx_d2 */
            K1X_PADCONF(GPIO_05,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_rx_d3 */
            K1X_PADCONF(GPIO_06,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_tx_d0 */
            K1X_PADCONF(GPIO_07,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_tx_d1 */
            K1X_PADCONF(GPIO_08,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_tx */
            K1X_PADCONF(GPIO_09,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_tx_d2 */
            K1X_PADCONF(GPIO_10,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_tx_d3 */
            K1X_PADCONF(GPIO_11,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_tx_en */
            K1X_PADCONF(GPIO_12,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_mdc */
            K1X_PADCONF(GPIO_13,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_mdio */
            K1X_PADCONF(GPIO_14,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_int_n */
            K1X_PADCONF(GPIO_45,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_clk_ref */
        >;
};
```

Tx pin 的上下拉功能不满足，默认定义为关闭上下拉，当前需要使能上拉。

有两种方法：

1. 方案 dts 重写 pin 组默认定义
2. 方案 dts 增加一组 pin 定义

下面分别进行介绍。

1. 重写 pin 组默认定义

在方案 dts 文件中增加如下配置，重写 gmac0 默认配置，将 gmac0 tx 设置为上拉。

```c
&pinctrl {
    pinctrl_gmac0: gmac0_grp {
        pinctrl-single,pins =<
            K1X_PADCONF(GPIO_00,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_rxdv */
            K1X_PADCONF(GPIO_01,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_rx_d0 */
            K1X_PADCONF(GPIO_02,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_rx_d1 */
            K1X_PADCONF(GPIO_03,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_rx_clk */
            K1X_PADCONF(GPIO_04,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_rx_d2 */
            K1X_PADCONF(GPIO_05,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_rx_d3 */
            K1X_PADCONF(GPIO_06,    MUX_MODE1, (EDGE_NONE | PULL_PULL | PAD_1V8_DS2))   /* gmac0_tx_d0 */
            K1X_PADCONF(GPIO_07,    MUX_MODE1, (EDGE_NONE | PULL_PULL | PAD_1V8_DS2))   /* gmac0_tx_d1 */
            K1X_PADCONF(GPIO_08,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_tx */
            K1X_PADCONF(GPIO_09,    MUX_MODE1, (EDGE_NONE | PULL_PULL | PAD_1V8_DS2))   /* gmac0_tx_d2 */
            K1X_PADCONF(GPIO_10,    MUX_MODE1, (EDGE_NONE | PULL_PULL | PAD_1V8_DS2))   /* gmac0_tx_d3 */
            K1X_PADCONF(GPIO_11,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_tx_en */
            K1X_PADCONF(GPIO_12,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_mdc */
            K1X_PADCONF(GPIO_13,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_mdio */
            K1X_PADCONF(GPIO_14,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_int_n */
            K1X_PADCONF(GPIO_45,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_clk_ref */
        >;
    };
};
```

2. 新定义 gmac0 pin 组

在方案 dts 文件中增加如下配置，将 gmac0 tx 设置为上拉。

```c
&pinctrl {
    pinctrl_gmac0_1: gmac0_1_grp {
        pinctrl-single,pins =<
            K1X_PADCONF(GPIO_00,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_rxdv */
            K1X_PADCONF(GPIO_01,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_rx_d0 */
            K1X_PADCONF(GPIO_02,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_rx_d1 */
            K1X_PADCONF(GPIO_03,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_rx_clk */
            K1X_PADCONF(GPIO_04,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_rx_d2 */
            K1X_PADCONF(GPIO_05,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_rx_d3 */
            K1X_PADCONF(GPIO_06,    MUX_MODE1, (EDGE_NONE | PULL_PULL | PAD_1V8_DS2))   /* gmac0_tx_d0 */
            K1X_PADCONF(GPIO_07,    MUX_MODE1, (EDGE_NONE | PULL_PULL | PAD_1V8_DS2))   /* gmac0_tx_d1 */
            K1X_PADCONF(GPIO_08,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_tx */
            K1X_PADCONF(GPIO_09,    MUX_MODE1, (EDGE_NONE | PULL_PULL | PAD_1V8_DS2))   /* gmac0_tx_d2 */
            K1X_PADCONF(GPIO_10,    MUX_MODE1, (EDGE_NONE | PULL_PULL | PAD_1V8_DS2))   /* gmac0_tx_d3 */
            K1X_PADCONF(GPIO_11,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_tx_en */
            K1X_PADCONF(GPIO_12,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_mdc */
            K1X_PADCONF(GPIO_13,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_mdio */
            K1X_PADCONF(GPIO_14,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_int_n */
            K1X_PADCONF(GPIO_45,    MUX_MODE1, (EDGE_NONE | PULL_DIS | PAD_1V8_DS2))   /* gmac0_clk_ref */
        >;
    };
};
```

## pin 使用

eth0 引用方案重写定义的 pinctrl_gmac0

```c
eth0 {
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_gmac0>;
};
```

或者引用方案新增加的 pinctrl_gmac0_1

```c
eth0 {
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_gmac0_1>;
};
```
