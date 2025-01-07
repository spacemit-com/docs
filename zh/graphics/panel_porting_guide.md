# Spacemit 屏幕调试文档
介绍spacemit k1平台 Uboot 和 kernel 的 mipi 与 hdmi 屏幕驱动用例和调试方法。
## 模块介绍

spacemit 平台 Display 模块使用 DRM 框架，DRM 全称是 Direct Rendering Manager，是Linux系统目前主流的显示框架，适应当前显示硬件的特性。


![display-kms](static/diaplay-kms.jpg )
## 一、Uboot 屏幕调试
### 1.1. 源码结构介绍
spacemit 平台 Uboot 显示驱动源码结构：
```
uboot-2022.10/drivers/video$ tree spacemit
spacemit
├── dsi
│   ├── drv
│   │   ├── spacemit_dphy.c                                  // mipi dsi dphy驱动
│   │   ├── spacemit_dphy.h
│   │   ├── spacemit_dsi_common.c
│   │   ├── spacemit_dsi_drv.c                               // mipi dsi 驱动
│   │   ├── spacemit_dsi_drv.h
│   │   └── spacemit_dsi_hw.h
│   ├── include
│   │   ├── spacemit_dsi_common.h
│   │   └── spacemit_video_tx.h
│   ├── Makefile
│   └── video
│       ├── lcd                                               // panel 配置
│       │   ├── lcd_ft8201sinx101.c
│       │   ├── lcd_gx09inx101.c
│       │   ├── lcd_icnl9911c.c
│       │   ├── lcd_icnl9951r.c
│       │   ├── lcd_jd9365dah3.c
│       │   └── lcd_lt8911ext_edp_1080p.c
│       ├── spacemit_mipi_port.c
│       └── spacemit_video_tx.c
├── Kconfig
├── Makefile
├── spacemit_dpu.c                                            // dpu 驱动
├── spacemit_dpu.h
├── spacemit_edp.c                                            // eDP panel驱动
├── spacemit_hdmi.c                                           // HDMI 驱动
├── spacemit_hdmi.h
├── spacemit_mipi.c                                           // mipi 驱动
└── spacemit_mipi.h
```
### 1.2. 配置介绍
#### 1.2.1. CONFIG配置
执行 make uboot-menuconfig，进入 Device Drivers -> Graphics support，将以下配置打开(默认情况下已开启)。
```
Device Drivers  --->
  Graphics support  --->
     <*> Enable SPACEMIT Video Suppor
     <*>    HDMI port
     <*>    MIPI Port
     <*>        EDP Port
```
#### 1.2.2. HDMI dts配置
配置 hdmi 相关设备树
```c
//uboot-2022.10/arch/riscv/dts/k1-x_deb1.dts
&dpu {
	status = "okay";
};

&hdmi {
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_hdmi_0>;          //pinctrl
	status = "okay";
};
```
#### 1.2.3. MIPI dts配置

##### MIPI DSI

###### Gpio

