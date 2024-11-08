介绍CPUFREQ的功能和使用方法。
# 模块介绍
CPUFREQ子系统负责cpu运行时，对其频率及电压进行调整，以求性能满足的前提下，cpu的功耗尽可能低的技术
## 功能介绍
![](static/cpufreq.png)

1. cpufreq core是cpufreq framework的核心模块，它主要实现三类功能：  
    1.1 抽象调频调压的公共逻辑接口  
    1.2 以sysfs的形式向用户空间提供统一的接口，以notifier的形式向其他driver提供频率变化的通知
    1.3 提供CPU频率和电压控制的驱动框架  
2. cpufreq governor负责调频调压的各种策略  
3. cpufreq driver负责平台相关调频调压机制的实现
4. cpufreq stats负责调频信息和各频点运行事件统计，提供每个CPU的cpufreq有关的统计信息

## 源码结构介绍
CPU调频平台驱动目录如下：
```
drivers/cpufreq/
├── cpufreq.c
├── cpufreq_conservative.c
├── cpufreq-dt.c
├── cpufreq-dt.h
├── cpufreq-dt-platdev.c
├── cpufreq_governor_attr_set.c
├── cpufreq_governor.c
├── cpufreq_governor.h
├── cpufreq_ondemand.c
├── cpufreq_ondemand.h
├── cpufreq_performance.c
├── cpufreq_powersave.c
├── cpufreq_stats.c
├── cpufreq_userspace.c
├── freq_table.c
├── spacemit-cpufreq.c --> 平台驱动  
```
# 关键特性
## 特性
| 特性 | 特性说明 |
| :-----| :----|
| 支持动态调频调压 |
| 支持频率boost到1.8Ghz |
## 性能参数
| 支持频率档位 | 频率档位对应电压 |  
| :-----| :----:|  
| 1600000000HZ | 1.05V |  
| 1228800000HZ | 0.95V |  
| 1000000000HZ | 0.95V |  
| 819000000HZ | 0.95V |  
| 614400000HZ | 0.95V |  

测试方法
```
1. 将策略修改成userspace模式
echo userspace > /sys/devices/system/cpu/cpufreq/policy0/scaling_governor

2. 查看所支持的频率列表
cat /sys/devices/system/cpu/cpufreq/policy0/scaling_available_frequencies
614400 819000 1000000 1228800 1600000

3. 设置cpu频率
echo 1228800 > /sys/devices/system/cpu/cpufreq/policy0/scaling_setspeed

4. 查看频率是否设置成功
cat /sys/devices/system/cpu/cpufreq/policy0/scaling_cur_freq

```
# 配置介绍
主要包括驱动使能配置和dts配置
## CONFIG配置
CPUFREQ配置如下：
```
CONFIG_SPACEMIT_K1X_CPUFREQ:

	This adds the CPUFreq driver support for Freescale QorIQ SoCs
	which are capable of changing the CPU's frequency dynamically.
 
	Symbol: SPACEMIT_K1X_CPUFREQ [=y]
	Type  : tristate
	Defined at drivers/cpufreq/Kconfig:315
	Prompt: CPU frequency scaling driver for Spacemit K1X
	Depends on: CPU_FREQ [=y] && OF [=y] && COMMON_CLK [=y]
	Location:
		-> CPU Power Management
			-> CPU Frequency scaling
				-> CPU Frequency scaling (CPU_FREQ [=y])
					-> CPU frequency scaling driver for Spacemit K1X (SPACEMIT_K1X_CPUFREQ [=y])
	Selects: CPUFREQ_DT [=y] && CPUFREQ_DT_PLATDEV [=y]    
```

## dts配置
```
由于dts配置太过庞大，目前进贴其文件：
arch/riscv/boot/dts/spacemit/k1-x_opp_table.dtsi
```
# 接口描述
## 测试介绍
测试调频调压，可以按照上述小结“测试方法”循环测试设置不同频率的正确性
## API介绍


## Debug介绍
### sysfs

```
在如下目录中有关于CPUFREQ所有的sysfs节点
/sys/devices/system/cpu/cpufreq/policy0/
affected_cpus
related_cpus
查看系统支持的策略
scaling_governor

支持boost的节点
boost

查看系统支持的频率点
scaling_available_frequencies

查看软件支持的最大频率
scaling_max_freq        

查看cpu跑的当前的频率
cpuinfo_cur_freq

查看系统支持的策略
scaling_available_governors   

查看软件支持的最小频率
scaling_min_freq

cpu硬件支持的最大频率
cpuinfo_max_freq

boost模式下cpu支持的频率
scaling_boost_frequencies

userspace模式是设置cpu频率的接口
scaling_setspeed

cpu硬件支持的最小频率
cpuinfo_min_freq

查看当前CPU跑的频率
scaling_cur_freq
cpuinfo_transition_latency

当前cpu调频驱动的名称
scaling_driver

```
### debugfs
# FAQ