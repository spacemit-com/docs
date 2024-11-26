# PWM

介绍PWM的配置和调试方式

## 模块介绍  

pwm控制器是一种通过改变电脉冲宽度来控制输出信号的电子元件。  

### 功能介绍  

![pwm](static/pwm.png)
内核通过pwm框架层接口使其他模块可以申请pwm控制器，并控制pwm信号的输出高低。
如：内核的风扇调速和背光亮度都可以用pwm来控制。  

### 源码结构介绍

pwm控制器驱动代码在drivers/pwm目录下：  

```
drivers/pwm  
|--core.c            #内核pwm框架接口代码
|--pwm-sysfs.c       #内核pwm框架注册到sysfs代码
|--pwm-pxa.c         #k1 pwm驱动  
```  

## 关键特性  

| 特性 |
| :-----|
| 可生成200HZ到6.4MHZ的pwm信号 |
| k1平台支持20路可配置的pwm |

## 配置介绍

主要包括驱动使能配置和dts配置

### CONFIG配置

CONFIG_PWM
此为内核平台pwm框架提供支持，支持k1 pwm驱动情况下，应为Y

```
Symbol: PWM [=y]
Device Drivers
      -> Pulse-Width Modulation (PWM) Support (PWM [=y])
```

在支持平台层pwm框架后，配置CONFIG_PWM_PXA为Y，支持k1 pwm驱动

```
Symbol: PWM_PXA [=y]
      ->PXA PWM support (PWM_PXA [=y])
```

### dts配置

由于20路pwm的使用方法和配置方法类似，这里以pwm0为例

#### pinctrl

可查看linux仓库的arch/riscv/boot/dts/spacemit/k1-x_pinctrl.dtsi，参考已配置好的pwm节点配置，如下：

```dts
      pinctrl_pwm0_1: pwm0_1_grp {
         pinctrl-single,pins =<
            K1X_PADCONF(GPIO_14, MUX_MODE3, (EDGE_NONE | PULL_UP | PAD_1V8_DS2))    /* pwm0 */
         >;
      };
```

#### dtsi配置示例

dtsi中配置pwm控制器基地址和时钟复位资源，正常情况无需改动

```dts
1351         pwm0: pwm@d401a000 {
1352             compatible = "spacemit,k1x-pwm";
1353             reg = <0x0 0xd401a000 0x0 0x10>;
1354             #pwm-cells = <1>;
1355             clocks = <&ccu CLK_PWM0>;
1356             resets = <&reset RESET_PWM0>;
1357             k1x,pwm-disable-fd;
1358             status = "disabled";
1359         };
```

#### dts配置示例

dts完整配置，如下所示

```dts
807 &pwm0 {
808     pinctrl-names = "default";
809     pinctrl-0 = <&pinctrl_pwm0_1>;
810     status = "okay";
811 };
```

## 接口介绍

### API介绍

linux内核实现了其他设备或框架如背光，led灯，背光等对pwm的引用和调节。
常用：

```
struct pwm_device *devm_pwm_get(struct device *dev, const char *con_id)
该接口实现了从pwm框架中获取pwm资源
int pwm_apply_state(struct pwm_device *pwm, const struct pwm_state *state)
该接口实现了对pwm状态的设置
```

## Debug介绍

pwm通过sysfs提供给用户层一个非编程的使用方法，可以依据上述pwm对应pin外接可调速风扇进行测试，过程如下
以下基于bianbu linux系统验证

```sh
# cd /sys/class/pwm/
# ls # 每个节点代表一个led灯
pwmchip0  pwmchip1  pwmchip2  pwmchip3  pwmchip4  pwmchip5  pwmchip6

# echo 0 > pwmchip0/export
# ls pwmchip0/pwm0/
capture     enable      polarity    uevent
duty_cycle  period      power

# 设置PWM一个周期的时间，单位为ns，即一个周期为1KHZ
# echo 1000000 > pwmchip0/pwm0/period 

# 设置PWM占空比
# echo 500000 > pwmchip0/pwm0/duty_cycle 

# 使能PWM
# echo 1 > pwmchip0/pwm0/enable 

# 调节占空比，此时风扇转速降低
# echo 50000 > pwmchip0/pwm0/duty_cycle 

# 关闭PWM
# echo 0 > pwmchip0/pwm0/enable 
```

* 需注意sysfs里可用的pwmchipx均为没有使用的pwm，若内核驱动中已将该pwm通过类似pwm_get的接口申请，则该pwm无法通过sysfs配置  

## 测试介绍

pwm输出的信号电平高低可以通过控制占空比来调节。  
实际测试可使用sysfs下的pwm节点和可调速的pwm风扇进行测试。

## FAQ
