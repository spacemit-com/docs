介绍THERMAL的功能和使用方法。
# 模块介绍
thermal特指一套关于温控机制的驱动框架，Linux thermal框架是Linux系统下温度控制的一套架构，主要用来解决随着设备性能的不断增强而引起的日益严重的发热问题
## 功能介绍
![](static/thermal.png)

1. thermal_cooling_device对应系实施冷却措施的驱动，是温控的执行者  
2. thermal core是thermal的只要程序，驱动初始化程序，维护thermal_zone，governor，cooling device三者的关系，并通过sysfs和用户空间交互
3. thermal governor是温度控制算法，解决温控发生时，cooling device如何选择cooling state的问题。
4. thermal zone device，主要用来创建thermal zone结点和连接thermal sensor， 在/sys/class/thermal目录下的thermal_zone, 该节点通过dts文件配置生成
5. thermal sensor是温度传感器，主要是给thermal提供温度感知

## 源码结构介绍
CPU调频平台驱动目录如下：
```
drivers/thermal/
├── cpufreq_cooling.c
├── cpufreq_cooling.o
├── cpuidle_cooling.c
├── devfreq_cooling.c
├── gov_bang_bang.c
├── gov_fair_share.c
├── gov_power_allocator.c
├── gov_step_wise.c
├── gov_user_space.c
├── k1x-thermal.c    ---> 平台驱动
├── k1x-thermal.h
├── thermal_core.c
├── thermal_core.h
├── thermal_helpers.c
├── thermal_hwmon.c
├── thermal_hwmon.h
├── thermal_of.c
├── thermal_sysfs.c
```
# 关键特性
## 特性
| 特性 | 特性说明 |
| :-----| :----|
| 支持cpu温度控制 |
| 支持115°C过温关机 |
## 性能参数
| 支持频率档位 | 频率档位对应电压 |  
| :-----| :----|   

测试方法
```
利用外部温枪或者跑负载大而变动的应用以创造温度一个温度可变的环境，查看thermal以及cpufreq的节点看是否cpu调温符合预期  
1. thermal sensor节点
    1.1 cat /sys/class/thermal/thermal_zone1/temp  
2. cpu调频节点
    2.1 cat /sys/devices/system/cpu/cpufreq/policy0/scaling_cur_freq
```
# 配置介绍
主要包括驱动使能配置和dts配置
## CONFIG配置
THERMAL配置如下：
```
CONFIG_K1X_THERMAL:
Enable this option if you want to have support for thermal management
controller present in Spacemit SoCs

	Symbol: K1X_THERMAL [=y]
	Type  : tristate
	Defined at drivers/thermal/Kconfig:450
	Prompt: Spacemit K1X Thermal Support
	Depends on: THERMAL [=y] && OF [=y] && SOC_SPACEMIT [=y]
	Location:
		-> Device Drivers
			-> Thermal drivers (THERMAL [=y])
				-> Spacemit K1X Thermal Support (K1X_THERMAL [=y]) 
```

## dts配置
```
&thermal_zones {
        cluster0_thermal {
                polling-delay = <0>;
                polling-delay-passive = <0>;
                thermal-sensors = <&thermal 3>;

                thermal0_trips: trips {
                        cls0_trip0: cls0-trip-point0 {
                                temperature = <75000>;
                                hysteresis = <5000>;
                                type = "passive";
                        };

                        cls0_trip1: cls0-trip-point1 {
                                temperature = <85000>;
                                hysteresis = <5000>;
                                type = "passive";
                        };

                        cls0_trip2: cls0-trip-point2 {
                                temperature = <95000>;
                                hysteresis = <5000>;
                                type = "passive";
                        };

                        cls0_trip3: cls0-trip-point3 {
                                temperature = <105000>;
                                hysteresis = <5000>;
                                type = "passive";
                        };

                        cls0_trip4: cls0-trip-point4 {
                                temperature = <115000>;
                                hysteresis = <5000>;
                                type = "critical";
                        };
                };

                cooling-maps {
                        map0 {
                                trip = <&cls0_trip0>;
                                cooling-device = <&cpu_0 0 0>,
                                                 <&cpu_1 0 0>,
                                                 <&cpu_2 0 0>,
                                                 <&cpu_3 0 0>,
                                                 <&cpu_4 0 0>,
                                                 <&cpu_5 0 0>,
                                                 <&cpu_6 0 0>,
                                                 <&cpu_7 0 0>;
                        };

                        map1 {
                                trip = <&cls0_trip1>;
                                cooling-device = <&cpu_0 1 1>,
                                                 <&cpu_1 1 1>,
                                                 <&cpu_2 1 1>,
                                                 <&cpu_3 1 1>,
                                                 <&cpu_4 1 1>,
                                                 <&cpu_5 1 1>,
                                                 <&cpu_6 1 1>,
                                                 <&cpu_7 1 1>;
                        };

                        map2 {
                                trip = <&cls0_trip2>;
                                cooling-device = <&cpu_0 2 3>,
                                                 <&cpu_1 2 3>,
                                                 <&cpu_2 2 3>,
                                                 <&cpu_3 2 3>,
                                                 <&cpu_4 2 3>,
                                                 <&cpu_5 2 3>,
                                                 <&cpu_6 2 3>,
                                                 <&cpu_7 2 3>;
                        };

                        map3 {
                                trip = <&cls0_trip3>;
                                cooling-device = <&cpu_0 4 5>,
                                                 <&cpu_1 4 5>,
                                                 <&cpu_2 4 5>,
                                                 <&cpu_3 4 5>,
                                                 <&cpu_4 4 5>,
                                                 <&cpu_5 4 5>,
                                                 <&cpu_6 4 5>,
                                                 <&cpu_7 4 5>;
                        };
                };
        };

        cluster1_thermal {
                polling-delay = <0>;
                polling-delay-passive = <0>;
                thermal-sensors = <&thermal 4>;

                thermal1_trips: trips {
                        cls1_trip0: cls1-trip-point0 {
                                temperature = <75000>;
                                hysteresis = <5000>;
                                type = "passive";
                        };

                        cls1_trip1: cls1-trip-point1 {
                                temperature = <85000>;
                                hysteresis = <5000>;
                                type = "passive";
                        };

                        cls1_trip2: cls1-trip-point2 {
                                temperature = <95000>;
                                hysteresis = <5000>;
                                type = "passive";
                        };

                        cls1_trip3: cls1-trip-point3 {
                                temperature = <105000>;
                                hysteresis = <5000>;
                                type = "passive";
                        };

                        cls1_trip4: cls1-trip-point4 {
                                temperature = <115000>;
                                hysteresis = <5000>;
                                type = "critical";
                        };
                };
        };
};

```
# 接口描述
## 测试介绍
测试thermal驱动可以按照上述测试方法的描述，进行测试
## API介绍
请参考内核目录下面的Document文档介绍：  
vi Documentation/driver-api/thermal/

## Debug介绍
### sysfs

```
请参考内核目录下面的：
Documentation/driver-api/thermal/sysfs-api.rst
```
### debugfs
```