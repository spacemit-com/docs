介绍crypto-engine使用方法

# 模块介绍  
crypto-engine实现了硬件加密算法，对明文进行加密。  
## 功能介绍  
![](static/openssl.jpg)
k1 crypto-engine(又称ce)通过硬件实现了(ecb/cbc/xts-)aes加密算法。
## 源码结构介绍
ce驱动代码在drivers/crypto/spacemit目录下：  
```  
drivers/crypto/spacemit
|--spacemit_ce_engine.c            #ce驱动代码
|--spacemit-ce-glue.c              #基于ce驱动实现的加密算法
|--spacemit_engine.h 
```  
crypto的内核框架层实现在内核crypto路径下，这里不做赘述
# 关键特性  
## 特性
支持ecb/cbc/xts模式的aes加密算法
## 性能参数
纯硬件性能可达500MB/s
通过内核实现的加密流程性能可达280MB/s(128k以上大小的数据)

测试方法
openssl speed工具,openssl工具代码最大支持16k大小数据，可二次开发改为128k
```
openssl speed -elapsed -async_jobs 1 -engine afalg -evp aes-128-cbc -multi 1
```
# 配置介绍
主要包括驱动使能配置和dts配置
## CONFIG配置
CONFIG_CRYPTO
此为内核平台crypto框架提供支持，支持k1 ce驱动情况下，应为Y
```
CONFIG_CRYPTO=y
CONFIG_SPACEMIT_REE_AES=y
CONFIG_SPACEMIT_REE_ENGINE=y
```
## dts配置
ce没有输入输出信号，dts中配置时钟复位资源即可

### dtsi配置示例
dtsi中配置ce控制器基地址和时钟复位资源，正常情况无需改动
```dts
	spacemit_crypto_engine@d8600000 {
		compatible = "spacemit,crypto_engine";
		spacemit-crypto-engine-0 = <0xd8600000 0x00100000>;
		interrupt-parent = <&intc>;
		interrupts = <113>;
		num-engines = <1>;
		clocks = <&ccu CLK_AES>;
		resets = <&reset RESET_AES>;
		interconnects = <&dram_range5>;
		interconnect-names = "dma-mem";
		status = "okay";
	};
```

# 接口描述
## 测试介绍
首先验证aes算法是否注册成功
```
cat /proc/crypto
结果中如下
name         : xts(aes)
driver       : __driver-xts-aes-spacemit-ce1
module       : kernel
priority     : 500
refcnt       : 1
selftest     : passed
internal     : no
type         : skcipher
async        : yes
blocksize    : 16
min keysize  : 32
max keysize  : 64
ivsize       : 16
chunksize    : 16
walksize     : 16

name         : cbc(aes)
driver       : __driver-cbc-aes-spacemit-ce1
module       : kernel
priority     : 500
refcnt       : 1
selftest     : passed
internal     : no
type         : skcipher
async        : yes
blocksize    : 16
min keysize  : 16
max keysize  : 32
ivsize       : 16
chunksize    : 16
walksize     : 16

name         : ecb(aes)
driver       : __driver-ecb-aes-spacemit-ce1
module       : kernel
priority     : 500
refcnt       : 2
selftest     : passed
internal     : no
type         : skcipher
async        : yes
blocksize    : 16
min keysize  : 16
max keysize  : 32
ivsize       : 16
chunksize    : 16
walksize     : 16
```
加密算法功能测试可通过openssl工具测试，用法如下：
```
echo "hello,world" | openssl enc -aes128 -e -a -salt -engine afalg  //加密字符串
echo "加密自动生成的密钥" | openssl enc -engine afalg -aes128 -a -d -salt   //解密字符串
openssl enc -aes128 -e -engine afalg -in data.txt -out encrypt.txt -pass pass:12345678   //使用密钥进行加密
openssl enc -aes-cbc -d -engine afalg -in encrypt.txt -out data.txt -pass pass:12345678   //使用密钥进行解密
将解密得到的字符串/文件于加密前比对即可，一样则认为加密功能正常
```
## API介绍
AES驱动主要实现了加密和解密两个API注册进crypto框架
常用：
```
以cbc为例
static int cbc_encrypt(struct skcipher_request *req)
该接口实现了ce硬件cbc模式加密的功能
static int cbc_decrypt(struct skcipher_request *req)
该接口实现了ce硬件cbc模式解密的功能
```
# FAQ