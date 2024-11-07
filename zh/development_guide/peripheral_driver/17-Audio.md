介绍Audio的功能和使用方法。

# 模块介绍
Audio模块包含2个I2S，1个HDMIAUDIO。

## 功能介绍
![](static/AUDIO.png)

ALSA音频框架可以分为以下几个层次：  
- ALSA Library
对应用程序提供统一的API接口，各个APP应用程序只要调用 alsa-lib 提供的 API接口来实现放音、录音、控制。现在提供了两套基本的库，tinyalsa是一个简化的alsa-lib库，Android系统主要使用
- ALSA CORE
ALSA核心层，向上提供逻辑设备（PCM/CTL/MIDI/TIMER/…）系统调用，向下驱动硬件设备（Machine/I2S/DMA/CODEC）
- ASoC Core
ALSA 的标准框架，是 ALSA-driver 的核心部分，提供了各种音频设备驱动的通用方法和数据结构
- Hardware driver
音频硬件设备驱动，由三大部分组成，分别是 Machine、Platform、Codec，提供的 ALSA Driver API 和相应音频设备的初始化及工作流程，实现具体的功能组件，这是驱动开发人员需要具体实现的部分。
Machine：一般是指某款单板，包含特定的外设，为CPU、Codec提供了一个载体，Machine驱动几乎是不可重用的。Machine驱动将Platform驱动和Codec驱动关联在一起，通过snd_soc_dai_link指定使用哪个Platform驱动，使用哪个Soc端的dai（digital audio interface）接口，使用哪个Codec驱动，使用Codec上的哪个dai接口，同时也做一些特定于单板的操作。
Platform：一般是指某一个SoC平台，可以理解为某款Soc，具有I2S，AC97音频接口等，内部有时钟，DMA单元用于传输音频数据。Platform驱动只与特定的Soc有关，实现Soc的音频DMA驱动和Soc端的dai接口驱动，它只与SoC相关，与Machine无关，这样我们就可以把Platform抽象出来，使得同一款SoC不用做任何的改动，就可以用在不同的Machine中。
Codec：音频编解码器，Codec里面包含了I2S接口、D/A、A/D、Mixer、PA(功放)，通常包含多种输入（Mic、Line-in、I2S、PCM）和多个输出（耳机、喇叭、听筒，Line-out），一般Soc可通过I2C来控制codec芯片。Codec驱动只与Codec编解码器驱动有关，与Soc和Machine无关。Codec和Platform一样，要实现为可重用的部件，同一个Codec可以被不同的Machine使用。

### k1 Audio功能介绍

K1目前支持两种音频声卡方案
方案一：HDMIAUDIO，支持播放
方案二：I2S0搭配一个I2C外挂的Codec ES8326B，支持播放和录制

## 源码结构介绍
I2S/HDMIAUDIO控制器驱动代码在sound/soc/spacemit目录下：
```
sound/soc/spacemit
├── Kconfig
├── Makefile
├── spacemit-dummy-codec.c    #dummy codec，配合hdmiaudio创建声卡
├── spacemit-snd-card.c       #声卡驱动
├── spacemit-snd-i2s.c        #i2s驱动
├── spacemit-snd-i2s.h
├── spacemit-snd-pcm-dma.c    #platform驱动，主要是pcm相关
├── spacemit-snd-sspa.c       #hdmiaudio驱动
├── spacemit-snd-sspa.h
```
Codec ES8326B驱动代码在sound/soc/codec目录下：
```
sound/soc/codec
├── es8326.c
├── es8326.h
```
# I2S

## 关键特性
  支持48000采样率，16bit采样深度，2声道
  支持播放和录制功能
  支持全双工

