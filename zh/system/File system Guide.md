
## Linux Storage Stack
图片是Linux存储栈，一般分为3层：文件系统层，块层，设备层。  
下图所示用不同的颜色区分不同的组成部分  
天蓝色：硬件存储设备，k1 涉及到的mmc、sd、U盘、SSD、SATA  
橙色：传输协议<>  
蓝色：Linux系统中的设备文件<包括sda（硬盘）、sr（光驱）等/代表真正的硬件设备，如mmc、nvme等。另一类表示虚拟的块设备，如loop、zram等>  
黄色：I/O调度策略<noop、deadline、CFQ>  
绿色：Linux系统的文件系统  
蓝绿色：Linux Storage操作的基本数据结构BIO  

## Linux回写
由于读写硬盘的速度比读写内存要慢很多，所以为了避免每次读写文件时，都需要对硬盘进行读写操作，Linux内核使用页缓存（Page Cache）机制来对文件中的数据进行缓存。几乎所有的文件读写操作都依赖于高速缓存，除非在打开文件时设置 O_DIRECT flag。  
缓存的存在提高了读写性能，但是也带来了一个可能掉电丢数据的问题，fwrite/write 函数并不能保证数据到存储设备。  
数据回写有两种方式：1.主动同步回写；2.异步后台回写；  
### 主动同步回写
write 只是把数据提交到内核的 Page Cache（假设没有启用 DIO)，要想写到存储设备，必须通过 
fdatasync 或 fsync 系统调用。  
除了内核空间有Page Cache，C库也有一层缓存，需要fflush把数据回写到下一层，也就是内核的page cache，最终需要调用 fsync 再刷到物理存储设备。  
系统支持的sync的区别

| 接口 | 作用范围及颗粒度 |
| :-----| :----|
| flush | 用于C库的缓存刷入内核 |
| fdatasync | 确保单个文件的数据回写，追求更高性能，也就是涉及到文件属性不保证回写，包括文件最后修改时间属性等 |
| fsync | 仅用于回写某一个文件，从内核cache刷入磁盘，包括文件的数据+所有属性 |
| sync |确保整个系统缓冲区（脏页）排入写入队列，然后就返回 |

### 被动回写
Linux 内核会定时触发，大致的异步回写机制步骤如下：   
step 1: 内核按 dirty_writeback_centisecs 的时间间隔唤醒回写线程   
step 2: 回写线程会遍历 Page Cache 寻找那些被标记为脏的时间超过 dirty_expire_centisecs 的页面，并全部回写   
step 3: 回写线程接着会判断脏数据总量是否超过 dirty_background_ratio（单位是百分比）或 dirty_background_bytes，如果超过则回写所有脏数据   
step 4: 回写线程等待下次唤醒周期   

#### 相关配置参数  
| 配置文件 | 功能 | 默认值 |
| :-----| :----| :----|
| dirty_background_ratio | 脏页总数占可用内存的百分比阈值，超过后唤醒回写线程 | 10 | 
| dirty_background_bytes | 脏页总量阈值，超过后唤醒回写线程 | 0 |
| dirty_ratio | 脏页总数占可用内存的百分比阈值，超过后pause进程 | 20 |
| dirty_bytes | 脏页总量阈值，超过后pause进程 | 0 |  
| dirty_expire_centisecs | 脏数据超时回写时间（单位：1/100s） | 3000 |
| dirty_writeback_centisecs | 回写进程定时唤醒时间（单位：1/100s） | 500 |  

### Linux 数据预读
预读分为同步预读、异步预读  
在 Linux 系统中，默认情况下不管是用户态调用 read，还是内核态调用 vfs_read，都会触发数据预读，即一部分数据到 Page Cache 中。
1. 顺序读加速：通过扩大预读窗口(ra->size)逐步增加预读量，典型场景如日志文件读取  
2. 随机读抑制：当检测到非连续访问模式时自动收缩预读窗口  
查看默认预读大小（单位：512B扇区）  
```
cat /sys/block/sda/queue/read_ahead_kb
```
编程层优化：
- 使用posix_fadvise()显式声明访问模式
- 对顺序访问文件设置POSIX_FADV_SEQUENTIAL标志  

### 性能测试
#### Fio
##### 源码
```
https://github.com/axboe/fio
```
可以自行下载源码并编译，或者在OS上自行安装，bian-linux默认已集成该工具  
#### 常用参数
```
[time]
runtime=time
告诉fio在指定的时间段后终止处理。当省略单位时间，该数值以秒为单位进行解释，也可以直接配置runtime=24h，一般用于读写老化场景
time_based
如果设置，即使文件被完全读取或写入，fio也将在指定的运行期间运行。它会在runtime准许时间内多次循环相同的工作负载。一般用于读写老化场景
[I/O tpye]
direct=bool
 是否使用 direct io，测试过程不使用OS 自带的buffer，使结果更真实。
rw=randwrite
测试随机写的I/O：
 顺序IO是指读写操作的访问地址顺序连接。
 随机IO是指读写操作时间连续，但是访问地址不连续，随机分布在磁盘的地址空间中。
rw=randrw
 测试随机写或者读的IO
rwmixwrite=int
 在混合读写模式下，写占用30%
[Blocks size]
bs=int
 单次IO的块文件大小为16k
bsrange=1k-4k,2k-8k,1k-4k
 指定数据块的大小范围，可以为读取、写入和修剪指定逗号分割的范围。
size=5G
 每个线程读写的数据量是5GB
 ```
