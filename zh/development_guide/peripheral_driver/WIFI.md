# WIFI
介绍WIFI的移植和使用方法。
## 模块介绍
K1平台上主要通过外部WIFI模块来实现WIFI功能，主要支持PCIE，SDIO以及USB等接口的模块。
### 功能介绍
![](static/wlan.png)

WIFI框架图可以分为以下几个层次：  

### 源码结构介绍

WIFI相关的源码可以分为三个部分：
1. WIFI驱动，由WIFI厂商提供，主要实现WIFI功能。
2. 平台相关部分，主要实现模组供电以及使能等相关接口，供WIFI驱动调用。
3. 接口驱动，主要实现WIFI数据传输接口功能，如PCIE，SDIO以及USB等接口。

WIFI驱动的源码一般放到以下目录：
```
drivers/net/wireless
|-- aic8800             #aic厂商驱动
|-- realtek             #realtek厂商驱动
    |-- rtl8852be       #rtl8852be
    |-- rtl8852bs       #rtl8852bs
|-- wuqi                #wuqi厂商驱动
```
平台相关的源码：
```
drivers/soc/spacemit/spacemit-rf
|-- spacemit-pwrseq.c   #WIFI和蓝牙等公共部分实现
|-- spacemit-wlan.c     #WIFI供电，gpio以及时钟相关接口实现
|-- spacemit-bt.c       #bt供电，gpio以及时钟相关接口实现
```
接口相关的源码参考各个接口驱动说明文档。

## 关键特性
### sdio接口特性
| 特性 | 特性说明 |
| :-----| :----|
| 兼容SDIO v4.10 | 兼容4bit SDIO 4.10规范 |
| 支持SD 3.0模式 | 支持SDR12/SDR25/DDR50/SDR50/SDR104模式 |
| 支持PIO/DMA | 支持PIO,SDMA,ADMA,ADMA2传输模式 |

### 性能参数
| 模组型号 | TX(Mb/s) | RX(Mb/s) |
| :-----| :----| :----: |
| rtl8852bs | 460 | 480 |
| aic8800d80 | 410 | 470 |

测试方法

```
同一局域网段
服务端：iperf3 -s
客户端：iperf3 -c 192.168.1.xxx -t 72000
```

## 配置介绍
主要包括驱动使能配置和dts配置
### CONFIG配置
CONFIG_SPACEMIT_RFKILL 为WIFI模组提供平台相关支持，默认情况，此选项为Y
```
Device Drivers
        SOC (System On Chip) specific Drivers
                Spacemit rfkill driver (SPACEMIT_RFKILL [=y])
```


### dts配置
#### sdio pinctrl

方案上一般 slot2 用于 sdio，对应 pinctrl_mmc2。

```
pinctrl_mmc2: mmc2_grp {
        pinctrl-single,pins =<
                K1X_PADCONF(GPIO_15, MUX_MODE1, (EDGE_NONE | PULL_UP | PAD_1V8_DS2))	/* mmc2_data3 */
                K1X_PADCONF(GPIO_16, MUX_MODE1, (EDGE_NONE | PULL_UP | PAD_1V8_DS2))	/* mmc2_data2 */
                K1X_PADCONF(GPIO_17, MUX_MODE1, (EDGE_NONE | PULL_UP | PAD_1V8_DS2))	/* mmc2_data1 */
                K1X_PADCONF(GPIO_18, MUX_MODE1, (EDGE_NONE | PULL_UP | PAD_1V8_DS2))	/* mmc2_data0 */
                K1X_PADCONF(GPIO_19, MUX_MODE1, (EDGE_NONE | PULL_UP | PAD_1V8_DS2))	/* mmc2_cmd */
                K1X_PADCONF(GPIO_20, MUX_MODE1, (EDGE_NONE | PULL_UP | PAD_1V8_DS2))	/* mmc2_clk */
        >;
};
```

如果需要支持WIFI唤醒，需要将wlan_hostwake配置为pinctl模式：
```
pinctrl_wlan_wakeup: wlan_wakeup_grp {
        pinctrl-single,pins =<
                K1X_PADCONF(GPIO_66, MUX_MODE0, (EDGE_FALL | PULL_DOWN | PAD_3V_DS2))   /* wifi edge detect */
        >;
};
```

#### 电源配置

sdio 需要配置两个电源，分别是 vmmc-supply 和 vqmmc-supply，分别对应卡的功能和 io 供电，vqmmc-supply 建议1.8v，具体根据sdio卡的模式选择实际电压。


```
&sdhci1 {
        vmmc-supply = <&dcdc_3>;
        vqmmc-supply = <&ldo_1>;
};
```

#### tuning 配置

