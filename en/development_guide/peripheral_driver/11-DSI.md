介绍DSI的功能和使用方法。

提供 1 个 MIPI DSI 接口，支持 4 lane, 速率 1.2Gbps/lane, 支持最高 1920x1440@60FPS。

## dts 配置

MIPI DSI 相关配置内容在 k1-x-lcd.dtsi，包括 irq, clock 及 power domain 相关配置。

```c
display-subsystem-dsi {
        compatible = "spacemit,saturn-le";
        reg = <0 0xC0340000 0 0x2A000>;
        ports = <&dpu_online2_dsi>;
};

dpu_online2_dsi: port@c0340000 {
        compatible = "spacemit,dpu-online2";
        interrupt-parent = <&intc>;
        interrupts = <90>, <89>;
        interrupt-names = "ONLINE_IRQ", "OFFLINE_IRQ";
        clocks = <&ccu CLK_DPU_PXCLK>,
                <&ccu CLK_DPU_MCLK>,
                <&ccu CLK_DPU_HCLK>,
                <&ccu CLK_DPU_ESC>,
                <&ccu CLK_DPU_BIT>;
        clock-names = "pxclk", "mclk", "hclk", "escclk", "bitclk";
        assigned-clocks = <&ccu CLK_DPU_PXCLK>,
                <&ccu CLK_DPU_MCLK>,
                <&ccu CLK_DPU_HCLK>,
                <&ccu CLK_DPU_ESC>,
                <&ccu CLK_DPU_BIT>;
        assigned-clock-parents = <&ccu CLK_PLL1_614>,
                <&ccu CLK_PLL1_307P2>,
                <0>,
                <&ccu CLK_PLL1_51P2_AP>,
                <&ccu CLK_PLL1_1228>;
        resets = <&reset RESET_MIPI>,
                <&reset RESET_LCD_MCLK>,
                <&reset RESET_LCD>,
                <&reset RESET_DSI_ESC>;
        reset-names= "dsi_reset", "mclk_reset", "lcd_reset","esc_reset";
        power-domains = <&power K1X_PMU_LCD_PWR_DOMAIN>;
        pipeline-id = <ONLINE2>;
        ip = "spacemit-saturn";
        spacemit-dpu-min-mclk = <40960000>;
        status = "disabled";

        dpu_online2_dsi_out: endpoint@0 {
                remote-endpoint = <&dsi2_in>;
        };

};

dsi2: dsi2@d421a800 {
        compatible = "spacemit,dsi2-host";
        #address-cells = <1>;
        #size-cells = <0>;
        reg = <0 0xD421A800 0 0x200>;
        interrupt-parent = <&intc>;
        interrupts = <95>;
        ip = "synopsys-dhost";
        dev-id = <2>;
        status = "disabled";

        ports {
                #address-cells = <1>;
                #size-cells = <0>;
               
                port@0 {
                        reg = <0>;
                        dsi2_out: endpoint {
                                remote-endpoint = <&dphy2_in>;
                        };
                };

                port@1 {
                        reg = <1>;
                        dsi2_in: endpoint {
                                remote-endpoint = <&dpu_online2_dsi_out>;
                        };
                };

        };
};

dphy2: dphy2@d421a800 {
        compatible = "spacemit,dsi2-phy";
        #address-cells = <1>;
        #size-cells = <0>;
        reg = <0 0xD421A800 0 0x200>;
        ip = "spacemit-dphy";
        dev-id = <2>;
        status = "okay";

        port@1 {
                reg = <1>;
                dphy2_in: endpoint {
                        remote-endpoint = <&dsi2_out>;
                };
        };
};

```

默认关闭 MIPI DSI 功能，在方案配置文件中选择 panel，及是否打开

```c
&dpu_online2_dsi {
        memory-region = <&dpu_resv>;
        spacemit-dpu-bitclk = <1000000000>;
        dsi_1v2-supply = <&ldo_5>;
        vin-supply-names = "dsi_1v2";
        status = "okay";
};

&dsi2 {
        status = "okay";

        panel2: panel2@0 {
                status = "ok";
                compatible = "spacemit,mipi-panel2";
                reg = <0>;

                gpios-reset = <81>;                     // reset pin脚
                gpios-dc = <82 83>;                     // power pin脚
                id = <2>;
                delay-after-reset = <10>;               // reset 延时 单位ms
                force-attached = "lcd_gx09inx101_mipi"; // 选择panel
        };
};

&lcds {
        status = "okay";
};
```

## panel 列表

已完成功能调试的 panel，相关 dtsi 文件放置在 lcd 目录，主要参数包括：MIPI DSI 工作模式，panel 的初始化序列，及 timing。

例如 panel lcd_gx09inx101_mipi 配置内容如下：