Job description  
```
name=str
 一个任务的名字，重复了也没关系。如果fio -name=job1 -name=job2，建立了两个任务，共享-name=job1之前的参数。-name=job1之后的就是job2任务独有的参数。
numjobs=int
 创建此作业的指定数量的克隆。每个作业克隆都作为一个独立的线程或者进程产生（执行相同工作的进程或者线程）。每个Job(任务)开1个线程，有多少 -name 就开多少个线程。所以最终线程数 = 任务数 （-name 的数量） * numjiobs。
filename=str
 测试文件的名称。指定相对路径或者绝对路径。没有的话会自行创建。
 ```
I/O engine  
```
ioengine=<str>
 str: sync 基本的同步·读写操作，参考fsync和fdatasync
 str:psync 基于 pread(2) 或者 pwrite(2) 进行IO操作。
 str:vsync
 str:pvsync
 str:pvsync2:
 str:io_uring 快速的Linux原生异步I/O。支持直接和缓冲IO操作。
 str:io_uring_cmd 用于传递命令的快速Linux本机异步I/O。
 str:libaio linux异步I/O。注意：Linux可能只支持具有非缓冲I/O的排队行为
```
IO depth
```
iodepth=int
 队列的深度。在异步模式下，CPU不能一直无限的发命令到SSD。
 ```
Buffers and memory
```
lockmem=int
 使用mlock(2)固定指定数量的内存。可用于模拟较小的内存量。只使用指定内存进行测试。
zero_buffers
 用全零初始化缓冲区。默认值：用随机数据填充缓冲区。一般使用默认值就行。
 ```
Command line options
```
--output=filename
 输出日志信息到指定的文件。
--output-format=format
 指定输出文件的格式。将日志格式设置为normal、terse、json 或 json+。可以选择多种格式，以逗号分隔。terse 是一种基于 CSV 的格式。json+ 类似于 json，只是它添加了延迟存储桶的完整转储。
 ```
Reports
```
group_reporting
 关于显示结果的，汇总每个进程的信息。
 ```
Vertify 

建议一般做读写老化过程中打开
```
verify=crc32c          # 使用CRC32C验证
verify_fatal=1         # 遇到验证错误立即停止
verify_dump=1          # dump错误数据
verify_state_save=1    # 保存验证状态
do_verify=1            # 写入后立即验证
```
fio支持以脚本传参  
### 使用示例  
测试读写速度  
```
Test write speed io:

fio --name=write_test --filename=/path/to/testfile --size=1G --bs=4k --rw=write --ioengine=libaio --direct=1 --numjobs=1 --runtime=60 --group_reporting

Test read speed io:

fio --name=read_test --filename=/path/to/testfile --size=1G --bs=4k --rw=read --ioengine=libaio --direct=1 --numjobs=1 --runtime=60 --group_reporting

Test read and write speed io:

fio --name=rw_test --filename=/path/to/testfile --size=1G --bs=4k --rw=randrw --rwmixread=70 --ioengine=libaio --direct=1 --numjobs=1 --runtime=60 --group_reporting
```
测试读写老化，并进行数据校验  
```
fio --name=randrw_test --filename=/dev/mmcblk2p6 --rw=randrw --rwmixread=70 -rwmixwrite=30 --bs=4k --size=1G --ioengine=libaio --direct=1 --numjobs=1 -runtime=24h -time_based --iodepth=32 --verify=crc32c --verify_fatal=1 --group_reporting
```
### dd
用于快速测试顺序读写性能  
#### 使用示例
```
写顺序性能测试
dd if=/dev/zero of=./testfile bs=1G count=1 oflag=direct
读顺序性能测试
dd if=/dev/zero of=./testfile bs=1G count=1 oflag=direct
```
在测试读性能前，需要确保这个文件不在内存缓存中，需要清除缓存后再测试顺序读性能
```
sudo sh -c 'sync && echo 3 > /proc/sys/vm/drop_caches'
```
随机测试  
dd命令的随机测试一般是通过 bs=4k 去模拟实际场景中的小块数据写入读出  
## 性能问题分析工具
### 应用层常用工具
#### iostat
iostat：实时监控I/O负载、延迟和利用率
适用场景：简单监控cpu使用情况和I/O性能情况和是否为瓶颈
基本语法：iostat [选项] [时间间隔] [次数]
常见参数

使用示例

#### strace
strace追踪进程产生的所有系统调用，包括参数、返回值和执行消耗时间
适用场景:程序意外退出、程序运行缓慢、进程阻塞
使用方式：
strace -f -F -T -tt -o output.txt staced_cmd
-f 跟踪fork产生的子进程
-F 跟踪vfork产生的子进程
-tt 在输出中的每一行前加上时间欣欣，微秒级
-T 显示每一个调用所耗的时间
-o 输出到指定文件

使用示例
strace -T -tt -f -F ls
strace -T -tt -e trace=all -o output.txt -p xxx

每一行都是一条系统调用，等号左边是系统调用的函数名及参数，右边是该调用的返回值。
strace从内核接受信息，不需要以任何特殊的方式来构建内核

