介绍Standby的功能和使用方法。
# 模块介绍
Linux standby模式是一种省电模式，在这种模式下，计算机进入睡眠状态以节省功耗；当系统处于standby模式时，系统自动将大部分硬件设备关闭或者进入低功耗状态，并且DDR进入自刷新状态。
## 功能介绍


## 源码结构介绍
控制器驱动代码在 drivers/soc/spacemit/pm/目录下：
```
drivers/soc/spacemit/pm/
├── Makefile
├── platform_hibernation_pm.c   ---> hibernation 平台层的代码
├── platform_hibernation_pm_ops.c  ---> hibernation平台层的代码
├── platform_pm.c    ----> standby平台层的代码
├── platform_pm_ops.c   ----> standby平台层的代码

```
# 关键特性
## 特性
无
## 性能参数
| 休眠时间 | 唤醒时间 | 休眠功耗 |
| :-----| :----| :----: | :----: |:----: |
| 3s | 1s | 32mw极致优化后 |

测试方法
```

```
# 配置介绍
主要包括驱动使能配置和dts配置
## CONFIG配置
```
 CONFIG_SUSPEND:
	Allow the system to enter sleep states in which main memory is
	powered and thus its contents are preserved, such as the
	suspend-to-RAM state (e.g. the ACPI S3 state).
 
	Symbol: SUSPEND [=y]
	Type  : bool
	Defined at kernel/power/Kconfig:2
		Prompt: Suspend to RAM and standby
		Depends on: ARCH_SUSPEND_POSSIBLE [=y]
		Location:
			-> Power management options
			-> Suspend to RAM and standby (SUSPEND [=y]) 
```
## dts配置
```
无
```
# 接口描述
## 测试介绍
我们可以通过rtc唤醒方式测试我们的我们standby的功能，例子如下:
```
#!/bin/sh
echo 9 > /proc/sys/kernel/printk
echo N > /sys/module/printk/parameters/console_suspend
counter=1

while [ $counter -le 5000 ]
do
        echo +120 > /sys/class/rtc/rtc0/wakealarm
        echo mem > /sys/power/state
        counter=$(($counter+1))
        echo "----->counter:$counter"
        sleep 20
done
```

## API介绍
- 
- 

## Debug介绍
### sysfs

```
无
```
### debugfs
```
cat /sys/power/state
freeze mem
```
# FAQ