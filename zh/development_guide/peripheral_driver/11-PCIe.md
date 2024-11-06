介绍PCIe的功能和使用方法。

k1 共有3个PCIe控制器, 支持外接各种PCIe接口设备，包括nvme ssd, sata和wifi等。
PCIe0和USB3控制器共用一个phy硬件, 不能同时使用。应用方案上一般使用USB3, 故PCIe0较少使用。

各控制器配置空间分配如下。
## 配置空间分配
1. PCIe0
mem 空间大小  240MB
io  空间大小  1MB   
2. PCIe1
mem 空间大小  240MB
io  空间大小  1MB
3. PCIe2
mem 空间大小  368MB
io  空间大小  1MB

## 配置空间详细说明
以PCIe2控制器为例，说明PCIe控制器的地址空间分配。
PCIe2 分配地址空间为 0xa00000000~0xb80000000, 大小0x18000000(384MB)。
可以根据需要，对mem、IO和config空间进行修改，只需保证在0xa00000000~0xb80000000范围，且三部分空间不重合。

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
                 <0x02000000 0x0 0xa0000000 0 0xa0000000 0x0 0x17000000>;
        ...
}

含义如下
mem 空间 
  <0x02000000 0x0 0xa0000000 0 0xa0000000 0x0 0x17000000>
  大小为368MB
IO  空间
  <0x01000000 0x0 0xb7002000 0 0xb7002000 0x0 0x100000>
  大小为1MB
config 空间
  <0x0 0xb7000000 0x0 0x00002000>
  大小为8kB

## pinctrl

查看方案原理图，找到 pcie 使用的 pin 组。参考 1.2.2 节，确定 pcie 使用的 pin 组。

假设 pcie1 可以直接采用 k1-x_pinctrl.dtsi 中定义 pinctrl_pcie1_3。

## dts 配置

方案 dts 中 pcie_rc 描述如下。

```c
&pcie1_rc {
        pinctrl-names = "default";
        pinctrl-0 = <&pinctrl_pcie1_3>;
        status = "okay";
};
```