lsof
lsof(list open files)是一个查看当前系统文件的工具。在linux环境下，任何事物都以文件的形式存在。
常见参数

lsof输出各列信息的意义如下：
COMMAND:进程名称
PID：进程标识符
PPID：父进程标识符（需要指定-R参数）
USER：进程所有者
PGID：进程所有组
FD：文件描述符，应用程序通过文件描述符识别该文件
TYPE：文件类型，如DIR，REG等
DEVICE：指磁盘名称
SIZE：文件的大小
NODE：索引系欸但
NAME：打开文件的确切名称

使用示例
lsof查看进程打开的文件，使用方式：
#查看文件系统阻塞 
lsof /boot
#查看端口号被哪个进程占用 
lsof  -i : 3306
#查看用户打开哪些文件 
lsof –u username
#查看进程打开哪些文件 
lsof –p  4838
#查看远程已打开的网络链接 
lsof –i @192.168.34.128

内核性能工具
Blktrace
提供 I/O 子系统上时间如何消耗的详细信息, 从中可以 
分析是IO调度慢还是硬件响应慢等；配套工具 blkparse 从 blktrace 读取原始输出，并产生人们可读的输入和输出操作摘要；btt 作为 blktrace 软件包的一部分而被提供， 它分析 blktrace 输出，并显示该数据用在每个 I/O 栈区域的时间量，使它更容易在 I/O 子系统中发现瓶颈。
内核配置
CONFIG_FTRACE=y
CONFIG_BLK_DEV_IO_TRACE=y
常见参数
暂时无法在飞书文档外展示此内容

使用示例
blktrace -d /dev/mmcblk2p6 
blkparse -i mmcblk2p6 -d blkparse.out
btt -i blkparse.out

输出示例
ALL           MIN           AVG           MAX           N
Q2Q      0.000001140   0.000029558   0.012633253   33822
D2C      0.000118989   0.003573138   0.015725019   33823

Q2Q‌：请求间隔延迟
D2C‌：驱动提交到IO完成的物理延迟（设备响应时间）‌
输出解析
关键字段说明
设备号‌：如 8,0（主设备号+次设备号）
CPU ID‌：IO所属CPU核心
序列号‌：IO请求唯一标识
时间戳‌：事件触发时间（秒）
进程ID‌：发起IO的进程ID
事件类型‌（核心字段）：
Q‌：IO请求生成
G‌：请求进入通用块层
I‌：插入IO调度队列
D‌：提交给设备驱动
C‌：IO完成37
操作类型‌：R（读）/W（写）/D（块操作）
扇区信息‌：如 223490+56（起始扇区+扇区数）
流程解读：IO请求生成（Q）→ 进入块层（G）→ 插入调度队列（I）→ 提交驱动（D）→ 完成（C）‌
Bpftrace
Bpftrace是基于eBPF技术的高阶追踪语言，专为Linux系统实时诊断设计。其核心优势包括：
- 动态探针：支持内核/用户态函数无重启挂钩
- 低开销：通过eBPF验证器确保安全执行
- 类C语法：简化脚本开发流程
- 多源数据融合：可同时捕获系统调用、性能事件等
主要应用场景
- 系统性能分析
  - CPU 使用分析‌：跟踪进程调度、上下文切换、CPU 缓存命中率等
  - 内存分析‌：监控内存分配/释放、页面错误、内存泄漏等
  - I/O 性能‌：测量文件读写、磁盘 I/O 延迟等
  - 网络性能‌：分析网络包处理、套接字操作等
- 故障排查与调试
  - 系统调用追踪‌：监控特定进程的系统调用及其参数
  - 函数调用跟踪‌：记录特定函数的调用路径和参数
  - 异常行为检测‌：识别异常的系统调用模式或错误返回值
探针类型
暂时无法在飞书文档外展示此内容
Kprobe
配置
CONFIG_KPROBES
CONFIG_HAVE_KPROBES
CONFIG_KPROBE_EVENTS
使用示例
获取函数的偏移
bpftrace -e 'kprobe:do_sys_open+9 {printf("in here);}'
获取sys_read()运行时间
支持添加过滤条件，可以记录调用上下文信息，如进程名，参数等
bpftrace的这种时间测量方法相比ftrace等工具更加精确，能够提供纳秒级的时间分辨率
#!/usr/bin/bpftrace

BEGIN {
    printf("Tracing sys_read execution time... Hit Ctrl-C to end.\n");
}

kprobe:sys_read 
{
    @start[tid] = nsecs;
}

kretprobe:sys_read 
/@start[tid]/
{
    $duration = nsecs - @start[tid];
    @times = hist($duration / 1000);  // 转换为微秒
    delete(@start[tid]);
}

