# PCIe

介绍PCIe的功能和使用方法。

## 模块介绍

PCIe 是一种高速串行计算机扩展总线标准。采用高速串行点对点双通道高带宽传输，所连接设备独享通道带宽。k1 共有3个PCIe控制器, 支持外接各种PCIe接口设备，包括nvme ssd, sata和wifi等。
PCIe0和USB3控制器共用一个phy硬件, 不能同时使用。应用方案上一般使用USB3, 故PCIe0较少使用。

### 功能介绍

![](static/linux_pcie.png)

Linux PCIe子系统框架包括三部分: PCIe core、PCIe控制器驱动和PCIe设备驱动。各部分主要功能如下:  
1.PCIe核心

- PCIe总线的枚举, 资源分配，中断
- PCIe设备添加和删除
- PCIe设备驱动注册与注销  

2.PCIe控制器驱动

- 对PCIe主机控制器进行操作  

3.PCIe设备驱动

- PCIe设备驱动，如GPU、NIC和NVMe等

### 源码结构介绍

控制器驱动代码在drivers/pci/controller/dwc目录下：

```
|-- pcie-designware.c           #dwc pcie驱动公共代码
|-- pcie-designware-host.c      #dwc pcie主机驱动代码
|-- pcie-k1x.c                  #k1x pcie控制器驱动
```

## 关键特性

### 特性

| 特性 | 特性说明 |
| :-----| :----|
| 支持模式 | 支持RC模式 |
| 支持协议和lane数 | 支持gen2x1, gen2x2 |
| 支持设备 | 支持NVMe ssd、PCIe转sata、PCIe 网卡和PCIE显卡 |

### 性能参数

| ssd型号 | 顺序读(MB/s) | 顺序写(MB/s)  |
| :-----| :----| :----: |
| LRC20Z500GCB | 829 | 812 |  

测试方法

```
# 测试读
fio --name read --eta-newline=5s --filename=/dev/nvme0n1 --rw=read --size=2g --io_size=10g --blocksize=1024k --ioengine=libaio --fsync=10000 --iodepth=32 --direct=1 --numjobs=1 --runtime=60 --group_reporting  

#测试写
fio --name write --eta-newline=5s --filename=/dev/nvme0n1 --rw=write --size=2g
 --io_size=60g --blocksize=1024k --ioengine=libaio --fsync=10000 --iodepth=32 --
direct=1 --numjobs=1 --runtime=60 --group_reporting  
```

## 配置介绍

主要包括驱动使能配置和dts配置

### CONFIG配置

CONFIG_PCI  为PCI和PCIe总线协议提供支持，默认情况，此选项为Y

```
Device Drivers
    PCI support (PCI [=y])
```

PCI_K1X_HOST 为K1 PCIe控制器驱动提供支持，默认情况下，此选型为Y

```
Device Drivers
    PCI support (PCI [=y])
        PCI controller drivers
            DesignWare-based PCIe controllers
                Spacemit K1X PCIe Controller - Host Mode (PCI_K1X_HOST [=y])
```

### dts配置

#### 配置空间分配

各控制器配置空间分配如下。

1. PCIe0  
mem 空间大小  240MB  
io  空间大小  1MB
2. PCIe1  
mem 空间大小  240MB  
io  空间大小  1MB
3. PCIe2  
mem 可预取空间大小  256MB  
mem 不可预取空间大小  112MB
io  空间大小  1MB

##### 配置空间详细说明

以PCIe2控制器为例，说明PCIe控制器的地址空间分配。
PCIe2 分配地址空间为 0xa00000000到0xb80000000, 大小0x18000000(384MB)。
可以根据需要，对mem、IO和config空间进行修改，只需保证在0xa00000000到0xb80000000范围，且三部分空间不重合。

当前k1-x.dts中配置

```c
pcie2_rc: pcie@ca800000 {
        ...
        reg = <0x0 0xca800000 0x0 0x00001000>, /* dbi */
              ...
              <0x0 0xb7000000 0x0 0x00002000>, /* config space */
              ...
        #address-cells = <3>;
 #size-cells = <2>;
 ranges = <0x01000000 0x0 0xb7002000 0 0xb7002000 0x0 0x100000>,    
             <0x42000000 0x0 0xa0000000 0 0xa0000000 0x0 0x10000000>, 
             <0x02000000 0x0 0xb0000000 0 0xb0000000 0x0 0x7000000>;
        ...
}
```

含义如下

- mem 空间  

```
  <0x42000000 0x0 0xa0000000 0 0xa0000000 0x0 0x10000000>   
  mem prefetchable 空间 大小256MB  
  <0x02000000 0x0 0xb0000000 0 0xb0000000 0x0 0x7000000>  
  mem non-prefetchable空间  大小112MB  
```

- IO  空间  

```
  <0x01000000 0x0 0xb7002000 0 0xb7002000 0x0 0x100000>  
  大小为1MB  
```

- config 空间  

```
  <0x0 0xb7000000 0x0 0x00002000>  
  大小为8kB  
```

#### pinctrl

查看方案原理图，找到 pcie 使用的 pin 组。参考pinctrl章节，确定 pcie 使用的 pin 组。

假设 pcie2 可以直接采用 k1-x_pinctrl.dtsi 中定义 pinctrl_pcie2_4。