MIPI DSI panel gpio相关配置，以k1-x_deb1方案为例： gpio81配置为panel复位pin，gpio82和gpio83配置为panel电源控制pin。
```c
//uboot-2022.10/arch/riscv/dts/k1-x_deb1.dts
&dpu {
	status = "okay";
};

&mipi_dsi {
        status = "okay";
};

&panel {
        dcp-gpios = <&gpio 82 0>;          // 配置panel 电源控制 gpio
        dcn-gpios = <&gpio 83 0>;          // 配置panel 电源控制 gpio
        backlight = <&backlight>;          // 配置背光 pwm
        reset-gpios = <&gpio 81 0>;        // 配置panel 复位 gpio
        status = "okay";
};
```
###### DSI电源配置
MIPI DSI需要配置MIPI DSI 1.2v电源。\
以k1-x_deb1方案为例： 需配置pmic ldo_5为MIPI DSI 1.2v。（方案实际可不需要配置，默认已开启）
```c
//uboot-2022.10/arch/riscv/dts/k1-x_deb1.dts
&ldo_27 {
    regulator-init-microvolt = <1200000>;
    regulator-boot-on;
    regulator-state-mem {
                    regulator-off-in-suspend;
    };
};
```
```c
//uboot-2022.10/arch/riscv/dts/k1-x_spm8821.dtsi
                    /* dldo */
        ldo_27: LDO_REG5 {
                regulator-name = "ldo5";
                regulator-min-microvolt = <500000>;
                regulator-max-microvolt = <3400000>;
                regulator-state-mem {
                        regulator-off-in-suspend;
                };
        };

```
###### PWM
通过pwm控制背光
```c
&pwm14 {
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_pwm14_1>;
	status = "okay";
};

&backlight {
	pwms = <&pwm14 0 2000>;
	default-brightness-level = <6>;
	status = "okay";
};
```
#### 1.2.4. display timing配置
Uboot阶段默认使用88000000作为pix-clock，614400000作为bit-clcok，无需额外配置
```c
//uboot-2022.10/drivers/video/spacemit/spacemit_mipi.c
	pix_clk = dev_read_u32_default(dev, "pix-clk", 88000000);
	ret = clk_set_rate(&priv->pxclk, pix_clk);

	if (ret < 0) {
		pr_err("clk_set_rate mipi dsi pxclk failed: %d\n", ret);
		return ret;
	}

	bit_clk = dev_read_u32_default(dev, "bit-clk", 614400000);
	ret = clk_set_rate(&priv->bitclk, bit_clk);
	if (ret < 0) {
		pr_err("clk_set_rate mipi dsi bitclk failed: %d\n", ret);
		return ret;
	}
```

