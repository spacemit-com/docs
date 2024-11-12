# DMA

介绍DMA控制器的配置和dma-slave使用dma的方法

## 模块介绍  

DMA(Direct Memory Access),是一种无需CPU直接控制传输，通过硬件为source和destination之间开辟一条传输数据的通路的方法。
本模块为DMA-controller，即DMAmaster，负责连接dma-channel和完成数据搬运。

### 功能介绍  

![](static/dma.JPEG)
通过DMA框架和K1 dma控制器驱动，实现了内存到内存，内存到外设，外设到内存三种传输方向。并实现了内存搬运，散列表传输和环形buffer的传输。

### 源码结构介绍

dma控制器驱动代码在drivers/dma目录下：  

```  
drivers/dma  
|--dmaengine.c dmaengine.h      #内核dma框架代码
|--dmatest.c         #内核dma测试代码
|--mmp_pdma_k1x.c          #k1 dma控制器驱动  
```  

## 关键特性  

### 特性

支持内存到内存，内存到外设，外设到内存传输
16路channel可用
burst数支持8 16 32 64
每个传输描述符最大支持8191B传输

### 性能参数

dma进行内存到内存方向的数据搬运的极限速度为220MB/s

测试方法：可使用dmatest.c进行测试，具体工具使用方法见debug章节

## 配置介绍

主要包括驱动使能配置和dts配置

### CONFIG配置

CONFIG_DMADEVICES
此为内核平台dma框架提供支持，支持k1 dma驱动情况下，应为Y

```
Symbol: DMADEVICES [=y]
Device Drivers
      -> DMA Engine support (DMADEVICES [=y])
```

在支持平台层dma框架后，配置CONFIG_MMP_PDMA_SPACEMIT_K1X为Y，支持k1 dma控制器驱动

```
Symbol: MMP_PDMA_SPACEMIT_K1X [=y]
      ->Spacemit mmp_pdma support (MMP_PDMA_SPACEMIT_K1X [=y])
```

### dts配置

dma的使用方法是选择一个通道并指定起始地址和目标地址，如内存到内存，内存到外设等。
故本节介绍使能dma控制器的dts配置和其他设备如uart使用dma时的dts配置

#### dma控制器配置

可查看linux仓库的arch/riscv/boot/dts/spacemit/k1-x.dtsi，参考已配置好的dma节点配置，如下：

```dts
    pdma0: pdma@d4000000 {
  compatible = "spacemit,pdma-1.0";
  reg = <0x0 0xd4000000 0x0 0x4000>;
  interrupts = <72>;
  interrupt-parent = <&intc>;
  clocks = <&ccu CLK_DMA>;
  resets = <&reset RESET_DMA>;
  #dma-cells= <2>;
  #dma-channels = <16>;
  max-burst-size = <64>;
  reserved-channels = <15 45>;
  power-domains = <&power K1X_PMU_BUS_PWR_DOMAIN>;
  clk,pm-runtime,no-sleep;
  cpuidle,pm-runtime,sleep;
  interconnects = <&dram_range4>;
  interconnect-names = "dma-mem";
  status = "ok";
 };
```

此节点配置了dma的时钟复位资源，通道数，最大burst数量，并将通道15预留给握手号为45号的设备

#### dma-slave使用dma的配置示例

这里以uart0为例，在uart0的节点下添加下述属性，其中宏DMA_UART0_RX和DMA_UART0_TX在include/dt-bindings/dma/k1x-dmac.h文件里定义

```dts
 dmas = <&pdma0 DMA_UART0_RX 1
   &pdma0 DMA_UART0_TX 1>;
 dma-names = "rx", "tx";
```

## 接口介绍

### API介绍

linux内核实现了申请dma通道，配置dma传输，准备资源，启动传输等接口。
常用：

```
申请通道
struct dma_chan *dma_request_chan(struct device *dev, const char *name)
配置通道参数，如传输宽度，数据量，起始/目标地址
static inline int dmaengine_slave_config(struct dma_chan *chan,
                                          struct dma_slave_config *config)
下述三个接口实现了dma开始传输前的准备资源
dmaengine_prep_dma_memcpy
dmaengine_prep_slave_sg
dmaengine_prep_dma_cyclic
将描述符加入到传输任务的链表里
static inline dma_cookie_t dmaengine_submit(struct dma_async_tx_descriptor *desc)
开始传输
static inline void dma_async_issue_pending(struct dma_chan *chan)
释放通道
static inline void dma_release_channel(struct dma_chan *chan)
停止传输，如音频播放时暂停
static inline int dmaengine_terminate_all(struct dma_chan *chan)
```

## 测试介绍

由于在内存到设备或设备到内存的数据通路工作中需要外设的驱动配合，如uart0在发送数据时要将数据通过dma从内存搬运到uart tx fifo里，这里dma需要uart来控制启动。所以一般在测试时我们采用内存到内存的数据通路，可直接使用内核自带的dmatest.c程序

```
echo dma0chan8 > /sys/module/dmatest/parameters/channel  #选择一个没有使用的通道
echo 1 > /sys/module/dmatest/parameters/iterations  #设置传输次数，这里以1为例
echo 4096 > /sys/module/dmatest/parameters/transfer_size #设置传输数据量大小(不大于16k)
echo 1 > /sys/module/dmatest/parameters/run   #开始传输
```

首先打开内核config配置：CONFIG_DMATEST，生成新的固件后启动到内核

## FAQ