```c
/ { lcds: lcds {
        lcd_gx09inx101_mipi: lcd_gx09inx101_mipi {
                dsi-work-mode = <1>; /* video burst mode*/
                dsi-lane-number = <4>;
                dsi-color-format = "rgb888";
                phy-bit-clock = <911000>;        /* kbps */
                phy-escape-clock = <52000>;/* kHz */

                width-mm = <142>;
                height-mm = <228>;

                use-dcs-write;

                /*mipi info*/
                height = <1920>;
                width = <1200>;
                hfp = <80>;
                hbp = <40>;
                hsync = <10>;
                vfp = <20>;
                vbp = <16>;
                vsync = <4>;
                fps = <60>;
                work-mode = <0>;
                rgb-mode = <3>;
                lane-number = <4>;
                phy-freq = <624000>;
                split-enable = <0>;
                eotp-enable = <0>;
                burst-mode = <2>;
                esd-check-enable = <0>;

                /* DSI_CMD, DSI_MODE, timeout, len, cmd */
                initial-command = [
                        39 01 00 02 B0 01
                        39 01 00 02 C3 4F
                        39 01 00 02 C4 40
                        39 01 00 02 C5 40
                        39 01 00 02 C6 40
                        39 01 00 02 C7 40
                        39 01 00 02 C8 4D
                        39 01 00 02 C9 52
                        39 01 00 02 CA 51
                        39 01 00 02 CD 5D
                        39 01 00 02 CE 5B
                        39 01 00 02 CF 4B
                        39 01 00 02 D0 49
                        39 01 00 02 D1 47
                        39 01 00 02 D2 45
                        39 01 00 02 D3 41
                        39 01 00 02 D7 50
                        39 01 00 02 D8 40
                        39 01 00 02 D9 40
                        39 01 00 02 DA 40
                        39 01 00 02 DB 40
                        39 01 00 02 DC 4E
                        39 01 00 02 DD 52
                        39 01 00 02 DE 51
                        39 01 00 02 E1 5E
                        39 01 00 02 E2 5C
                        39 01 00 02 E3 4C
                        39 01 00 02 E4 4A
                        39 01 00 02 E5 48
                        39 01 00 02 E6 46
                        39 01 00 02 E7 42
                        39 01 00 02 B0 03
                        39 01 00 02 BE 03
                        39 01 00 02 CC 44
                        39 01 00 02 C8 07
                        39 01 00 02 C9 05
                        39 01 00 02 CA 42
                        39 01 00 02 CD 3E
                        39 01 00 02 CF 60
                        39 01 00 02 D2 04
                        39 01 00 02 D3 04
                        39 01 00 02 D4 01
                        39 01 00 02 D5 00
                        39 01 00 02 D6 03
                        39 01 00 02 D7 04
                        39 01 00 02 D9 01
                        39 01 00 02 DB 01
                        39 01 00 02 E4 F0
                        39 01 00 02 E5 0A
                        39 01 00 02 B0 00
                        39 01 00 02 BD 50
                        39 01 00 02 C2 08
                        39 01 00 02 C4 10
                        39 01 00 02 CC 00
                        39 01 00 02 B0 02
                        39 01 00 02 C0 00
                        39 01 00 02 C1 0A
                        39 01 00 02 C2 20
                        39 01 00 02 C3 24
                        39 01 00 02 C4 23
                        39 01 00 02 C5 29
                        39 01 00 02 C6 23
                        39 01 00 02 C7 1C
                        39 01 00 02 C8 19
                        39 01 00 02 C9 17
                        39 01 00 02 CA 17
                        39 01 00 02 CB 18
                        39 01 00 02 CC 1A
                        39 01 00 02 CD 1E
                        39 01 00 02 CE 20
                        39 01 00 02 CF 23
                        39 01 00 02 D0 07
                        39 01 00 02 D1 00
                        39 01 00 02 D2 00
                        39 01 00 02 D3 0A
                        39 01 00 02 D4 13
                        39 01 00 02 D5 1C
                        39 01 00 02 D6 1A
                        39 01 00 02 D7 13
                        39 01 00 02 D8 17
                        39 01 00 02 D9 1C
                        39 01 00 02 DA 19
                        39 01 00 02 DB 17
                        39 01 00 02 DC 17
                        39 01 00 02 DD 18
                        39 01 00 02 DE 1A
                        39 01 00 02 DF 1E
                        39 01 00 02 E0 20
                        39 01 00 02 E1 23
                        39 01 00 02 E2 07
                        39 01 F0 01 11
                        39 01 28 01 29
                ];
                sleep-in-command = [
                        39 01 78 01 28
                        39 01 78 01 10
                ];
                sleep-out-command = [
                        39 01 96 01 11
                        39 01 32 01 29
                ];
                read-id-command = [
                        37 01 00 01 05
                        14 01 00 05 fb fc fd fe ff
                ];

                display-timings {
                        timing0 {
                                clock-frequency = <156400000>;
                                hactive = <1200>;
                                hfront-porch = <80>;
                                hback-porch = <40>;
                                hsync-len = <10>;
                                vactive = <1920>;
                                vfront-porch = <20>;
                                vback-porch = <16>;
                                vsync-len = <4>;
                                vsync-active = <1>;
                                hsync-active = <1>;
                        };
                };
        };
};};
```