END {
    printf("\nSys_read execution time distribution (us):\n");
    print(@times);
}
Uprobe
配置
CONFIG_UPROBE
ARCH_SUPPORT_UPROBES
CONFIG_UPROBE_EVENTS
使用示例
统计程序每次执行的时间
bpftrace -e 'BEGIN {@ts = nsecs} uprobe:./example:test {$lat=nsecs - @ts; printf("%s: duration at %d is %d\n", comm, arg0,
$lat); @ts=nsecs}'
Attaching 2 probes...
example: duration at 1 is 657420674
example: duration at 2 is 105863
example: duration at 3 is 16843
example: duration at 4 is 9517
example: duration at 5 is 8882
example: duration at 6 is 9386
example: duration at 7 is 10101
example: duration at 8 is 10086
example: duration at 9 is 10005
example: duration at 10 is 10003
example: duration at 1 is 1000382292
...
文件访问监控
# 跟踪文件打开操作（含进程名和文件名） 
bpftrace -e 'tracepoint:syscalls:sys_enter_openat { printf("%s -> %s\n", comm, str(args.filename)); }
系统调用统计
# 按进程统计系统调用次数 
bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[comm] = count(); }' 
测量vfs_read调用时间分布
bpftrace -e 'kprobe:vfs_read { @start[tid] = nsecs; } kretprobe:vfs_read /@start[tid]/ { @ns[comm] = hist(nsecs - @start[tid]); delete(@start[tid]); }'
Attaching 2 probes...

[...]
@ns[snmp-pass]:
[0, 1]                 0 |                                                    |
[2, 4)                 0 |                                                    |
[4, 8)                 0 |                                                    |
[8, 16)                0 |                                                    |
[16, 32)               0 |                                                    |
[32, 64)               0 |                                                    |
[64, 128)              0 |                                                    |
[128, 256)             0 |                                                    |
[256, 512)            27 |@@@@@@@@@                                           |
[512, 1k)            125 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@       |
[1k, 2k)              22 |@@@@@@@                                             |
[2k, 4k)               1 |                                                    |
[4k, 8k)              10 |@@@                                                 |
[8k, 16k)              1 |                                                    |
[16k, 32k)             3 |@                                                   |
[32k, 64k)           144 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|
[64k, 128k)            7 |@@                                                  |
[128k, 256k)          28 |@@@@@@@@@@                                          |
[256k, 512k)           2 |                                                    |
[512k, 1M)             3 |@                                                   |
[1M, 2M)               1 |                                                    |
bpftrace vs strace
暂时无法在飞书文档外展示此内容
Perf
Perf 是用来进行软件性能分析的工具。
通过它，应用程序可以利用 PMU，tracepoint 和内核中的特殊计数器来进行性能统计。
它不但可以分析指定应用程序的性能问题 (per thread)，也可以用来分析内核的性能问题，当然也可以同时分析应用代码和内核，从而全面理解应用程序中的性能瓶颈。
使用 perf，可以分析程序运行期间发生的硬件事件也可以分析软件事件，比如 Page Fault 和进程切换，cache mising等。
使用 Perf 可以计算IPC，Perf 还可以替代 strace，可以添加动态内核 probe 点，还可以做 benchmark 衡量调度器的好坏
stat：显示简要的性能统计信息，包括CPU周期、指令数、缓存命中率等。
perf list：列出可用的性能事件和计数器。
record：记录性能事件并生成数据文件，用于后续分析。
top：显示正在运行的进程和函数的性能统计信息，类似于 top 命令。
report：对数据文件进行分析和报告，生成性能分析结果。
annotate：将源代码与性能分析结果关联，以便在源代码级别进行分析。
perf record -e <event>：记录指定的性能事件。
perf record -p <pid>：记录指定进程的性能事件。
perf record -a：记录系统范围的性能事件。
perf report -n：以不依赖源代码的方式生成性能报告。

使用示例
Cache命中率
 perf stat -e l1d_load_access,l1d_load_miss,l1i_load_access,l1i_load_miss,l2_load_miss,l2_load_access -p 3014 sleep 2
 Performance counter stats for process id '3014':
        50,436,940      l1d_load_access #6.42% of all L1-dcache accesses
         3,136,603      l1d_load_miss
       768,043,161      l1i_load_access
         5,217,676      l1i_load_miss
         5,227,730      l2_load_miss #44%
        11,883,914      l2_load_access
       2.006649641 seconds time elapsed
显示当前系统的性能统计信息
用于实时显示当前系统的性能统计信息。perf top 则显示每个特定功能使用的 CPU 时间。在其默认状态下，perf top 告知您在用户空间和内核空间中的所有 CPU 之间使用的功能。该命令主要用来观察整个系统当前的状态，比如可以通过查看该命令的输出来查看当前系统最耗时的内核函数或某个用户进程
perf top -e cpu-clock

perf stat通过概括精简的方式提供被调试程序运行的整体情况和汇总数据
perf stat -p 6421
 Performance counter stats for process id '6421':

         37,059.60 msec task-clock                       #    1.354 CPUs utilized
            29,052      context-switches                 #  783.927 /sec
             1,011      cpu-migrations                   #   27.280 /sec
               141      page-faults                      #    3.805 /sec
    58,635,997,124      cycles                           #    1.582 GHz
     9,428,213,699      instructions                     #    0.16  insn per cycle
     1,000,575,638      branches                         #   26.999 M/sec
        74,397,683      branch-misses                    #    7.44% of all branches

      27.378870476 seconds time elapsed
