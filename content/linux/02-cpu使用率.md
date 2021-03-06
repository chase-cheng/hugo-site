---
title: "cpu使用率"
date: 2018-12-10T10:37:23+08:00
weight: 3
---

## cpu 使用率
将每个 CPU 的时间划分为很短的时间片，再通过调度器轮流分配给各个任务使用，造成多任务同时运行的错觉。
CPU 使用率，就是除了空闲时间外的其他时间占总 CPU 时间的百分比。
为了维护 CPU 时间，Linux 通过事先定义的节拍率（内核中表示为 HZ），触发时间中断，并使用全局变量 Jiffie 记录了开机以来的节拍数。每发生一次时间中断，Jiffies 的值就加1。

### 查看节拍率
```
$ grep 'CONFIG_HZ=' /boot/config-$(uname -r)
CONFIG_HZ=1000
```
### 查看cpu状态
```
$ cat /proc/stat |grep cpu
cpu  47628241 157 12557227 250314602 125526 0 121792 0 0 0
cpu0 7849077 35 2058248 41707790 26909 0 28698 0 0 0
cpu1 8340822 33 2201009 41244155 17465 0 9942 0 0 0
cpu2 8205061 16 2167119 41373587 21730 0 28459 0 0 0
cpu3 7875954 21 2079928 41848888 17409 0 8379 0 0 0
cpu4 7752912 25 2046167 41974655 21071 0 22868 0 0 0
cpu5 7604412 24 2004754 42165524 20939 0 23445 0 0 0
```

- /proc/stat 提供的是系统的 CPU 和任务统计信息
- /proc/[pid]/stat 展示进程的CPU和任务统计信息

### 查看cpu使用率
- top 显示了系统总体的 CPU 和内存使用情况，，以及各个进程的资源使用情况
- ps 则只显示了每个进程的资源使用情况。

### 性能分析工具 perf
性能分析工具给出的都是间隔一段时间的平均 CPU 使用率，所以要注意间隔时间的设置。

```
$ perf top
Samples: 175K of event 'cpu-clock', Event count (approx.): 14453304010
Overhead  Shared Object                     Symbol                                                                                                           ◆
   4.34%  [kernel]                          [k] __do_softirq                                                                                                 ▒
   3.61%  libpython2.7.so.1.0               [.] PyEval_EvalFrameEx                                                                                           ▒
   2.78%  [kernel]                          [k] _raw_spin_unlock_irqrestore                                                                                  ▒
   1.88%  [kernel]                          [k] __d_lookup_rcu                                                                                               ▒
   1.56%  [kernel]                          [k] finish_task_switch                                                                                           ▒
   1.17%  rngd                              [.] 0x00000000000028cb
   ...
```

- Samples: 采样数
- event: 事件类
- Event count: 事件总数量


- Overhead 是该符号的性能事件在所有采样中的比例，用百分比来表示。
- Shared 是该函数或指令所在的动态共享对象（Dynamic Shared Object），如内核、进程名、动态链接库名、内核模块名等。
- Object ，是动态共享对象的类型。比如 [.] 表示用户空间的可执行程序、或者动态链接库，而 [k] 则表示内核空间。
- Symbol 是符号名，也就是函数名。当函数名未知时，用十六进制的地址来表示。

```
# -g 开启调用关系分析，-p 指定进程号
$ perf top -g -p 21515

# 记录性能事件，等待大约 15 秒后按 Ctrl+C 退出
$ perf record -g

# 查看报告
$ perf report

```

### execsnoop
execsnoop 就是一个专为短时进程设计的工具。它通过 ftrace 实时监控进程的 exec() 行为，并输出短时进程的基本信息，包括进程 PID、父进程 PID、命令行参数以及执行的结果。   
execsnoop 所用的 ftrace 是一种常用的动态追踪技术，一般用于分析 Linux 内核的运行时行为。



### 小结
1. CPU 使用率是最直观和最常用的系统性能指标，更是我们在排查性能问题时，通常会关注的第一个指标。所以我们更要熟悉它的含义，尤其要弄清楚用户（%user）、Nice（%nice）、系统（%system） 、等待 I/O（%iowait） 、中断（%irq）以及软中断（%softirq）这几种不同 CPU 的使用率。比如说：
* 用户 CPU 和 Nice CPU 高，说明用户态进程占用了较多的 CPU，所以应该着重排查进程的性能问题。
* 系统 CPU 高，说明内核态占用了较多的 CPU，所以应该着重排查内核线程或者系统调用的性能问题。
* I/O 等待 CPU 高，说明等待 I/O 的时间比较长，所以应该着重排查系统存储是不是出现了 I/O 问题。
* 软中断和硬中断高，说明软中断或硬中断的处理程序占用了较多的 CPU，所以应该着重排查内核中的中断服务程序。
碰到 CPU 使用率升高的问题，你可以借助 top、pidstat 等工具，确认引发 CPU 性能问题的来源；再使用 perf 等工具，排查出引起性能问题的具体函数。

2. 碰到常规问题无法解释的 CPU 使用率情况时，首先要想到有可能是短时应用导致的问题，比如有可能是下面这两种情况。
* 第一，应用里直接调用了其他二进制程序，这些程序通常运行时间比较短，通过 top 等工具也不容易发现。
* 第二，应用本身在不停地崩溃重启，而启动过程的资源初始化，很可能会占用相当多的 CPU。   
对于这类进程，我们可以用 pstree 或者 execsnoop 找到它们的父进程，再从父进程所在的应用入手，排查问题的根源。

