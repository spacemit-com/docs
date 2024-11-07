介绍Standby的功能和使用方法。
# 模块介绍

## 功能介绍


## 源码结构介绍
控制器驱动代码在vi drivers/soc/spacemit/pm/目录下：
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

## API介绍
- 
- 

## Debug介绍
### sysfs

```
```
### debugfs
```
```
# FAQ