Perf recored
perf record 记录单个函数级别的统计信息，并使用 perf report 来显示统计结果 。
参数解释：
-F 99 ：1秒钟进行采集99次；
-a：采集所有cpus信息
perf record -e cpu-clock
^C[ perf record: Woken up 72 times to write data ]
[ perf record: Captured and wrote 19.862 MB perf.data (349631 samples) ]
Perf report
执行perf record之后在当前目录下会生成一个perf.data文件，可通过perf report可以读取perf.data文件并在终端中展示。
Samples: 349K of event 'cpu-clock', Event count (approx.): 87407750000
Overhead  Command          Shared Object                    Symbol
  14.34%  swapper          [kernel.kallsyms]                [k] default_idle_cal
   3.64%  stress-ng-cpu    libgcc_s.so.1                    [.] __multf3
   2.52%  stress-ng-cpu    libm.so.6                        [.] __multf3
   1.70%  stress-ng-cpu    libgcc_s.so.1                    [.] __addtf3
   1.53%  glmark2-es2-way  libc.so.6                        [.] memmove
   1.17%  swapper          [kernel.kallsyms]                [k] finish_task_swit
   0.79%  stress-ng-cpu    libm.so.6                        [.] __addtf3
   0.74%  gnome-system-mo  libpixman-1.so.0.42.2            [.] 0x000000000002c4
   0.66%  stress-ng-cpu    libm.so.6                        [.] __subtf3
   0.57%  glmark2-es2-way  glmark2-es2-wayland              [.] WaveMesh::update
   0.52%  fio              [kernel.kallsyms]                [k] finish_task_swit
   0.42%  gnome-shell      libc.so.6                        [.] memcpy
   0.38%  mpv              [kernel.kallsyms]                [k] arch_sync_dma_fo
   0.30%  stress-ng-cpu    libm.so.6                        [.] __sqrtl_finite@G
   0.29%  stress-ng-cpu    libgcc_s.so.1                    [.] __divtf3
   0.26%  mpv/vo           [kernel.kallsyms]                [k] arch_sync_dma_fo
   0.24%  fio              fio                              [.] get_io_u
   0.23%  gnome-system-mo  libcairo.so.2.11800.0            [.] 0x000000000002de
   0.23%  chrome           libc.so.6                        [.] memcpy
   0.23%  ThreadPoolForeg  chrome                           [.] 0x0000000005457b
   0.23%  chrome           [kernel.kallsyms]                [k] finish_task_swit
bpftrace vs perf
暂时无法在飞书文档外展示此内容

文件系统常见的异常
文件系统变为只读 (Read-only Filesystem)
这是最常见的一种保护性异常。
表现：无法创建新文件、无法修改或删除现有文件，提示 Read-only file system。
排查步骤
1.  排除本次启动之前是否发生意外断电
如果发生意外断电，可手动调用对应的文件系统修复工具扫描并修复文件系统，保证元数据一致，常见的sd，u盘等支持热插拔的设备多会发生类似的情况
2.  排除是否有主存错误
可通过dmesg，查看系统log是否有除文件系统读写报错以外的存储驱动报错
解决方案
1. 手动使用fsck修复文件系统
基本命令语法：
sudo fsck [选项] <设备名>
常用选项：
- -p：自动修复所有安全的错误（不询问）。这是默认行为之一，但可能不会修复所有问题。
- -y / --yes：对所有询问自动回答 "yes"。这是最常用的强制修复选项。
- -n：只检查文件系统，而不进行任何修复。
- -f：强制检查，即使文件系统看起来是干净的。
- -v： verbose 模式，输出详细过程。
- -C：显示进度条（对 ext2/3/4 文件系统有效）
2. 如果检查log中发现存储驱动报错，如mmc0，mmc2，usb等驱动报错
建议查看对应的驱动说明文档，查看FAQ中是否有类似的问题解决方法，如果是卡或者usb等外部存储设备，建议多尝试几款，确认是否存储设备异常。
挂载失败
找不到指定的IO字符集
mount -t vfat /dev/block/vol-179-33 /mnt/data/external/8461-3631                         <
[  562.455742] FAT-fs (mmcblk0p1): IO charset iso8859-1 not found
解决方案
步骤1. 检查是否配置相应字符集
zcat /proc/config.gz | grep CONFIG_NLS_ISO8859_1
步骤2. 在内核menuconfig中配置相应字符集
-> File systems
    -> Native language support (NLS [=y])
        -> NLS ISO 8859-1 (Latin 1; Western European Languages)(NLS_IS08859_1 [=y])
磁盘空间不足 (No Space Left on Device)
表现：无法写入文件，提示 No space left on device。
成因：
  - 磁盘块 (Block) 用尽：使用 df -h 检查发现某个分区的使用率是 100%。
  - 索引节点 (Inode) 用尽：磁盘可能还有剩余空间，但存放文件元信息（如文件名、权限、时间戳）的 inode 表已满。常见于存在大量小文件（如邮件、缓存文件）的系统。使用 df -i 检查