已完成功能调试的MIPI DSI panel，相关c文件放置在lcd目录。
```
uboot-2022.10/drivers/video/spacemit/dsi/video$ tree lcd
lcd
├── lcd_ft8201sinx101.c
├── lcd_gx09inx101.c
├── lcd_icnl9911c.c
├── lcd_icnl9951r.c
├── lcd_jd9365dah3.c
└── lcd_lt8911ext_edp_1080p.c
```
### 1.3. 新增 mipi panel 配置参考实例
以lcd_gx09inx101为例
1. 在uboot-2022.10/drivers/video/spacemit/dsi/video/lcd/路径下新建lcd_gx09inx101.c
```c
/ SPDX-License-Identifier: GPL-2.0
/*
 * Copyright (C) 2023 Spacemit Co., Ltd.
 *
 */

#include <linux/kernel.h>
#include "../../include/spacemit_dsi_common.h"
#include "../../include/spacemit_video_tx.h"
#include <linux/delay.h>

#define UNLOCK_DELAY 0

struct spacemit_mode_modeinfo gx09inx101_spacemit_modelist[] = {
        {
                .name = "1200x1920-60",
                .refresh = 60,                //fps
                .xres = 1200,                 // width 像素
                .yres = 1920,                 // height 像素
                .real_xres = 1200,
                .real_yres = 1920,
                .left_margin = 40,            //hbp
                .right_margin = 80,           //hfp
                .hsync_len = 10,              //hsync
                .upper_margin = 16,           //vbp
                .lower_margin = 20,           //vfp
                .vsync_len = 4,               //vsync
                .hsync_invert = 0,
                .vsync_invert = 0,
                .invert_pixclock = 0,
                .pixclock_freq = 156*1000,    // 像素时钟
                .pix_fmt_out = OUTFMT_RGB888,
                .width = 142,                 //显示屏物理宽度
                .height = 228,                //显示屏物理长度
        },
};

struct spacemit_mipi_info gx09inx101_mipi_info = {
        .height = 1920,    // mipi dsi dphy中配置屏幕高
        .width = 1200,     // mipi dsi dphy中配置屏幕宽
        .hfp = 80,         // mipi dsi dphy中配置水平前肩（Horizontal Front Porch）
        .hbp = 40,         // mipi dsi dphy中配置水平后肩（Horizontal Back Porch）
        .hsync = 10,       // mipi dsi dphy中配置水平同步信号（Horizontal Sync）
        .vfp = 20,         // mipi dsi dphy中配置垂直前肩（Vertical Front Porch）
        .vbp = 16,         // mipi dsi dphy中配置垂直后肩（Vertical Back Porch）
        .vsync = 4,        // mipi dsi dphy中配置垂直同步信号（Vertical Sync）
        .fps = 60,         // mipi dsi dphy中配置帧率

        .work_mode = SPACEMIT_DSI_MODE_VIDEO,   /*command_mode, video_mode*/
        .rgb_mode = DSI_INPUT_DATA_RGB_MODE_888,// panel中配置mipi dsi 数据格式
        .lane_number = 4,                       // mipi dsi dphy中配置mipi dsi lane数量
        .phy_bit_clock = 614400000,             // mipi dsi dphy中配置mipi dsi dphy bit clock
        .phy_esc_clock = 51200000,              // mipi dsi dphy中配置mipi dsi dphy esc clock
        .split_enable = 0,                      // mipi dsi dphy中配置mipi dsi使能split
        .eotp_enable = 0,                       // mipi dsi dphy中配置mipi dsi使能eotp

        .burst_mode = DSI_BURST_MODE_BURST,
};

static struct spacemit_dsi_cmd_desc gx09inx101_set_id_cmds[] = {
        {SPACEMIT_DSI_SET_MAX_PKT_SIZE, SPACEMIT_DSI_LP_MODE, UNLOCK_DELAY, 1, {0x01}},
};

static struct spacemit_dsi_cmd_desc gx09inx101_read_id_cmds[] = {
        {SPACEMIT_DSI_GENERIC_READ1, SPACEMIT_DSI_LP_MODE, UNLOCK_DELAY, 1, {0xfb}},
};

static struct spacemit_dsi_cmd_desc gx09inx101_set_power_cmds[] = {
        {SPACEMIT_DSI_SET_MAX_PKT_SIZE, SPACEMIT_DSI_HS_MODE, UNLOCK_DELAY, 1, {0x1}},
};

static struct spacemit_dsi_cmd_desc gx09inx101_read_power_cmds[] = {
        {SPACEMIT_DSI_GENERIC_READ1, SPACEMIT_DSI_HS_MODE, UNLOCK_DELAY, 1, {0xA}},
};

static struct spacemit_dsi_cmd_desc gx09inx101_init_cmds[] = {
        //8279 + INX10.1
        {SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   2, {0xB0,0x01}},
        {SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   2, {0xC3,0x4F}},
        {SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   2, {0xC4,0x40}},
        {SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   2, {0xC5,0x40}},
        {SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   2, {0xC6,0x40}},
        {SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   2, {0xC7,0x40}},
        {SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   2, {0xC8,0x4D}},
        {SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   2, {0xC9,0x52}},
        {SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   2, {0xCA,0x51}},
        {SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   2, {0xCD,0x5D}},
        {SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   2, {0xCE,0x5B}},
        {SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   2, {0xCF,0x4B}},
        {SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   2, {0xD0,0x49}},

		·······

        {SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   2, {0xDA,0x19}},
        {SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   2, {0xDB,0x17}},
        {SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   2, {0xDC,0x17}},
        {SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   2, {0xDD,0x18}},
        {SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   2, {0xDE,0x1A}},
        {SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   2, {0xDF,0x1E}},
        {SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   2, {0xE0,0x20}},
        {SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   2, {0xE1,0x23}},
        {SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   2, {0xE2,0x07}},
        {SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 200, 2, {0x11, 0x00}},
        {SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE,  50, 2, {0x29, 0x00}},
};

static struct spacemit_dsi_cmd_desc gx09inx101_sleep_out_cmds[] = {
        {SPACEMIT_DSI_DCS_SWRITE,SPACEMIT_DSI_LP_MODE,200,1,{0x11}},
        {SPACEMIT_DSI_DCS_SWRITE,SPACEMIT_DSI_LP_MODE,50,1,{0x29}},
};

static struct spacemit_dsi_cmd_desc gx09inx101_sleep_in_cmds[] = {
        {SPACEMIT_DSI_DCS_SWRITE,SPACEMIT_DSI_LP_MODE,50,1,{0x28}},
        {SPACEMIT_DSI_DCS_SWRITE,SPACEMIT_DSI_LP_MODE,200,1,{0x10}},
};


struct lcd_mipi_panel_info lcd_gx09inx101 = {
        .lcd_name = "gx09inx101",
        .lcd_id = 0x8279,
        .panel_id0 = 0x1,
        .power_value = 0x14,
        .panel_type = LCD_MIPI,
        .width_mm = 142,
        .height_mm = 228,
        .dft_pwm_bl = 128,
        .set_id_cmds_num = ARRAY_SIZE(gx09inx101_set_id_cmds),
        .read_id_cmds_num = ARRAY_SIZE(gx09inx101_read_id_cmds),
        .init_cmds_num = ARRAY_SIZE(gx09inx101_init_cmds),
        .set_power_cmds_num = ARRAY_SIZE(gx09inx101_set_power_cmds),
        .read_power_cmds_num = ARRAY_SIZE(gx09inx101_read_power_cmds),
        .sleep_out_cmds_num = ARRAY_SIZE(gx09inx101_sleep_out_cmds),
        .sleep_in_cmds_num = ARRAY_SIZE(gx09inx101_sleep_in_cmds),
        //.drm_modeinfo = gx09inx101_modelist,
        .spacemit_modeinfo = gx09inx101_spacemit_modelist,
        .mipi_info = &gx09inx101_mipi_info,
        .set_id_cmds = gx09inx101_set_id_cmds,
        .read_id_cmds = gx09inx101_read_id_cmds,
        .set_power_cmds = gx09inx101_set_power_cmds,
        .read_power_cmds = gx09inx101_read_power_cmds,
        .init_cmds = gx09inx101_init_cmds,
        .sleep_out_cmds = gx09inx101_sleep_out_cmds,
        .sleep_in_cmds = gx09inx101_sleep_in_cmds,
        .bitclk_sel = 3,
        .bitclk_div = 1,
        .pxclk_sel = 2,
        .pxclk_div = 6,
};

int lcd_gx09inx101_init(void)
{
        int ret;

        ret = lcd_mipi_register_panel(&lcd_gx09inx101);
        return ret;
}
```
以下各项根据屏幕相关信息进行配置。\
**initial-cmd**：
```c
static struct spacemit_dsi_cmd_desc gx09inx101_init_cmds[] = {
        //8279 + INX10.1
        {SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   2, {0xB0,0x01}},
        {SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   2, {0xC3,0x4F}},
        {SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   2, {0xC4,0x40}},
        {SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   2, {0xC5,0x40}},
        {SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   2, {0xC6,0x40}},
        {SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   2, {0xC7,0x40}},
        {SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   2, {0xC8,0x4D}},
        {SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   2, {0xC9,0x52}},
        {SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   2, {0xCA,0x51}},
        {SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   2, {0xCD,0x5D}},
        {SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   2, {0xCE,0x5B}},
        {SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   2, {0xCF,0x4B}},
        {SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   2, {0xD0,0x49}},

		·······

        {SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   2, {0xDA,0x19}},
        {SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   2, {0xDB,0x17}},
        {SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   2, {0xDC,0x17}},
        {SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   2, {0xDD,0x18}},
        {SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   2, {0xDE,0x1A}},
        {SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   2, {0xDF,0x1E}},
        {SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   2, {0xE0,0x20}},
        {SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   2, {0xE1,0x23}},
        {SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 0,   2, {0xE2,0x07}},
        {SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE, 200, 2, {0x11, 0x00}},
        {SPACEMIT_DSI_DCS_LWRITE, SPACEMIT_DSI_LP_MODE,  50, 2, {0x29, 0x00}},
};
```
通过配置 **read id** 读取 **panel_id**\
lcd_gx09inx101读取0xfb寄存器
```c
//uboot-2022.10/drivers/video/spacemit/dsi/video/lcd/lcd_gx09inx101.c
static struct spacemit_dsi_cmd_desc gx09inx101_read_id_cmds[] = {
        {SPACEMIT_DSI_GENERIC_READ1, SPACEMIT_DSI_LP_MODE, UNLOCK_DELAY, 1, {0xfb}},
};
```
0xfb寄存器理应读取的到值为0x1
```c
//uboot-2022.10/drivers/video/spacemit/dsi/video/lcd/lcd_gx09inx101.c
struct lcd_mipi_panel_info lcd_gx09inx101 = {
        .lcd_name = "gx09inx101",
        .panel_id0 = 0x1,
```
通过读取**power_value**进行**esd check**\
lcd_gx09inx10读取0xA寄存器进行esd check
```
//uboot-2022.10/drivers/video/spacemit/dsi/video/lcd/lcd_gx09inx101.c
static struct spacemit_dsi_cmd_desc gx09inx101_read_power_cmds[] = {
        {SPACEMIT_DSI_GENERIC_READ1, SPACEMIT_DSI_HS_MODE, UNLOCK_DELAY, 1, {0xA}},
};
```
**power_value**：0xA寄存器理应读取到的值
```c
//uboot-2022.10/drivers/video/spacemit/dsi/video/lcd/lcd_gx09inx101.c
struct lcd_mipi_panel_info lcd_gx09inx101 = {

        .power_value = 0x10,
```