sdio 跑高速模式下需要进行 tuning，不同的硬件版型都需要调整 tx 的相关参数。

#### sdio dts 配置示例

sdio 的完整方案配置如下：

```
/* SDIO */
&sdhci1 {
        pinctrl-names = "default";
        pinctrl-0 = <&pinctrl_mmc2>;
        bus-width = <4>;
        non-removable;
        vqmmc-supply = <&dcdc_3>;
        no-mmc;
        no-sd;
        keep-power-in-suspend;

        spacemit,sdh-host-caps-disable = <(
                        MMC_CAP_UHS_DDR50 |
                        MMC_CAP_NEEDS_POLL
                        )>;
        spacemit,sdh-quirks = <(
                        SDHCI_QUIRK_BROKEN_CARD_DETECTION |
                        SDHCI_QUIRK_BROKEN_TIMEOUT_VAL
                        )>;
        spacemit,sdh-quirks2 = <(
                        SDHCI_QUIRK2_PRESET_VALUE_BROKEN |
                        SDHCI_QUIRK2_BROKEN_PHY_MODULE
                        )>;
        spacemit,rx_dline_reg = <0x0>;
        spacemit,tx_delaycode = <0x7f>;
        spacemit,rx_tuning_limit = <50>;
        spacemit,sdh-freq = <375000000>;
        status = "okay";
};
```
sdio tx_delaycode 的默认值是0x7f，实际上需要根据具体的方案走线等差异进行调整。

#### 平台部分dts配置

平台完整方案配置如下：
```
rf_pwrseq: rf-pwrseq {
        compatible = "spacemit,rf-pwrseq";
        //vdd-supply = <&ldo_7>;
        //vdd_voltage = <3300000>;
        io-supply = <&dcdc_3>;
        io_voltage = <1800000>;
        pwr-gpios  = <&gpio 67 0>;
        status = "okay";

        wlan_pwrseq: wlan-pwrseq {
                compatible = "spacemit,wlan-pwrseq";
                regon-gpios = <&gpio 116 0>;
                interrupt-parent = <&pinctrl>;
                interrupts = <268>;
                pinctrl-names = "default";
                pinctrl-0 = <&pinctrl_wlan_wakeup>;
        };

        bt_pwrseq: bt-pwrseq {
                compatible = "spacemit,bt-pwrseq";
                reset-gpios     = <&gpio 63 0>;
        };
};
```

目前市面上很多模组都是WIFI和蓝牙二合一，WIFI和蓝牙的供电部分很多都是共用的，共用的部分建议放到rf_pwrseq中进行配置，仅影响WIFI的部分，放到wlan_pwrseq中进行配置。

单WIFI模组只需要配置wlan_pwrseq即可，不需要配置rf_pwrseq，但要使能rf_pwrseq节点。

在打开WIFI电源时会先使能共用部分的电源以及gpio状态，平台会维护对应的引用计数，关闭时只有WIFI和蓝牙都关闭后才会真正关闭共用部分的电源以及gpio状态。

rf_pwrseq：
- vdd-supply是配置模组的供电，具体按实际硬件配置。
- vdd_voltage用于设定模组供电的电压。
- io-supply是配置模组io的供电，具体按实际硬件配置。
- io_voltage用于设定模组io供电的电压。
- pwr-gpios是模组使能脚，配置后会默认拉高，支持多个gpio的配置。
- clock是模组共用的时钟配置。
- power-on-delay-ms是设置模组上电后的延时，默认是100ms。

wlan_pwrseq：
- regon-gpios是WIFI部分的使能脚，调用spacemit_wlan_set_power(1)接口时会拉高。
- interrupts配置WIFI的中断唤醒脚，表示采用pinctl的方式进行唤醒。
- power-on-delay-ms是设置WIFI上电后的延时，默认是10ms。

bt_pwrseq：
- reset-gpios是蓝牙的使能脚，使能蓝牙对应的rfkill时会将该gpio拉高。
- clock是蓝牙的时钟配置。
- power-on-delay-ms是设置蓝牙上电后的延时，默认是10ms。

## 接口描述
### WIFI基本功能测试接口介绍

首先确认wpa_supplicant服务有正常运行。
```
wpa_supplicant -iwlan0 -Dnl80211 -c/wpa_supplicant.conf -B
```
wpa_supplicant.conf参考配置如下：
```
ctrl_interface=/var/run/wpa_supplicant
wowlan_triggers=any
```
wowlan_triggers是wow相关的配置，支持WIFI唤醒需要配置。