开机或其他老化场景Oops
Oops非法指令
 [  135.238092] Oops - illegal instruction [#1]
[  135.238110] Modules linked in: husb239 typec 8852bs
[  135.238127] CPU: 3 PID: 2787 Comm: distribute#1 Not tainted 6.6.36 #1
[  135.238137] Hardware name: M1-MUSE-PAPER2 (DT)
[  135.238142] epc : close_fd_get_file+0x2/0x46
[  135.238162]  ra : __riscv_sys_close+0x10/0x60
[  135.238171] epc : ffffffff8024759a ra : ffffffff80224a84 sp : ffffffc80a05bea0
[  135.238176]  gp : ffffffff82302708 tp : ffffffd961e0b300 t0 : ffffffff80ffc052
[  135.238180]  t1 : ffffffff816011b0 t2 : ffffffff81601230 s0 : ffffffc80a05bec0
[  135.238185]  s1 : ffffffc80a05bf30 a0 : 0000000000000024 a1 : 0000000000000000
[  135.238188]  a2 : 0000000000000000 a3 : 0000000300000000 a4 : 0000000200000000
[  135.238191]  a5 : ffffffff80224a74 a6 : 0000003f8947f2d4 a7 : 0000000000000039
[  135.238196]  s2 : 0000003f8a2e2c70 s3 : 0000000000000000 s4 : 0000000000000008
[  135.238199]  s5 : 0000003f8947faf8 s6 : 0000000000000000 s7 : 0000000100000000
[  135.238202]  s8 : ffffffffffffffff s9 : 0000003f8978a010 s10: 7fffffffffffffff
[  135.238205]  s11: 8000000000000000 t3 : 0000003f8a2e2da8 t4 : 0000000000000010
[  135.238208]  t5 : 000000000147ae14 t6 : ffffffffffff7fff
[  135.238211] status: 0000000200000120 badaddr: 000000000904ff07 cause: 0000000000000002
[  135.238216] [<ffffffff8024759a>] close_fd_get_file+0x2/0x46
[  135.238225] [<ffffffff80ffc10c>] do_trap_ecall_u+0xba/0x12e
[  135.238238] [<ffffffff81005582>] ret_from_exception+0x0/0x6e
[  135.238251] Code: ff05 0a04 ff05 0a04 ff05 0904 ff07 0904 ff07 0904 (ff07) 0904
[  135.238259] ---[ end trace 0000000000000000 ]---
[  135.238263] Kernel panic - not syncing: Fatal exception in interrupt
[  135.238266] SMP: stopping secondary CPUs
[  135.392375] ---[ end Kernel panic - not syncing: Fatal exception in interrupt ]---
上述非法指令表现为在linux核心层，甚至表现为随机异常，启动多次对应的堆栈不同
排查步骤
1. 排除ddr 容量配置异常
如果是全新的板子
请确认板子上使用的ddr size是否与启动过程中现实的size一致，如果size不一致，需要烧号工具重新烧写CS值，以保证正确配置ddr
2. 排除ddr稳定性问题<如果新板子>
使用memtest压测ddr，排除ddr稳定性问题，进迭内部有闭源的裸机ddr测试工具，如果需要可以联系AE提供
3. 如果Image是手动安装或者异常前有升级
确认手动安装后是否调用reboot重启，如果存在直接掉电或者reboot -f重启，可能会导致数据在文件系统缓存中，没有被安全完整的写入主存中
4. 确认uboot加载Image地址是否有异常
如下log显示Image被加载到物理地址起始0x600000~0x2786000，建议打开cmdline中memblock=debug，查看内核预留内存是否有与这段地址冲突的。
No FDT memory address configured. Please configure
the FDT address via "fdt addr <address>" command.
Aborting!
[   1.719] Moving Image from 0x200000 to 0x600000, end=2786000
[   1.740] ## Flattened Device Tree blob at 31000000
[   1.741]    Booting using the fdt blob at 0x31000000
[   1.746]    Loading Ramdisk to 7d8b8000, end 7dd7dee7 ... OK
[   1.759]    Loading Device Tree to 000000007d89f000, end 000000007d8b793f ... OK

使能memblock调试信息的方式常用的有3种
  1. 修改bootfs分区的env_k1-x.txt，添加memblock=debug后重启
  2. 进入uboot shell，通过env set重新设置bootargs，添加memblock=debug
  3. 修改内核dtsi，在bootargs中添加memblock=debug，这个方式需要重新编译内核
解决方案
1. 如果ddr容量配置有异常，需要通过烧号工具重新写CS值，保证容量识别正常。其中写号工具集成到刷机工具中，https://developer.spacemit.com/documentation?token=O6wlwlXcoiBZUikVNh2cczhin5d 
2. 如果memtest测试有异常，检查ddr是否在进迭ddr支持列表中，如果不在，请使用在支持列表中的物料，或者将ddr型号告诉进迭，待压测入支持列表后继续使用。支持列表查询链接 https://developer.spacemit.com/documentation?token=MHrtw0FymiAJFNkRsXjc46ygnDh&type=file
3. 如果异常前存在更新并异常重启，建议通过uboot shell修改env选择上一个版本的Image加载，正常进入系统后，重新安装deb或者升级，sync整个系统后，运行reboot重启，让新的Image生效。
4. 如果Image加载地址异常
     请通过memblock debug确认冲突的地址是什么，可以通过调整Image加载地址解决问题
Bianbu OS是非fit格式加载Image，需要通过linux menuconfig配置CONFIG_IMAGE_LOAD_OFFSET调整
Symbol: IMAGE_LOAD_OFFSET [=0x600000]                                                                                                                       │
  │ Type  : hex                                                                                                                                                 │
  │ Defined at arch/riscv/Kconfig:1072                                                                                                                          │
  │   Prompt: Image load offset from start of RAM when load kernel to RAM                                                                                       │
  │   Location:                                                                                                                                                 │
  │ (1) -> Image load offset from start of RAM when load kernel to RAM (IMAGE_LOAD_OFFSET [=0x600000])  
Oops地址异常
Unable to handle kernel paging request at virtual address ffffffdb33e98040
[   87.422523] systemd-journal[382]: unhandled signal 4 code 0x1 at 0x0000003f9c21ec1e in libsystemd-shared-255.so[3f9c000000+33b000]
[   87.422561] CPU: 6 PID: 382 Comm: systemd-journal Not tainted 6.6.63 #2.2~beta1.2+20250325010730
[   87.422568] Hardware name: spacemit k1-x MUSE-Pi-Pro board (DT)
[   87.422572] epc : 0000003f9c21ec1e ra : 0000003f9c2089ce sp : 0000003fe8845980
[   87.422576]  gp : 0000002abe00e800 tp : 0000003f9bbce780 t0 : 000000003d454741
[   87.422580]  t1 : 0000003f9c06d9dc t2 : 4f4e4f4d5f454352 s0 : 0000003fe8845ab0
[   87.422584]  s1 : 000000000000000a a0 : 0000003fe8845a18 a1 : 0000003f9bbcf100
[   87.422587]  a2 : 0000000000000001 a3 : ffffffffffffffff a4 : fffffffffffff8d0
[   87.422590]  a5 : 6515380c376e55a9 a6 : 0000003f9c453000 a7 : 0000000000000000
[   87.422594]  s2 : 0000003fe8845af0 s3 : 0000003fe8845a28 s4 : 0000000000000000
[   87.422598]  s5 : 0000002abe017280 s6 : 0000003fe8845a18 s7 : 0000003f9c47dd08
[   87.422602]  s8 : 000000000534c50e s9 : 0000003fe8845a28 s10: 0000003fe8845c20
[   87.422605]  s11: 0000000000000000 t3 : 0000003f9c466aa8 t4 : ffffffffffffffff
[   87.422609]  t5 : 000000009aa8aa71 t6 : 0000003fe8847ad0
[   87.422612] status: 0000000200004020 badaddr: 0000000000000000 cause: 0000000000000002
[   87.423783] apport[2790]: unhandled signal 4 code 0x1 at 0x0000003fb82236d2 in ld-linux-riscv64-lp64d.so.1[3fb820f000+22000]
[   87.423827] CPU: 4 PID: 2790 Comm: apport Not tainted 6.6.63 #2.2~beta1.2+20250325010730
[   87.423834] Hardware name: spacemit k1-x MUSE-Pi-Pro board (DT)
[   87.423838] epc : 0000003fb82236d2 ra : 0000003fb821f9f8 sp : 0000003fc88dbc00
[   87.423843]  gp : 0000000000000000 tp : 0000000000000000 t0 : 0000000000000000
[   87.423847]  t1 : 0000003fb820d000 t2 : 0000000000000000 s0 : 0000003fc88dbc50
[   87.423852]  s1 : 0000003fb8220c10 a0 : 0000000000000000 a1 : 0000003fc88dbbec
[   87.423855]  a2 : 0000000000000000 a3 : 0000003fb8232228 a4 : 0000003fb8232ae0
[   87.423859]  a5 : 0000000000000001 a6 : 0000000000000000 a7 : 0000000000000002
[   87.423863]  s2 : 0000003fb8232d10 s3 : 0000003fb8232cd8 s4 : 0000003fb820f000
[   87.423867]  s5 : 0000000000000005 s6 : 0000000000000002 s7 : 0000003fb820f000
[   87.423872]  s8 : 0000003fb8234008 s9 : 0000000000000000 s10: 0000003fb820f388
[   87.423876]  s11: 0000000000000003 t3 : 0000000000000440 t4 : 0000000000000000
[   87.423880]  t5 : 0000000000000064 t6 : 0000000000000000
[   87.423883] status: 0000000200004020 badaddr: 0000000000000000 cause: 0000000000000002
排查步骤
1. 排除ddr 容量配置异常
如果是全新的板子
请确认板子上使用的ddr size是否与启动过程中现实的size一致，如果size不一致，需要烧号工具重新烧写CS值，以保证正确配置ddr
2. 排除ddr稳定性问题<如果新板子>
使用memtest压测ddr，排除ddr稳定性问题，进迭内部有闭源的裸机ddr测试工具，如果需要可以联系AE提供
3. 如果跑飞的堆栈是是内核原生代码
建议先开启内核内存debug，KASAN，确认是否有内存使用不当的风险，如果有驱动报错可反馈给进迭处理
解决方案
1. 如果ddr容量配置有异常，需要通过烧号工具重新写CS值，保证容量识别正常。其中写号工具集成到刷机工具中，https://developer.spacemit.com/documentation?token=O6wlwlXcoiBZUikVNh2cczhin5d 
2. 如果memtest测试有异常，检查ddr是否在进迭ddr支持列表中，如果不在，请使用在支持列表中的物料，或者将ddr型号告诉进迭，待压测入支持列表后继续使用。支持列表查询链接 https://developer.spacemit.com/documentation?token=MHrtw0FymiAJFNkRsXjc46ygnDh&type=file
3. 使能KASAN，配置KASAN=Y后编译运行
Symbol: KASAN [=n]                                                                                                                                          │
  │ Type  : bool                                                                                                                                                │
  │ Defined at lib/Kconfig.kasan:34                                                                                                                             │
  │   Prompt: KASAN: dynamic memory safety error detector                                                                                                       │
  │   Depends on: ((HAVE_ARCH_KASAN [=y] && CC_HAS_KASAN_GENERIC [=y] || HAVE_ARCH_KASAN_SW_TAGS [=n] && CC_HAS_KASAN_SW_TAGS [=n]) && CC_HAS_WORKING_NOSANITIZ │
  │   Location:                                                                                                                                                 │
  │     -> Kernel hacking                                                                                                                                       │
  │       -> Memory Debugging                                                                                                                                   │
  │ (1)     -> KASAN: dynamic memory safety error detector (KASAN [=n])                                                                                         │
  │ Selects: STACKDEPOT_ALWAYS_INIT [=n]                   
如果有遇到BUG: KASAN: global-out-of-bounds等异常，可以附上log，提单给进迭解决
KASAN对性能影响较大，仅在debug场景打开，其他场景默认关闭
swiotlb full导致部分场景异常
用于在内存超过device访问范围内，比如usb，存储等模块会使用从框架层传输过来的地址，但是硬件IP不一定能够访问这个地址空间，这个时候就需要swiotlb做一次copy，这个会影响性能。进迭预留的的swiotlb size是128M
swiotlb full 异常log
spacemit-k1xvi c0230000.vi: swiotlb buffer is full (sz: 1048576 bytes), total 65536 (slots), used 498 (slots)
排查步骤
trace event 查看都有谁使用了swiotlb？
步骤1. 挂载tracefs

步骤2. 使能swiotlb event
cd /sys/kernel/tracing/
echo 1 > events/swiotlb/enable
这个trace event可以支持过滤
步骤3. cat trace
disp_eng_share_-1439    [000] .....   197.644246: swiotlb_bounced: dev_name: d4280800.sdh dma_mask=ffffffff dev_addr=ffffffffffffffff size=4 NORMAL
disp_eng_share_-1439    [000] .....   197.644264: swiotlb_bounced: dev_name: d4280800.sdh dma_mask=ffffffff dev_addr=ffffffffffffffff size=4 NORMAL
jbd2/mmcblk2p6--289     [006] .....   197.926040: swiotlb_bounced: dev_name: d4281000.sdh dma_mask=ffffffff dev_addr=ffffffffffffffff size=4096 NORMAL
jbd2/mmcblk2p6--289     [006] .....   197.926052: swiotlb_bounced: dev_name: d4281000.sdh dma_mask=ffffffff dev_addr=ffffffffffffffff size=4096 NORMAL
jbd2/mmcblk2p6--289     [006] .....   197.926055: swiotlb_bounced: dev_name: d4281000.sdh dma_mask=ffffffff dev_addr=ffffffffffffffff size=4096 NORMAL
jbd2/mmcblk2p6--289     [006] .....   197.926059: swiotlb_bounced: dev_name: d4281000.sdh dma_mask=ffffffff dev_addr=ffffffffffffffff size=4096 NORMAL
kworker/6:1H-76      [006] .....   197.930374: swiotlb_bounced: dev_name: d4281000.sdh dma_mask=ffffffff dev_addr=ffffffffffffffff size=4096 NORMAL
ksdioirqd/mmc1-1416    [005] .....   199.158983: swiotlb_bounced: dev_name: d4280800.sdh dma_mask=ffffffff dev_addr=ffffffffffffffff size=4 NORMAL
ksdioirqd/mmc1-1416    [005] .....   199.159042: swiotlb_bounced: dev_name: d4280800.sdh dma_mask=ffffffff dev_addr=ffffffffffffffff size=4 NORMAL
ksdioirqd/mmc1-1416    [005] .....   199.159071: swiotlb_bounced: dev_name: d4280800.sdh dma_mask=ffffffff dev_addr=ffffffffffffffff size=212 NORMAL
ksdioirqd/mmc1-1416    [005] .....   199.159111: swiotlb_bounced: dev_name: d4280800.sdh dma_mask=ffffffff dev_addr=ffffffffffffffff size=4 NORMAL
disp_eng_share_-1439    [000] .....   199.653920: swiotlb_bounced: dev_name: d4280800.sdh dma_mask=ffffffff dev_addr=ffffffffffffffff size=4 NORMAL

上述trace event可以动态检测swiotlb的使用动态，包括谁用的，使用了多少
解决方案
报错的驱动是否设置dma_mask？如果驱动不显式的设置coherent_dma_mask()或者dma_set_mask_and_coherent()，系统会按照32bit默认配置的，如果驱动与其他模块存在数据传输，当有其他模块数据传输时，系统检测到驱动dma能力在能力以外，会默认使用swiotlb内存进行copy，并占用128M的 swiotlb内存。
如驱动新增

+  ret = dma_set_mask_and_coherent(&pdev->dev, DMA_BIT_MASK(33));
+  if (ret)
+      return ret;
