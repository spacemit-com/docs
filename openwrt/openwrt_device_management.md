# 设备管理
本文档介绍如何通过EEPROM实现自适应设备(板型)，包括如何添加新板型支持。对于OpenWrt，设备一般是挂靠在具体某个方案下，如k1-x_MUSE-Pi.dts在sbc方案下。

后续文档描述的设备和板型等效




# 添加新设备
以下以新增k1-x_MUSE-Pi板型为例说明

## 内核设备树


拷贝k1-x_deb1.dts到如下文件，修改相应内容

```
target/linux/spacemit/dts/k1-x_MUSE-Pi.dts

```

对于新增的版型，dts文件均存放在以上目录。编译的时候打包进bootfs分区，uboot启动kernel前加载设备树文件。

## 方案配置
如是开发板形态，则修改openwrt/target/linux/spacemit/image/k1-sbc.mk，添加k1-x_MUSE-Pi板型支持

```
DEVICE_DTS := k1-x_deb1 k1-x_MUSE-Pi

```
如果需要新增方案，请参考[方案管理](openwrt_solution_management.md)


## uboot修改
新增板型需要在uboot源码仓库修改，如下：

1. 添加uboot设备树u-boot-2022.10/arch/riscv/dts/k1-x_MUSE-Pi.dts
2. 修改Makefile，将k1-x_MUSE-Pi.dtb添加到里面，如下：(注意后缀改成dtb)

```
 11 dtb-$(CONFIG_TARGET_SPACEMIT_K1X) += k1-x_evb.dtb k1-x_deb2.dtb k1-x_deb1.dtb k1-x_hs450.dtb \
 12                      k1-x_kx312.dtb k1-x_MINI-PC.dtb k1-x_mingo.dtb k1-x_MUSE-N1.dtb \
 13                      k1-x_MUSE-Pi.dtb k1-x_spl.dtb k1-x_milkv-jupiter.dtb \
 14                      k1-x_MUSE-Book.dtb
 15 

```

3. 修改uboot-2022.10/board/spacemit/k1-x/configs/uboot_fdt.its，添加新节点。

```
@@ -46,15 +46,6 @@
 				algo = "crc32";
 			};
 		};
+		fdt_4 {
+                        description = "k1-x_MUSE-Pi";
+                        type = "flat_dt";
+                        compression = "none";
+                        data = /incbin/("../dtb/1-x_MUSE-Pi.dtb");
+                        hash-1 {
+                                algo = "crc32";
+                        };
+                };
 	};
 
 	configurations {
@@ -74,10 +65,5 @@
 			loadables = "uboot";
 			fdt = "fdt_3";
 		};
+		conf_4 {
+                        description = "k1-x_MUSE-Pi";
+                        loadables = "uboot";
+                        fdt = "fdt_4";
+                };
 	};
 };

```

4. 修改uboot-2022.10/include/configs/k1-x.h，更新DEFAULT_PRODUCT_NAME，FSBL和u-boot将默认根据DEFAULT_PRODUCT_NAME加载dtb。如果板子有EEPROM记录product_name等信息，则优先级更高，FSBL和u-boot将使用EEPROM的信息实现自适应启动。

```
@@ -25,7 +25,7 @@
 #define CONFIG_GATEWAYIP	10.0.92.1
 #define CONFIG_NETMASK		255.255.255.0

-#define DEFAULT_PRODUCT_NAME	"k1_deb1"
+#define DEFAULT_PRODUCT_NAME	"k1-x_MUSE-Pi"

 #define K1X_SPL_BOOT_LOAD_ADDR	(0x20200000)
 #define DDR_TRAINING_DATA_BASE	(0xc0829000)
```

## EEPROM写号
通过进迭Titanflasher工具给板子eeprom写号，实现启动的自适应，即一个固件支持多个板子。系统启动时读取eeprom的product_name字段，匹配uboot_fdt.its中定义的每个conf的description字段，从而实现多板型支持。



