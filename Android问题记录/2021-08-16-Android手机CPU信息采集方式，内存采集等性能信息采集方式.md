## 1. CPU 通过 /proc/stat
/proc/stat 文件解析方式可以参考：http://gityuan.com/2017/08/12/proc_stat/

## 2. CPU top -n 1
可以通过 `adb shell top -n 1`
或者代码运行 `Runtime.getRuntime().exec("top -n 1");`  

结果解析


```
Tasks: 552 total,   1 running, 510 sleeping,   0 stopped,   0 zombie  
任务(进程) 系统现在共有552个进程，其中处于运行中的有1个，510个在休眠（sleep），stoped状态的有0个，zombie状态（僵尸）的有0个。

Mem:   5849960k total,  4014628k used,  1835332k free,     5756k buffers    
内存状态: 物理内存总量 (5.6G)  使用中的内存总量  空闲内存总量 缓存的内存量
1TB=1024GB ,1GB=1024MB ,1MB=1024KB ,1KB=1024字节。

Swap:  2293756k total,  1039804k used,  1253952k free,   918600k cached  
swap交换分区: 交换区总量  使用的交换区总量  空闲交换区总量  缓冲的交换区总量

如果出于习惯去计算可用内存数，这里有个近似的计算公式：
Mem的free + Mem的buffers + Swap的cached
按这个公式此台服务器的可用内存：1835332k + 5756k + 918600k = 2759688k(约2.6G)

800%cpu  13%user   0%nice  31%sys 756%idle   0%iow   0%irq   0%sirq   0%host    
cpu状态  
800%cpu -- CPU总量
13%user -- 用户空间占用CPU的百分比。
0%nice -- 改变过优先级的进程占用CPU的百分比
31%sys -- 内核空间占用CPU的百分比
756%idle -- 空闲CPU百分比
0%iow  --  IO等待占用CPU的百分比
0%irq -- 硬中断（Hardware IRQ）占用CPU的百分比
0%sirq --  软中断（Software Interrupts）占用CPU的百分比
0%host -- 


PID — 进程id
USER — 进程所有者
PR — 进程优先级
NI — nice值。负值表示高优先级，正值表示低优先级
VIRT — 进程使用的虚拟内存总量，单位kb。VIRT=SWAP+RES
RES — 进程使用的、未被换出的物理内存大小，单位kb。RES=CODE+DATA
SHR — 共享内存大小，单位kb
S — 进程状态。D=不可中断的睡眠状态 R=运行 S=睡眠 T=跟踪/停止 Z=僵尸进程
%CPU — 上次更新到现在的CPU时间占用百分比
%MEM — 进程使用的物理内存百分比
TIME+ — 进程使用的CPU时间总计，单位1/100秒
COMMAND — 进程名称（命令名/命令行）

  PID USER         PR  NI VIRT  RES  SHR S[%CPU] %MEM     TIME+ ARGS                                                 [0m
25059 shell        20   0  10M 2.4M 1.5M R 25.0   0.0   0:00.08 top
20678 u0_a21       20   0 4.5G  27M  23M S  9.3   0.4   0:02.56 com.google.android.gms.unstable
24458 root         20   0    0    0    0 S  3.1   0.0   0:01.81 [kworker/u16:4]
24367 root         20   0    0    0    0 S  3.1   0.0   0:03.28 [kworker/2:1]
 1092 system       18  -2 4.9G 184M  90M S  3.1   3.2 417:44.25 system_server
```
参考自：https://www.cnblogs.com/yc-c/p/9957959.html

## 3.内存采集
```
long var0 = Runtime.getRuntime().totalMemory();
long var2 = Runtime.getRuntime().freeMemory();
return var0 - var2;
```

## 4.# 帧率
借助 FrameCallback 和 OnDrawListener进行监控  

可以参考：  
https://vivianking6855.github.io/2018/03/05/Android-optimization-6-Block/