## 配置介绍
主要包括驱动使能配置和dts配置
### CONFIG配置
#### 音频功能支持
CONFIG_SOUND、CONFIG_SND、CONFIG_SND_SOC为ALSA音频驱动框架提供支持，默认情况，此选项为Y
```
Device Drivers
        Sound card support (SOUND [=y])
                Advanced Linux Sound Architecture (SND [=y])
                        ALSA for SoC audio support (SND_SOC [=y])
```
#### K1音频功能支持
CONFIG_SND_SOC_SPACEMIT、CONFIG_SPACEMIT_CARD、CONFIG_SPACEMIT_PCM为K1音频功能提供支持，默认情况下，此选项为Y
```
Device Drivers
        Sound card support (SOUND [=y])
                Advanced Linux Sound Architecture (SND [=y])
                        ALSA for SoC audio support (SND_SOC [=y])
                                SoC Audio for SPACEMIT System-on-Chip (SND_SOC_SPACEMIT [=y])
                                        Audio Simple Card (SPACEMIT_CARD [=y])
                                        Audio Platform Pcm (SPACEMIT_PCM [=y])
```
#### I2S功能支持
CONFIG_SPACEMIT_I2S为I2S功能提供支持，默认情况下，此选项为Y
```
Device Drivers
        Sound card support (SOUND [=y])
                Advanced Linux Sound Architecture (SND [=y])
                        ALSA for SoC audio support (SND_SOC [=y])
                                SoC Audio for SPACEMIT System-on-Chip (SND_SOC_SPACEMIT [=y])
                                        Audio Simple Card (SPACEMIT_CARD [=y])
                                        Audio Platform Pcm (SPACEMIT_PCM [=y])
                                        Audio Cpudai I2S (SPACEMIT_I2S [=y])
```
### DTS配置
#### pinctrl
- i2s0 pinctrl配置
有两组pinctrol配置，根据实际情况硬件设计进行配置
```c
        pinctrl_sspa0_0: sspa0_0_grp {
                pinctrl-single,pins =<
                        K1X_PADCONF(GPIO_118, MUX_MODE3, (EDGE_NONE | PULL_UP | PAD_1V8_DS0))   /* sspa0_clk */
                        K1X_PADCONF(GPIO_119, MUX_MODE3, (EDGE_NONE | PULL_UP | PAD_1V8_DS0))   /* sspa0_frm */
                        K1X_PADCONF(GPIO_120, MUX_MODE3, (EDGE_NONE | PULL_UP | PAD_1V8_DS0))   /* sspa0_txd */
                        K1X_PADCONF(GPIO_121, MUX_MODE3, (EDGE_NONE | PULL_UP | PAD_1V8_DS0))   /* sspa0_rxd */
                        K1X_PADCONF(GPIO_122, MUX_MODE3, (EDGE_NONE | PULL_UP | PAD_1V8_DS0))   /* sspa0_sysclk */
                >;
        };

        pinctrl_sspa0_1: sspa0_1_grp {
                pinctrl-single,pins =<
                        K1X_PADCONF(GPIO_58,  MUX_MODE2, (EDGE_NONE | PULL_UP | PAD_1V8_DS0))   /* sspa0_sysclk */
                        K1X_PADCONF(GPIO_111, MUX_MODE2, (EDGE_NONE | PULL_UP | PAD_1V8_DS0))   /* sspa0_clk */
                        K1X_PADCONF(GPIO_112, MUX_MODE2, (EDGE_NONE | PULL_UP | PAD_1V8_DS0))   /* sspa0_frm */
                        K1X_PADCONF(GPIO_113, MUX_MODE2, (EDGE_NONE | PULL_UP | PAD_1V8_DS0))   /* sspa0_txd */
                        K1X_PADCONF(GPIO_114, MUX_MODE2, (EDGE_NONE | PULL_UP | PAD_1V8_DS0))   /* sspa0_rxd */
                >;
        }

&i2s0 {
        pinctrl-names = "default";
        pinctrl-0 = <&pinctrl_sspa0_0>;         #使用pinctrl_sspa0_0这组pin
        status = "okay";
};

```

- i2s1 pinctrl配置
```c
        pinctrl_sspa1: sspa1_grp {
                pinctrl-single,pins =<
                        K1X_PADCONF(GPIO_24, MUX_MODE3, (EDGE_NONE | PULL_UP | PAD_1V8_DS0))    /* sspa1_sysclk */
                        K1X_PADCONF(GPIO_25, MUX_MODE1, (EDGE_NONE | PULL_UP | PAD_1V8_DS0))    /* sspa1_sclk */
                        K1X_PADCONF(GPIO_26, MUX_MODE1, (EDGE_NONE | PULL_UP | PAD_1V8_DS0))    /* sspa1_frm */
                        K1X_PADCONF(GPIO_27, MUX_MODE1, (EDGE_NONE | PULL_UP | PAD_1V8_DS0))    /* sspa1_txd */
                        K1X_PADCONF(GPIO_28, MUX_MODE1, (EDGE_NONE | PULL_UP | PAD_1V8_DS0))    /* sspa1_rxd */
                >;
        };

&i2s1 {
        pinctrl-names = "default";
        pinctrl-0 = <&pinctrl_sspa1>;          #使用pinctrl_sspa1这组pin
        status = "okay";
};

```

