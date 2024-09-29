# PWM

介绍PWM的配置和调试方式

## pinctrl

可查看linux仓库的arch/riscv/boot/dts/spacemit/k1-x_pinctrl.dtsi，参考已配置好的pwm节点配置，如下：

```dts
   412     pinctrl_pwm0_1: pwm0_1_grp {
   413         pinctrl-single,pins =<
   414             K1X_PADCONF(GPIO_14, MUX_MODE3, (EDGE_NONE | PULL_UP | PAD_1V8_DS2))    /* pwm0 */
   415         >;
   416     };
```

## dts配置

* 默认已经适配好所有的pwm节点，如下所示

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

* 在方案配置开启该节点，如下，开启节点以及配置pin节点

```dts
807 &pwm0 {
808     pinctrl-names = "default";
809     pinctrl-0 = <&pinctrl_pwm0_1>;
810     status = "okay";
811 };
```

## 调试验证

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

# 调节占空比，此时亮度降低
# echo 50000 > pwmchip0/pwm0/duty_cycle 

# 关闭PWM
# echo 0 > pwmchip0/pwm0/enable 
```