2. 修改Makefile
```c
//uboot-2022.10/drivers/video/spacemit/dsi/Makefile
# SPDX-License-Identifier: GPL-2.0

obj-y += video/spacemit_video_tx.o \
        video/spacemit_mipi_port.o \
        drv/spacemit_dphy.o \
        drv/spacemit_dsi_common.o \
        drv/spacemit_dsi_drv.o

obj-y += video/lcd/lcd_icnl9911c.o
obj-y += video/lcd/lcd_icnl9951r.o
obj-y += video/lcd/lcd_jd9365dah3.o
obj-y += video/lcd/lcd_gx09inx101.o
obj-y += video/lcd/lcd_lt8911ext_edp_1080p.o
```
3. 修改 spacemit_dsi_common.h
```c
//uboot-2022.10/drivers/video/spacemit/dsi/include/spacemit_dsi_common.h
int lcd_icnl9911c_init(void);
int lcd_icnl9951r_init(void);
int lcd_gx09inx101_init(void);    // 增加 lcd_gx09inx101.c实现的相应函数的声明
int lcd_jd9365dah3_init(void);
int lcd_lt8911ext_edp_1080p_init(void);
```
4. 修改 spacemit_mipi_port.c
```c
//uboot-2022.10/drivers/video/spacemit/dsi/video/spacemit_mipi_port.c

if (strcmp("lt8911ext_edp_1080p", priv->panel_name) == 0) {
        tx_device_client.panel_type = LCD_EDP;
        tx_device.panel_type = tx_device_client.panel_type;
        lcd_lt8911ext_edp_1080p_init();
    } else if(strcmp("icnl9951r", priv->panel_name) == 0) {
        tx_device_client.panel_type = LCD_MIPI;
        tx_device.panel_type = tx_device_client.panel_type;
        lcd_icnl9951r_init();
    } else if(strcmp("jd9365dah3", priv->panel_name) == 0) {
        tx_device_client.panel_type = LCD_MIPI;
        tx_device.panel_type = tx_device_client.panel_type;
        lcd_jd9365dah3_init();
    } else {
        // lcd_icnl9911c_init();
        lcd_gx09inx101_init();          //增加gx09inx101 panel识别，新增其它panel参考以上三个
    }
```
### 1.4. Uboot 启动相关 log
```c
[   0.842] Found device 'mipi@d421a800', disp_uc_priv=000000007deb1aa0
            /*   read id ,在lcd_gx09inx101.c 中
                 struct lcd_mipi_panel_info lcd_gx09inx101.id0 配置 */
[   1.001] read panel id OK: read value = 0x1, 0x0, 0x0
[   1.003] Panel is gx09inx101
[   1.260] fb=7f700000, size=1200x1920
```
## 二、Kernel 屏幕调试
spacemit平台Display模块的功能和使用方法参考：[spacemit平台display模块介绍](https://bianbu-linux.spacemit.com/device/peripheral_driver/Display)

### 2.1. HDMI 配置
以k1-x_deb1方案为例，方案配置HDMI。

```c
// linux-6.6\arch\riscv\boot\dts\spacemit\k1-x_deb1.dts
&dpu_online2_hdmi {
	memory-region = <&dpu_resv>;                    // 配置hdmi dpu预留内存
	status = "okay";
};

&hdmi{
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_hdmi_0>;
	status = "okay";
};
```

### 2.2. MIPI DSI panel 配置实例
kernel阶段，mipi屏幕配置步骤：
1. 配置供电以及GPIO。
2. 新建mipi屏的dtsi。
3. 根据屏幕供应商提供的mipi屏，主控芯片的datasheet、时序等信息，配置dtsi相应的clock（包含前后肩、分辨率以及计算得出的pix clock，bit clock）和initial、read id 等 command。
4. 将mipi panel与相应的方案关联。

以lcd_gx09inx101_mipi为例，k1-x_deb1方案采用lcd_gx09inx101_mipi作为显示屏幕，与mipi dsi相关的部分显示dts 如下：
#### 2.2.1. k1-x_deb1方案
```c
//linux-6.6/arch/riscv/boot/dts/spacemit/k1-x_deb1.dts
#include "lcd/lcd_gx09inx101_mipi.dtsi"
#include "k1-x-lcd.dtsi"
#include "k1-x-hdmi.dtsi"

&dpu_online2_dsi {
	memory-region = <&dpu_resv>;
	spacemit-dpu-bitclk = <1000000000>;     //bit clock，参考时序计算
	spacemit-dpu-escclk = <76800000>;
	dsi_1v2-supply = <&ldo_5>;              // 引用PMIC DLDO1
	vin-supply-names = "dsi_1v2";           // 配置MIPI DSI 1.2v
	status = "okay";
};

&dsi2 {
        status = "okay";

        panel2: panel2@0 {
                status = "okay";
                compatible = "spacemit,mipi-panel2";
                reg = <0>;

                gpios-reset = <81>;             // 配置panel 复位 gpio
                gpios-dc = <82 83>;             // 配置panel 电源控制 gpio
                id = <2>;                       // 配置 panel id
                delay-after-reset = <10>;       // 配置 plane 复位延时时间（单位：ms）
                force-attached = "lcd_gx09inx101_mipi"; //方案与 panel相关联
        };
};

&lcds {
        status = "okay";                        // 使能lcds
};

&pwm14 {                                        // 配置pwm
        pinctrl-names = "default";
        pinctrl-0 = <&pinctrl_pwm14_1>;
        status = "okay";
};

&pwm_bl {                                       // 配置背光
        pwms = <&pwm14 2000>;
        brightness-levels = <
                0   20  20  20  21  21  21  22  22  22  23  23  23  24  24  24
                25  25  25  26  26  26  27  27  27  28  28  29  29  30  30  31
                32  33  34  35  36  37  38  39  40  41  42  43  44  45  46  47
                48  49  50  51  52  53  54  55  56  57  58  59  60  61  62  63
                64  65  66  67  68  69  70  71  72  73  74  75  76  77  78  79
                80  81  82  83  84  85  86  87  88  89  90  91  92  93  94  95
                96  97  98  99  100 101 102 103 104 105 106 107 108 109 110 111
                112 113 114 115 116 117 118 119 120 121 122 123 124 125 126 127
                128 129 130 131 132 133 134 135 136 137 138 139 140 141 142 143
                144 145 146 147 148 149 150 151 152 153 154 155 156 157 158 159
                160 161 162 163 164 165 166 167 168 169 170 171 172 173 174 175
                176 177 178 179 180 181 182 183 184 185 186 187 188 189 190 191
                192 193 194 195 196 197 198 199 200 201 202 203 204 205 206 207
                208 209 210 211 212 213 214 215 216 217 218 219 220 221 222 223
                224 225 226 227 228 229 230 231 232 233 234 235 236 237 238 239
                240 241 242 243 244 245 246 247 248 249 250 251 252 253 254 255
        >;
        default-brightness-level = <50>;
        status = "okay";
};
```
##### DSI供电
DSI 需要 AVDD1.8 和 AVDD1.2 供电
  - AVDD18_DSI由BUCK3_1V8默认供电，不需要配置。
  - AVDD12_DSI由DLDO1_1V1供电。
```c
dsi_1v2-supply = <&ldo_5>;              //dldo1
vin-supply-names = "dsi_1v2"
```
##### Mipi屏供电
本方案mipi接口根据原理图需要配置GPIO和pwm。
###### GPIO
需要控制的GPIO有：\
        dcp-gpios = gpio 82;\
        dcn-gpios = gpio 83;\
        reset-gpios = gpio 81;
```c
gpios-reset = <81>;             // 配置panel 复位 gpio
gpios-dc = <82 83>;             // 配置panel 电源控制 gpio
```
###### pwm
采用pwm控制背光，对应设备树
```c
&pwm14 {
        pinctrl-names = "default";
        pinctrl-0 = <&pinctrl_pwm14_1>;
        status = "okay";
};
```
#### 2.2.2. lcd dts 配置
在linux-6.6/arch/riscv/boot/dts/spacemit/lcd/路径新建lcd_gx09inx101_mipi.dtsi
```c
// SPDX-License-Identifier: GPL-2.0

/ { lcds: lcds {
        lcd_gx09inx101_mipi: lcd_gx09inx101_mipi {
                dsi-work-mode = <1>;            // panel中配置 mipi dsi工作模式：1 DSI_MODE_VIDEO_BURST;
                dsi-lane-number = <4>;          // panel中配置mipi dsi lane数量
                dsi-color-format = "rgb888";
                width-mm = <142>;               //panel中配置屏幕active area
                height-mm = <228>;              // panel中配置屏幕active area
                use-dcs-write;                  // panel中配置是否使用dcs命令模式

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
                phy-bit-clock = <1000000000>;    // mipi dsi dphy bitclk 配置
                phy-esc-clock = <76800000>;
                split-enable = <0>;              // mipi dsi dphy中配置mipi dsi使能
                eotp-enable = <0>;               // mipi dsi dphy中配置mipi dsi使能eotp
                burst-mode = <2>;                // mipi dsi dphy中配置mipi dsi burst mode: 2 DSI_BURST_MODE_BURST;
                esd-check-enable = <0>;          // panel中配置使能esd check
                /* DSI_CMD, DSI_MODE, timeout, len, cmd */
                // initial-command,根据屏幕信息配置
                initial-command = [
                39 01 00 02 B0 01
                39 01 00 02 C3 4F
                39 01 00 02 C4 40
                39 01 00 02 C5 40
                39 01 00 02 C6 40
                39 01 00 02 C7 40
                39 01 00 02 C8 4D

                    ··········

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
                // read-id-command,根据屏幕信息配置
                read-id-command = [
                        37 01 00 01 05
                        14 01 00 05 fb fc fd fe ff
                ];

                display-timings {
                        timing0 {
                                clock-frequency = <153000000>;  // mipi dsi pixclk 配置
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
##### 时序计算
pixel clock= (hactive + hfp + hbp + hsync) * (vactive + vfp + vbp + vsync) * fps = （1200 + 50 + 40 + 10）* (1920 + 20 + 16 + 4) * 60 = 152880000 HZ\
bit clock = (((hactive + hfp + hbp + hsync) * (vactive + vfp + vbp + vsync) * fps  * bpp) / lane bumber) * 1.1 = (（（1200 + 50 + 40 + 10）* (1920 + 20 + 16 + 4) * 60 * 24）/ 4) * 1.1 = 1009008000 HZ\
通过display timing计算，pixel clock值为152880000 HZ，系统可配置为153000000 HZ，bit clock值为1009008000 HZ，系统可配置为1000000000 HZ。
dts文件中clock-frequency配置为153000000, spacemit-dpu-bitclk和phy-bit-clock配置为1000000000。
## FAQ
### 1. 只配置hdmi
以 k1-deb1 方案为例，驱动dts默认关闭 lcd，打开 hdmi，配置如下：\
默认配置
```c
//linux-6.6/arch/riscv/boot/dts/spacemit/k1-x_deb1.dts
&dpu_online2_dsi {
	memory-region = <&dpu_resv>;
	spacemit-dpu-bitclk = <1000000000>;
	spacemit-dpu-escclk = <76800000>;
	dsi_1v2-supply = <&ldo_5>;
	vin-supply-names = "dsi_1v2";
	status = "disabled";
};

&dsi2 {
	status = "disabled";

	panel2: panel2@0 {
		status = "ok";
		compatible = "spacemit,mipi-panel2";
		reg = <0>;

		gpios-reset = <81>;
		gpios-dc = <82 83>;
		id = <2>;
		delay-after-reset = <10>;
		force-attached = "lcd_gx09inx101_mipi";
	};
};

&lcds {
	status = "disabled";
};

&dpu_online2_hdmi {
	memory-region = <&dpu_resv>;
	status = "okay";
};

&hdmi{
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_hdmi_0>;
	status = "okay";
};
```
### 2. 同时配置hdmi、dsi
若需同时打开 lcd 和 hdmi，则需要把两个设备相关dts的状态都配置为"okay"
```c
//linux-6.6/arch/riscv/boot/dts/spacemit/k1-x_deb1.dts
&dpu_online2_dsi {
	memory-region = <&dpu_resv>;
	spacemit-dpu-bitclk = <1000000000>;
	spacemit-dpu-escclk = <76800000>;
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

		gpios-reset = <81>;
		gpios-dc = <82 83>;
		id = <2>;
		delay-after-reset = <10>;
		force-attached = "lcd_gx09inx101_mipi";
	};
};

