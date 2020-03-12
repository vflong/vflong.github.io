---
layout: post
title:  "【译文】Linux 平均负载：解决这个奥秘"
date:   2020-3-12 21:44:35 +0800
categories: sre linux
---

平均负载是一项行业关键指标，我的公司基于这些指标和其他指标花费了数百个自动扩展云实例，但是在 Linux 上却存在一些神秘之处。Linux 平均负载不仅跟踪可运行的任务，而且还跟踪处于不间断睡眠状态的任务。为什么？我从未见过任何解释。在这篇文章中，我将解决这个奥秘，并总结平均负载来为每个试图解释平均负载的人做个参考。

Linux 平均负载是“系统平均负载”，它显示系统上正在运行的线程（任务）的需求为运行线程和等待线程的平均数量。这个可以衡量需求，该需求可能大于系统当前正在处理的请求。大多数工具显示 3 个平均值，1、5、15 分钟：

```bash
$ uptime
 16:48:24 up  4:11,  1 user,  load average: 25.25, 23.40, 23.46

top - 16:48:42 up  4:12,  1 user,  load average: 25.25, 23.14, 23.37

$ cat /proc/loadavg 
25.72 23.19 23.35 42/3411 43603
```

一些解释：

* 如果平均值为 0.0，则您的系统处于空闲状态。
* 如果 1 分钟的平均值高于 5 或 15 分钟的平均值，则负载正在增加。
* 如果 1 分钟的平均值低于 5 或 15 分钟的平均值，则负载正在减少。
* 如果他们高于您的 CPU 数量，那么您可能会遇到性能问题（取决于实际情况）。

以 3 个为 1 组，您可以判断负载是在增加还是在减少，这很有用。当需要单个需求值时（例如对于云服务自动缩放规则），它们也很有用。但是，如果没有其他指标的帮助，很难详细地了解它们。23 至 25 之间的单个值并不表示任何意义，但是如果知道 CPU 计数以及已知是 CPU 负担的工作负载，则可能表示某些含义。

我通常不尝试调试平均负载，而是切换到其他指标。我将在结尾处的“更好的指标”部分中讨论这些内容。

# 历史

原始平均负载仅显示 CPU 需求：正在运行的进程数加上正在等待运行的进程数。1973 年 8 月的 RFC 546 中有一个很好的描述，标题为“TENEX 平均负载”：

> [1] TENEX 平均负载是对 CPU 需求的度量。平均负载是给定时间段内可运行进程数的平均值。例如，每小时平均负载为 10，则意味着（对于单个 CPU 系统）在该小时内的任何时间都可以看到有 1 个进程正在运行，而有 9 个其他进程准备运行（即，没有 I/O 阻塞）等待使用 CPU。

此版本在 [ieft.org](https://tools.ietf.org/html/rfc546) 上链接到 1973 年 7 月以来的手绘平均负载图的 PDF 扫描，表明该行为受到了数十年的监视：

![rfc546.jpg](/assets/img/rfc546.jpg)
> 出处：[https://tools.ietf.org/html/rfc546](https://tools.ietf.org/html/rfc546)

如今，旧操作系统源代码也可以在网上找到。这是 [TENEX](https://github.com/PDP-10/tenex)（1970 年代初）的 SCHED.MAC 的 DEC 汇编语言的代码段：

```assembly
NRJAVS==3               ;NUMBER OF LOAD AVERAGES WE MAINTAIN
GS RJAV,NRJAVS          ;EXPONENTIAL AVERAGES OF NUMBER OF ACTIVE PROCESSES
[...]
;UPDATE RUNNABLE JOB AVERAGES

DORJAV: MOVEI 2,^D5000
        MOVEM 2,RJATIM          ;SET TIME OF NEXT UPDATE
        MOVE 4,RJTSUM           ;CURRENT INTEGRAL OF NBPROC+NGPROC
        SUBM 4,RJAVS1           ;DIFFERENCE FROM LAST UPDATE
        EXCH 4,RJAVS1
        FSC 4,233               ;FLOAT IT
        FDVR 4,[5000.0]         ;AVERAGE OVER LAST 5000 MS
[...]
;TABLE OF EXP(-T/C) FOR T = 5 SEC.

EXPFF:  EXP 0.920043902 ;C = 1 MIN
        EXP 0.983471344 ;C = 5 MIN
        EXP 0.994459811 ;C = 15 MIN
```

这是当今 [Linux](https://github.com/torvalds/linux/blob/master/include/linux/sched/loadavg.h) 源码的摘录（include/linux/sched/loadavg.h）：

```c
#define EXP_1           1884            /* 1/exp(5sec/1min) as fixed-point */
#define EXP_5           2014            /* 1/exp(5sec/5min) */
#define EXP_15          2037            /* 1/exp(5sec/15min) */
```

Linux 还对 1、5 和 15 分钟常量进行了硬编码。

在较旧的系统中也有类似的平均负载指标，例如拥有平均调度队列指数的 [Multics](http://web.mit.edu/Saltzer/www/publications/instrumentation.html) 系统。

# 3 个数字

这 3 个数字分别是 1、5 和 15 分钟的平均负载。除非它们不是真正的平均值，而且不是 1、5 和 15 分钟。从上面的源代码中可以看出，1、5 和 15 分钟是方程式中使用的常数，该方程式计算 5 秒钟平均值的指数阻尼移动的总和。所得的 1、5 和 15 分钟的平均负载反映了远超过 1、5 和 15 分钟的负载。

如果您使用一个闲置的系统，然后开始一个单线程的 CPU 约束工作负载（一个线程在一个循环中），那么 60 秒后平均 1 分钟的负载将是多少？如果它是纯平均值，则为 1.0。这是该实验，以图形方式显示：

![loadavg.png](/assets/img/loadavg.png)
> *平均负载实验以可视化指数阻尼*

所谓的“1 分钟平均值” 仅在 1 分钟标记处达到约 0.62。有关方程式和类似实验的更多信息，Neil Gunther 博士写了一篇有关平均负载的文章：[平均负载的工作原理](http://www.teamquest.com/import/pdfs/whitepaper/ldavg1.pdf)，并且在 [loadavg.c](https://github.com/torvalds/linux/blob/master/kernel/sched/loadavg.c) 中有许多 Linux 源代码块注释。

# Linux 不间断任务

当平均负载首次出现在 Linux 上时，它们反应了 CPU 需求，就像其他操作系统一样。但是后来 Linux 进行了更改，使其不仅包括可运行的任务，而且还包括处于不间断状态（TASK_UNINTERRUTIBLE 或 nr_uninterruptible）的任务。希望避免信号中断的代码路径使用此状态，其中包括磁盘 I/O 上阻塞的任务和一些锁。您可能之前已经看到过这种状态：它在 `ps` 和 `top` 命令的输出中显示为“D”状态。`ps`(1) 的 man 页面将其称为“不间断 sleep（通常是 IO）”。

添加不间断状态意味着 Linux 的平均负载可能会由于磁盘（或 NFS）I/O 工作负载而增加，而不仅仅是 CPU 需求。对于熟悉其他操作系统及其 CPU 平均负载的每个人来说，包括这种状态一开始都是令人困惑的。

**为什么？**到底为什么 Linux 会这样做？

关于平均负载的文章不计其数，其中许多指出了 Linux nr_uninterruptible 陷阱。但是我还没有看到有人能解释甚至是进行错误的猜测为什么要包含它。我自己的猜测是，它应该更一般地反应需求，而不仅仅是 CPU 需求。

# 搜索一个古老的 Linux 补丁