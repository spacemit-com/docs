# PWM

Introduction to PWM Configuration and Debugging

## pinctrl

You can refer to the configured pwm node settings in `k1-x_pinctrl.dtsi` located in the Linux repository under `arch/riscv/boot/dts/spacemit/`. The configuration is as follows:

```dts
   412     pinctrl_pwm0_1: pwm0_1_grp {
   413         pinctrl-single,pins =<
   414             K1X_PADCONF(GPIO_14, MUX_MODE3, (EDGE_NONE | PULL_UP | PAD_1V8_DS2))    /* pwm0 */
   415         >;
   416     };
```

## dts configuration

* By default, all pwm nodes are configured, as shown in the following figure

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

* Enable the node in solution configuration, as follows: Enable the node and configure the pin node

```dts
807 &pwm0 {
808     pinctrl-names = "default";
809     pinctrl-0 = <&pinctrl_pwm0_1>;
810     status = "okay";
811 };
```

## Debug

The following verification is based on the bianbu linux system

```sh
# cd /sys/class/pwm/
# ls # Each node represents an led light
pwmchip0  pwmchip1  pwmchip2  pwmchip3  pwmchip4  pwmchip5  pwmchip6

# echo 0 > pwmchip0/export
# ls pwmchip0/pwm0/
capture     enable      polarity    uevent
duty_cycle  period      power

# The unit is ns, that is, a period is 1KHZ
# echo 1000000 > pwmchip0/pwm0/period 

# Set PWM duty cycle
# echo 500000 > pwmchip0/pwm0/duty_cycle 

# enable PWM
# echo 1 > pwmchip0/pwm0/enable 

# Adjust duty cycle, this time the brightness decreases
# echo 50000 > pwmchip0/pwm0/duty_cycle 

# disable PWM
# echo 0 > pwmchip0/pwm0/enable 
```