&lcds {
	status = "okay";
};

&dpu_online2_hdmi {
	memory-region = <&dpu_resv>;
	status = "okay";
};

&hdmi{
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_hdmi_0>;
	status = "okay";
};
```
### 3. 只配置dsi
```c
//linux-6.6/arch/riscv/boot/dts/spacemit/k1-x_deb1.dts
&dpu_online2_dsi {
	memory-region = <&dpu_resv>;
	spacemit-dpu-bitclk = <1000000000>;
	spacemit-dpu-escclk = <76800000>;
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

		gpios-reset = <81>;
		gpios-dc = <82 83>;
		id = <2>;
		delay-after-reset = <10>;
		force-attached = "lcd_gx09inx101_mipi";
	};
};

&lcds {
	status = "okay";
};

&dpu_online2_hdmi {
	memory-region = <&dpu_resv>;
	status = "disabled";
};

&hdmi{
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_hdmi_0>;
	status = "disabled";
};
```
### 4. weston 主屏配置
weston 显示脚本
```c
//buildroot/package/weston/run_weston.sh
#export EGL_LOG_LEVEL=debug
export MESA_LOADER_DRIVER_OVERRIDE=pvr
export XDG_RUNTIME_DIR=/root
export QT_QPA_PLATFORM_PLUGIN_PATH=/usr/lib/qt/plugins/platforms
export QT_QPA_PLATFORM=wayland
weston --log=/var/log/weston --tty=1 --idle-time=0
```
在hdmi和lcd同时配置为“okay”时，该脚本会使用第一个drm显示设备显示。\
查看drm设备命令:
```bash
cd /sys/class/drm
ls
```
lcd默认配置为card1，hdmi为card2。若需weston界面默认在hdmi上显示，则需修改run_weston.sh脚本
```c
//buildroot/package/weston/run_weston.sh
#export EGL_LOG_LEVEL=debug
export MESA_LOADER_DRIVER_OVERRIDE=pvr
export XDG_RUNTIME_DIR=/root
export QT_QPA_PLATFORM_PLUGIN_PATH=/usr/lib/qt/plugins/platforms
export QT_QPA_PLATFORM=wayland
weston --log=/var/log/weston --tty=1 --drm-device=card2 --idle-time=0
```
### 5. bianbu-desktop 主屏配置
mutter 应用默认将dsi作为主显示器，若需修改为hdmi，则需修改mutter包源码并重新编译。
```c
//mutter/src/backends/meta-monitor.c
meta_monitor_is_laptop_panel (MetaMonitor *monitor)
{
  const MetaOutputInfo *output_info =
    meta_monitor_get_main_output_info (monitor);

  switch (output_info->connector_type)
    {
    case META_CONNECTOR_TYPE_eDP:
    case META_CONNECTOR_TYPE_LVDS:
    case META_CONNECTOR_TYPE_HDMIA: //默认为 META_CONNECTOR_TYPE_DSI
      return TRUE;
    default:
      return FALSE;
    }
}
```