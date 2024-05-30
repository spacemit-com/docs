介绍PCIe的功能和使用方法。

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