dts 配置

方案 dts 中 pcie_rc 描述如下。

```c
&pcie2_rc {
        pinctrl-names = "default";
        pinctrl-0 = <&pinctrl_pcie2_4>;
        status = "okay";
};
```

#### 完整的pcie dts

pcie2_rc模式控制器dts如下

```
pcie2_rc: pcie@ca800000 {
   compatible = "k1x,dwc-pcie";
   reg = <0x0 0xca800000 0x0 0x00001000>, /* dbi */
         <0x0 0xcab00000 0x0 0x0001ff24>, /* atu registers */
         <0x0 0xb7000000 0x0 0x00002000>, /* config space */
         <0x0 0xd4282bdc 0x0 0x00000008>, /* k1x soc config addr */
         <0x0 0xc0d20000 0x0 0x00001000>, /* phy ahb */
         <0x0 0xc0d10000 0x0 0x00001000>, /* phy addr */
         <0x0 0xd4282bcc 0x0 0x00000008>, /* conf0 addr */
         <0x0 0xc0b10000 0x0 0x00001000>; /* phy0 addr */
   reg-names = "dbi", "atu", "config", "k1x_conf", "phy_ahb", "phy_addr", "conf0_addr", "phy0_addr";

   k1x,pcie-port = <2>;
   clocks = <&ccu CLK_PCIE2>;
   clock-names = "pcie-clk";
   resets = <&reset RESET_PCIE2>;
   reset-names = "pcie-reset";
   power-domains = <&power K1X_PMU_BUS_PWR_DOMAIN>;

   bus-range = <0x00 0xff>;
   max-link-speed = <2>;
   num-lanes = <2>;
   num-viewport = <8>;
   device_type = "pci";
   #address-cells = <3>;
   #size-cells = <2>;
   ranges = <0x01000000 0x0 0xb7002000 0 0xb7002000 0x0 0x100000>,
     <0x42000000 0x0 0xa0000000 0 0xa0000000 0x0 0x10000000>,
     <0x02000000 0x0 0xb1000000 0 0xb1000000 0x0 0x07000000>;
   interconnects = <&dram_range2>;
   interconnect-names = "dma-mem";

   interrupts = <143>, <147>;
   interrupt-parent = <&intc>;
   #interrupt-cells = <1>;
   interrupt-map-mask = <0 0 0 0x7>;
   interrupt-map = <0000 0 0 1 &pcie2_intc 1>, /* int_a */
     <0000 0 0 2 &pcie2_intc 2>, /* int_b */
     <0000 0 0 3 &pcie2_intc 3>, /* int_c */
     <0000 0 0 4 &pcie2_intc 4>; /* int_d */
   linux,pci-domain = <2>;

   pcie2_intc: interrupt-controller@0 {
    interrupt-controller;
    reg = <0 0 0 0 0>;
    #address-cells = <0>;
    #interrupt-cells = <1>;
   };

      pinctrl-names = "default";
     pinctrl-0 = <&pinctrl_pcie2_4>;
 
      status = "okay";
};
```

## 接口介绍

### API介绍

注册pci设备驱动

```
/* Proper probing supporting hot-pluggable devices */
int __must_check __pci_register_driver(struct pci_driver *, struct module *,
                       const char *mod_name);

/* pci_register_driver() must be a macro so KBUILD_MODNAME can be expanded */
#define pci_register_driver(driver)     \
    __pci_register_driver(driver, THIS_MODULE, KBUILD_MODNAME)
```

注销pci设备驱动

```
void pci_unregister_driver(struct pci_driver *dev);
```

## Debug介绍

### sysfs

/sys/bus/pci  查看系统spi总线设备和驱动信息

```
|-- devices                 //pci总线上的设备
|-- drivers                 //pci总线上注册的设备驱动
|-- drivers_autoprobe
|-- drivers_probe
|-- rescan
|-- resource_alignment
|-- slots
`-- uevent
```

## 测试介绍

1.查看pci总线拓扑信息

```
#lspci
0001:00:00.0 PCI bridge: Device 201f:0001 (rev 01)
0001:01:00.0 Network controller: Realtek Semiconductor Co., Ltd. RTL8852BE PCIe 802.11ax Wireless Network Controller
0002:00:00.0 PCI bridge: Device 201f:0001 (rev 01)
0002:01:00.0 Non-Volatile memory controller: Silicon Motion, Inc. SM2263EN/SM2263XT (DRAM-less) NVMe SSD Controllers (rev 03)  
```
  
2.查看pci设备详细信息  
如下显示0001:01:00.0设备详细信息

```
lspci -vvvs 0001:01:00.0

```

3.nvme ssd读测试

```
fio --name read --eta-newline=5s --filename=/dev/nvme0n1 --rw=read --size=2g --io_size=10g --blocksize=1024k --ioengine=libaio --fsync=10000 --iodepth=32 --direct=1 --numjobs=1 --runtime=60 --group_reporting
```

4.nvme ssd写测试

```
fio --name write --eta-newline=5s --filename=/dev/nvme0n1 --rw=write --size=2g --io_size=60g --blocksize=1024k --ioengine=libaio --fsync=10000 --iodepth=32 --direct=1 --numjobs=1 --runtime=60 --group_reporting
```

# FAQ