使用wpa_cli与wpa_supplicant服务进行交互，wpa_supplicant.conf的ctrl_interface如果不是默认的/var/run/wpa_supplicant，则wpa_cli运行时需要使用-p进行显式的指定。
```
 wpa_cli -iwlan0 -p/var/run/wpa_supplicant
 scan
 scan_results
```
正常扫描会有类似如下输出：
```
bssid / frequency / signal level / flags / ssid
f6:12:b3:d4:65:ef       2462    -37     [WPA2-PSK-CCMP][WPS][ESS][P2P]  wilson
78:85:f4:82:01:3c       2462    -66     [WPA2-PSK-CCMP][WPS][ESS]       HUAWEI-LX45AG_HiLink
02:0e:5e:76:a5:6e       2412    -69     [WPA-PSK-CCMP+TKIP][ESS]        ChinaNet-1mMr
30:8e:7a:2f:64:8c       2437    -69     [WPA-PSK-CCMP+TKIP][WPA2-PSK-CCMP+TKIP][ESS]    K03_1tlftb
dc:16:b2:57:9e:65       2437    -78     [WPA2-PSK-CCMP][ESS]    \x00\x00\x00\x00\x00\x00\x00\x00
dc:16:b2:57:9e:60       2437    -78     [WPA-PSK-CCMP][WPA2-PSK-CCMP][WPS][ESS] TK-ZJB
48:0e:ec:ad:52:4d       2462    -78     [WPA-PSK-CCMP][WPA2-PSK-CCMP][WPS][ESS] TP-LINK_524D
3c:d2:e5:c6:08:9b       2452    -83     [WPA2-PSK-CCMP][ESS]
3e:d2:e5:16:08:9b       2452    -83     [WPA-PSK-CCMP+TKIP][WPA2-PSK-CCMP+TKIP][ESS]    young
80:ea:07:dc:f2:be       2462    -88     [WPA-PSK-CCMP][WPA2-PSK-CCMP][ESS]      HZXF
9a:00:74:84:d1:60       2412    -85     [WPA-PSK-CCMP+TKIP][WPA2-PSK-CCMP+TKIP][ESS]   ChinaNet-ieR7
dc:f8:b9:46:ec:30       2472    -85     [WPA-PSK-CCMP+TKIP][WPA2-PSK-CCMP+TKIP][ESS]   ChinaNet-MiZK
```
选择需要连接的ap网络进行连接
```
> add_network
0
> set_network 0 ssid "wilson"
OK
> set_network 0 key_mgmt WPA-PSK
OK
> set_network 0 psk "wilson2001"
OK
> enable_network 0
```
### API介绍
平台部分的接口，包括WIFI上下电，获取中断以及sdio扫描等接口。
- void spacemit_wlan_set_power(bool on_off)

   设置WIFI电源，0为关闭，1为打开
- int spacemit_wlan_get_oob_irq(void);

   获取平台irq中断号
- void spacemit_sdio_detect_change(int enable_scan);

   扫描sdio总线

### Debug介绍
#### sysfs

```
sdio的tx_delaycode的值默认是在方案dts中指定，可以通过sysfs下的该节点进行动态修改，方便调试阶段的验证。
echo 0x7f > /sys/devices/platform/soc/d4280800.sdh
动态修改必须要在wifi驱动加载之前才能生效。
```
#### debugfs
```
常用于查询sdio的工作状态，包括频率，位宽，模式等信息。
cat /sys/kernel/debug/mmc1/ios
clock:          204800000 Hz
actual clock:   187500000 Hz
vdd:            21 (3.3 ~ 3.4 V)
bus mode:       2 (push-pull)
chip select:    0 (don't care)
power mode:     2 (on)
bus width:      2 (4 bits)
timing spec:    6 (sd uhs SDR104)
signal voltage: 1 (1.80 V)
driver type:    0 (driver type B)
```
## FAQ
问题1

现象：能检测到sdio设备，但是初始化失败。

打印：
```
mmc1: error -110 whilst initialising SDIO card
```
解决办法
1. 修改sdio的tx_delaycode参数进行验证。

问题2

现象：sdio读写报错。
```
[ 23.662558] rtl8852bs mmc1:0001:1: rtw_sdio_raw_read: sdio read failed (-110)
[ 23.669829] rtl8852bs mmc1:0001:1: RTW_SDI0: READ use CMD53
[ 23.675507] rtl8852bs mmc1:0001:1: RTW_SDIO: READ from 0x00270, 4 bytes
[ 23.682193] RTW SDIO: READ 0000000: 63 00 00 81
```
解决办法：
1. 修改sdio的tx_delaycode参数进行验证。
2. 方法1无效后可以尝试降频验证，在sdh-host-caps-disable中增加MMC_CAP_UHS_SDR104，禁掉SDR104模式。