### I2S-Codec声卡配置
#### Codec配置
以Codec ES8326B为例，进行完整声卡配置
##### Config配置
打开ES8326B配置
```
Device Drivers│
        Sound card support (SOUND [=y])
                Advanced Linux Sound Architecture (SND [=y])
                        ALSA for SoC audio support (SND_SOC [=y])
                                CODEC drivers
                                        Everest Semi ES8326 CODEC (SND_SOC_ES8326 [=y])
```

##### dts配置
- gpio

需要按实际硬件设计来配置，例如K1某些开发板上，ES8326B通过gpio129完成耳机插拔检测，通过gpio127控制板级扬声器。
```c
        es8326: es8326@19{
                interrupt-parent = <&gpio>;
                interrupts = <126 1>;                 #耳机插拔检测
                spk-ctl-gpio = <&gpio 127 0>;         #板级扬声器控制gpio
        };

```

##### dts 配置示例

Codec ES8236B的完整方案配置如下：

```c
        es8326: es8326@19{
                compatible = "everest,es8326";
                reg = <0x19>;
                #sound-dai-cells = <0>;
                interrupt-parent = <&gpio>;
                interrupts = <126 1>;                 #耳机插拔检测
                spk-ctl-gpio = <&gpio 127 0>;         #板级扬声器控制gpio
                everest,mic1-src = [44];              #ADC sourcs配置
                everest,mic2-src = [66];
                status = "okay";
        };
```
##### mclk配置
Codec ES8326B的mclk由I2S0提供，在声卡节点sound_codec中进行配置
```c

&sound_codec {
        status = "okay";
        simple-audio-card,name = "snd-es8326";
        spacemit,mclk-fs = <64>;                      #配置mclk = 64 * fs，即3.072MHz
        simple-audio-card,codec {
                sound-dai = <&es8326>;
        };
};

```
注意：spacemit,mclk-fs只支持配置成64/128/256，即3.072/6.144/12.288MHz

#### 声卡配置
##### dts配置

```c
        sound_codec: snd-card@1 {
                compatible = "spacemit,simple-audio-card";
                simple-audio-card,format = "i2s";
                status = "disabled";
                interconnects = <&dram_range4>;
                interconnect-names = "dma-mem";
                spacemit,init-jack;
                simple-audio-card,cpu {                      #cpudai配置
                        sound-dai = <&i2s0>;
                };
                simple-audio-card,plat {
                        sound-dai = <&i2s0_dma>;             #platform pcm配置
                };
        };

&sound_codec {
        status = "okay";
        simple-audio-card,name = "snd-es8326";
        spacemit,mclk-fs = <64>;
        simple-audio-card,codec {
                sound-dai = <&es8326>;                       #codecdai配置
        };
};

```

# HDMIAUDIO

## 关键特性
HDMIAUDIO
  支持48000采样率，16bit采样深度，2声道
  只支持播放

## 配置介绍
主要包括驱动使能配置和dts配置。因为HDMIAUDIO依赖于HDMI显示功能，需要确保HDMI显示已支持，请参考对应的文档。
### CONFIG配置
#### 音频功能支持
CONFIG_SOUND、CONFIG_SND、CONFIG_SND_SOC为ALSA音频驱动框架提供支持，默认情况，此选项为Y
```
Device Drivers
        Sound card support (SOUND [=y])
                Advanced Linux Sound Architecture (SND [=y])
                        ALSA for SoC audio support (SND_SOC [=y])
```
#### K1音频功能支持
CONFIG_SND_SOC_SPACEMIT、CONFIG_SPACEMIT_CARD、CONFIG_SPACEMIT_PCM为K1音频功能提供支持，默认情况下，此选项为Y
```
Device Drivers
        Sound card support (SOUND [=y])
                Advanced Linux Sound Architecture (SND [=y])
                        ALSA for SoC audio support (SND_SOC [=y])
                                SoC Audio for SPACEMIT System-on-Chip (SND_SOC_SPACEMIT [=y])
                                        Audio Simple Card (SPACEMIT_CARD [=y])
                                        Audio Platform Pcm (SPACEMIT_PCM [=y])
```
#### HDMIAUDIO功能支持
CONFIG_SPACEMIT_HDMIAUDIO、CONFIG_SPACEMIT_DUMMYCODEC为HDMIAUDIO功能提供支持，默认情况下，此选项为Y
```
Device Drivers
        Sound card support (SOUND [=y])
                Advanced Linux Sound Architecture (SND [=y])
                        ALSA for SoC audio support (SND_SOC [=y])
                                SoC Audio for SPACEMIT System-on-Chip (SND_SOC_SPACEMIT [=y])
                                        Audio Simple Card (SPACEMIT_CARD [=y])
                                        Audio Platform Pcm (SPACEMIT_PCM [=y])
                                        Audio Cpudai HDMI Audio (SPACEMIT_HDMIAUDIO [=y])
                                        Audio CodecDai Dummy Codec (SPACEMIT_DUMMYCODEC [=y]) 
```
### DTS配置
```c
&hdmiaudio {
        status = "okay";
};
```

