介绍SDHC的功能和使用方法。

## pinctrl

sdhc 一共有三个 slot，slot1 支持 sd/sdio(1/4 bit)，slot2 支持 sdio/emmc(1/4 bit)，slot3 只支持 emmc(1/4/8 bit)。

方案上一般 slot1 用于 sd，slot2 用于 sdio，slot3 用于 emmc。

sd 和 sdio 都需要配置卡的信号线对应的 pinctl 为 mode0 模式，分别对应 pinctrl_mmc1 和 pinctrl_mmc2。

mmc1 的 pinctl 还有一个 fast 模式，在时钟高于 100M 时需要切换到 pinctrl_mmc1_fast 模式。

```c
    pinctrl_mmc1: mmc1_grp {
        pinctrl-single,pins = <
            K1X_PADCONF(MMC1_DAT3, MUX_MODE0, (EDGE_NONE | PULL_UP | PAD_3V_DS4))         /* mmc1_d3 */
            K1X_PADCONF(MMC1_DAT2, MUX_MODE0, (EDGE_NONE | PULL_UP | PAD_3V_DS4))         /* mmc1_d2 */
            K1X_PADCONF(MMC1_DAT1, MUX_MODE0, (EDGE_NONE | PULL_UP | PAD_3V_DS4))         /* mmc1_d1 */
            K1X_PADCONF(MMC1_DAT0, MUX_MODE0, (EDGE_NONE | PULL_UP | PAD_3V_DS4))         /* mmc1_d0 */
            K1X_PADCONF(MMC1_CMD,  MUX_MODE0, (EDGE_NONE | PULL_UP | PAD_3V_DS4))         /* mmc1_cmd */
            K1X_PADCONF(MMC1_CLK,  MUX_MODE0, (EDGE_NONE | PULL_DOWN | PAD_3V_DS4))       /* mmc1_clk */
        >;
    };

    pinctrl_mmc1_fast: mmc1_fast_grp {
        pinctrl-single,pins = <
            K1X_PADCONF(MMC1_DAT3, MUX_MODE0, (EDGE_NONE | PULL_UP | PAD_1V8_DS3))         /* mmc1_d3 */
            K1X_PADCONF(MMC1_DAT2, MUX_MODE0, (EDGE_NONE | PULL_UP | PAD_1V8_DS3))         /* mmc1_d2 */
            K1X_PADCONF(MMC1_DAT1, MUX_MODE0, (EDGE_NONE | PULL_UP | PAD_1V8_DS3))         /* mmc1_d1 */
            K1X_PADCONF(MMC1_DAT0, MUX_MODE0, (EDGE_NONE | PULL_UP | PAD_1V8_DS3))         /* mmc1_d0 */
            K1X_PADCONF(MMC1_CMD,  MUX_MODE0, (EDGE_NONE | PULL_UP | PAD_1V8_DS3))         /* mmc1_cmd */
            K1X_PADCONF(MMC1_CLK,  MUX_MODE0, (EDGE_NONE | PULL_DOWN | PAD_1V8_DS3))       /* mmc1_clk */
        >;
    };
```

## gpio

sd 的检测是通过 gpio 完成的，需要按实际原理图来配置卡检测的 gpio。

```c
&sdhci0 {
        cd-gpios = <&gpio 80 0>;
        cd-inverted;
}；
```

比如方案使用 gpio80 来做卡的检测，还需要配置 gpio80 的 pintcl 功能。

```c
&pinctrl {
        pinctrl-single,gpio-range = <
                &range GPIO_80  1 (MUX_MODE0 | EDGE_NONE | PULL_UP   | PAD_3V_DS4)
        >;
};

&gpio{
        gpio-ranges = <
                &pinctrl 80  GPIO_80  4
        >;
};
```

## 电源配置

sd 和 sdio 需要配置两个电源，分别是 vmmc-supply 和 vqmmc-supply，分别对应卡的功能和 io 供电，vqmmc-supply 会根据卡的运行模式动态切换电源，硬件设计上需要确保能支持 3.3v 和 1.8v。

emmc 设计上会保证供电，不需要配置电源。

```c
&sdhci0 { 
        vmmc-supply = <&dcdc_4>;
        vqmmc-supply = <&ldo_1>;
}；
```

## tuning 配置

sd 跑高速模式下需要进行 tuning，不同的硬件版型都需要调整 tx 和 rx 的相关参数。

## dts 配置

sd 的完整方案配置如下：

```c
&sdhci0 {
        pinctrl-names = "default","fast";
        pinctrl-0 = <&pinctrl_mmc1>;
        pinctrl-1 = <&pinctrl_mmc1_fast>;
        bus-width = <4>;
        cd-gpios = <&gpio 80 0>;
        cd-inverted;
        vmmc-supply = <&dcdc_4>;
        vqmmc-supply = <&ldo_1>;
        no-mmc;
        no-sdio;
        spacemit,sdh-host-caps-disable = <(
                        MMC_CAP_UHS_SDR12 |
                        MMC_CAP_UHS_SDR25
                        )>;
        spacemit,sdh-quirks = <(
                        SDHCI_QUIRK_BROKEN_CARD_DETECTION |
                        SDHCI_QUIRK_INVERTED_WRITE_PROTECT |
                        SDHCI_QUIRK_BROKEN_TIMEOUT_VAL
                        )>;
        spacemit,sdh-quirks2 = <(
                        SDHCI_QUIRK2_PRESET_VALUE_BROKEN |
                        SDHCI_QUIRK2_BROKEN_PHY_MODULE |
                        SDHCI_QUIRK2_SET_AIB_MMC
                        )>;
        spacemit,aib_mmc1_io_reg = <0xD401E81C>;
        spacemit,apbc_asfar_reg = <0xD4015050>;
        spacemit,apbc_assar_reg = <0xD4015054>;
        spacemit,rx_dline_reg = <0x0>;
        spacemit,tx_dline_reg = <0x0>;
        spacemit,tx_delaycode = <0xA0>;
        spacemit,rx_tuning_limit = <50>;
        spacemit,sdh-freq = <204800000>;
        status = "okay";
};
```

emmc 的完整方案配置如下：

```c
/* eMMC */
&sdhci2 {
        bus-width = <8>;
        non-removable;
        mmc-hs400-1_8v;
        mmc-hs400-enhanced-strobe;
        no-sd;
        no-sdio;
        spacemit,sdh-quirks = <(
                        SDHCI_QUIRK_BROKEN_CARD_DETECTION |
                        SDHCI_QUIRK_BROKEN_TIMEOUT_VAL
                        )>;
        spacemit,sdh-quirks2 = <(
                        SDHCI_QUIRK2_PRESET_VALUE_BROKEN
                        )>;
        spacemit,sdh-freq = <375000000>;
        status = "okay";
};
```