### HDMIAUDIO声卡配置
#### dts配置

```c

        sound_hdmi: snd-card@0 {
                compatible = "spacemit,simple-audio-card";
                simple-audio-card,name = "snd-hdmi";
                status = "disabled";
                interconnects = <&dram_range4>;
                interconnect-names = "dma-mem";
                simple-audio-card,plat {                        #platform pcm配置
                        sound-dai = <&hdmi_dma>;
                };
                simple-audio-card,codec {                       #codecdai配置
                        sound-dai = <&dummy_codec>;
                };
        };

&sound_hdmi {
        status = "okay";
        simple-audio-card,cpu {
                sound-dai = <&hdmiaudio>;                       #cpudai配置
        };
};

```

# 接口描述
## 测试介绍
音频功能可以通过alsa-utils/tinyalsa工具完成功能测试，目前bianbu-linux上已集成alsa-utils工具。
### 播放测试
- 查看playback设备
```c
//aplay-l查看播放设备，可以看到有两个播放设备
root:/# aplay -l
**** PLAYBACK 硬體裝置清單 ****
card 0: sndhdmi [snd-hdmi], device 0: SSPA2-dummy_codec dummy_codec-0 []
  子设备: 1/1
  子设备 #0: subdevice #0
card 1: sndes8326 [snd-es8326], device 0: i2s-dai0-ES8326 HiFi ES8326 HiFi-0 []
  子设备: 1/1
  子设备 #0: subdevice #0
root:/#
//hdmiaudio播放设备，cardid为0，deviceid为0
//I2S-Codec播放设备，cardid为1，deviceid为0
```
- 播放测试
```c
选择设备进行播放，通过cardid和deviceid进行指定
//选择hdmiaudio声卡进行播放
aplay -Dhw:0,0 -r 48000 -f S16_LE --period-size=480 --buffer-size=1920 xxx.way
//选择I2S-Codec声卡进行播放
aplay -Dhw:1,0 -r 48000 -f S16_LE --periopd-size=1024 --buffer-size=4096 xxx.way

```
### 录音测试
- 查看capture设备
```c
//arecord-l查看播放设备，可以看到有一个录制设备
root@spacemit-k1-x-deb1-board:~# arecord -1
****
CAPTURE硬體裝置清單
****
card 1: sndes8326 [snd-es8326], device 0: i2s-daai0-ES8326 HiFi ES8326 HiFi-@
子设备:1/1
子设备 #0: subdevice #0
root@spacemit-k1-x-deb1-board:~#
//I2S-Codec录制设备，cardid为1，deviceid为0
```
- 录音测试
```c
//选择I2S-Codec声卡进行录制
arecord -Dhw:1,0 -r 48000 -c 2 -f S16_LE --period-size=1024 --buffer-size=4096 xxx.wav
```
## API介绍

请参阅相关的Linux官方文档。

## Debug介绍

可以通过/proc/asound/下的节点进行debug
- 查看声卡设备
```c
root:/# cat /proc/asound/pcm
00-00: SSPA2-dummy_codec dummy_codec-0 :  : playback 1
01-00: i2s-dai0-ES8326 HiFi ES8326 HiFi-0 :  : playback 1 : capture 1
root:/#
```
- 查看声卡状态等信息
```c
root:/# cat /proc/asound/card1/pcm0p/sub0/status
state: RUNNING
owner_pid   : 3767
trigger_time: 224110.719883196
tstamp      : 224164.735391138
delay       : 2048
avail       : 2048
avail_max   : 2048
-----
hw_ptr      : 2592768
appl_ptr    : 2594816
root:/# cat /proc/asound/card1/pcm0p/sub0/status
state: RUNNING
owner_pid   : 3767
trigger_time: 224110.719883196
tstamp      : 224166.975406348
delay       : 3072
avail       : 1024
avail_max   : 2048
-----
hw_ptr      : 2700288
appl_ptr    : 2703360
root:/# cat /proc/asound/card1/pcm0p/sub0/hw_params
access: RW_INTERLEAVED
format: S16_LE
subformat: STD
channels: 2
rate: 48000 (48000/1)
period_size: 1024
buffer_size: 4096
root:/# 
```

# FAQ
