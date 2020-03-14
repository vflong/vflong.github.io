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

了解为什么在 Linux 中进行某些更改很容易：您可以阅读相关文件的 `git commit` 历史记录，并阅读变更说明。我检查了 [loadavg.c](https://github.com/torvalds/linux/commits/master/kernel/sched/loadavg.c) 的历史记录，但是添加了不间断状态的更改早于改文件，改文件是使用较早文件的代码创建的。我检查了另一个文件，但是那条线索也不太乐观：代码本身跳到了不通的文件周围。希望借此捷径，我为整个 Linux github 存储库转储了“git log -p”，这是 4GB 的文本，并开始向后阅读查看代码何时首次出现。这也是一个死胡同。整个 Linux repo 中最早的变更可以追溯到 2005 年，当时 Linus 导入了 Linux 2.6.12-rc2，并且此更改早于此。

存在一些历史版本的 Linux repo（[这里](https://git.kernel.org/pub/scm/linux/kernel/git/tglx/history.git)和[这里](https://kernel.googlesource.com/pub/scm/linux/kernel/git/nico/archive/)），但是这些变更说明也从中丢失了。为了至少发现这种变化发生的时间，我在 [kernel.org](https://www.kernel.org/pub/linux/kernel/Historic/v0.99/) 上搜索了 tarball，发现它的变化是 0.99.15，而不是  0.99.13 —— 但是， 0.99.14 的 tarball 缺失了。我在其他地方找到了它，并确认该更改是在 1993 年 11 月的 Linux 0.99 不定程序级别 14 中进行的。我希望 Linus 对 0.99.14 的发行说明能够解释这一变更，但[那](http://www.linuxmisc.com/30-linux-announce/4543def681c7f27b.htm)也是死胡同：

> “对于之后一个正式版本（p13）的更改太多，以至于无法提及（甚至不记得）……” —— Linux

他提到了主要的变更，但是未提及平均负载的变更。

根据日期，我查询了内核邮件列表档案，以查找实际的补丁程序，但是最早的电子邮件是 1995 年 6 月，当时系统管理员写道：

> “在研究使这些邮件更有效地扩展的系统时，我无意间破坏了当前的档案（哎呀）。”

我的搜索开始受到诅咒。值得庆幸的是，我发现了一些较旧的 linux-devel 邮件列表归档文件，这些归档文件是从服务器备份中提取的，通常以摘要的压缩文件形式存储。我搜索了 6,000 多个摘要，其中包含 98,000 封电子邮件，其中 30,000 封来自 1993 年。但是不知何故，所有邮件都没有。看起来原始补丁说明可能永远消失了，“为什么”仍然是个谜。

# 不间断（uninterruptible）的起源

幸运的是，我终于在 [oldlinux.org](http://oldlinux.org/Linux.old/mail-archive/) 上的 1993 年压缩邮箱中找到了更改。如下：

```mail
From: Matthias Urlichs <urlichs@smurf.sub.org>
Subject: Load average broken ?
Date: Fri, 29 Oct 1993 11:37:23 +0200


The kernel only counts "runnable" processes when computing the load average.
I don't like that; the problem is that processes which are swapping or
waiting on "fast", i.e. noninterruptible, I/O, also consume resources.

It seems somewhat nonintuitive that the load average goes down when you
replace your fast swap disk with a slow swap disk...

Anyway, the following patch seems to make the load average much more
consistent WRT the subjective speed of the system. And, most important, the
load is still zero when nobody is doing anything. ;-)

--- kernel/sched.c.orig Fri Oct 29 10:31:11 1993
+++ kernel/sched.c  Fri Oct 29 10:32:51 1993
@@ -414,7 +414,9 @@
    unsigned long nr = 0;

    for(p = &LAST_TASK; p > &FIRST_TASK; --p)
-       if (*p && (*p)->state == TASK_RUNNING)
+       if (*p && ((*p)->state == TASK_RUNNING) ||
+                  (*p)->state == TASK_UNINTERRUPTIBLE) ||
+                  (*p)->state == TASK_SWAPPING))
            nr += FIXED_1;
    return nr;
 }
--
Matthias Urlichs        \ XLink-POP N|rnberg   | EMail: urlichs@smurf.sub.org
Schleiermacherstra_e 12  \  Unix+Linux+Mac     | Phone: ...please use email.
90491 N|rnberg (Germany)  \   Consulting+Networking+Programming+etc'ing      42
```

阅读将近 24 年前这一变化背后的想法真是太神奇了。

这证实了故意更改了平均负载以反映对其他系统资源的需求，而不仅仅是 CPU。Linux 从“CPU 平均负载”更改为“系统平均负载”。

他使用较慢的 swap 磁盘的示例很有道理：通过降低系统性能，对系统的需求（测量 running + queued 状态）可以增加。然而，平均负载下降了，因为它们仅仅跟踪 CPU 运行状态，而不跟踪 swap 状态。Matthias 认为这是非直觉的，因此，他将其修复。

# 不间断（uninterruptible）的现状

但是，Linux 的平均负载量有时是否过高，甚至超过了磁盘 I/O 所能解释的水平？是的，尽管我的猜测是这是由于使用 TASK_UNINTERRUPTIBLE 的新代码路径所致，而该路径在 1993 年不存在。在 Linux 0.99.14 中，有 13 个代码路径直接设置为 TASK_UNINTERRUPTIBLE 或 TASK_SWAPPING （swap 状态后来从 Linux 中删除了）。如今，在 Linux 4.12 中，有将近 400 个设置 TASK_UNINTERRUPTIBLE 的代码路径，包括一些锁原语。这些代码路径之一可能不应该包含在平均负载中。下次我的平均负载似乎过高时，我将查看情况是否如此以及是否可以解决。

我第一次给 Matthias 发电子邮件，问他对将近 24 年后的平均负载变化的看法。他在一个小时内做出了回应（正如我在 [Twitter](https://twitter.com/brendangregg/status/891716419892551680) 上提到的那样），并写道：

> ““平均负载”的观点是要得出一个与人的观点有关的系统繁忙程度的数字。TASK_UNINTERRUPTIBLE 表示（意味着？）该进程正在等待诸如磁盘读取之类的事情，这会增加系统负载。大量磁盘绑定系统可能非常缓慢，但 TASK_RUNNING 平均值仅为 0.1，这对任何人都无济于事。”

（如此迅速地得到回复，真的让我很高兴。谢谢！）

因此，至少考虑到 TASK_UNINTERRUPTIBLE 过去的意思，Matthias 仍然认为这是有道理的。

但是，TASK_UNITERRUPTIBLE 如今可以匹配更多的东西。我们是否应该将平均负载更改为仅 CPU 和磁盘需求？调度程序维护者 Peter Zijstra 已经给我发送了一个聪明的方法进行探索：将 task_struct->in_iowait 包含在平均负载中，而不是 TASK_UNINTERRUPTIBLE，以便它与磁盘 I/O 更紧密地匹配。但是，这引出了另一个问题，我们真正想要的是什么？我们是否要根据线程或仅对物理资源的需求来衡量对系统的需求？如果是前者，则应该包括等待不间断锁，因为这些线程对系统是必需的。它们不是空闲的。因此，也许 Linux 平均负载已经按照我们希望的方式工作了。

为了更好地理解不间断（uninterruptible）的代码路径，我想到一种方法来衡量它们的作用。然后，我们可以检查不同的示例，量化在这些示例中花费的时间，然后看看这一切是否有意义。

# 衡量不间断（uninterruptible）的任务

以下是生产服务器上的 [Off-CPU 火焰图](http://www.brendangregg.com/blog/2016-01-20/ebpf-offcpu-flame-graph.html)，显示了 60 秒，并且仅显示了内核堆栈，在这里我进行过滤以仅包括 TASK_UNINTERRUPTIBLE 状态（[SVG](http://www.brendangregg.com/blog/images/2017/out.offcputime_unint02.svg)）（译者注：下面的 svg 图片无法点击交互，请点击此处的链接或在图片上点击右键选择“在新标签页打开图片”查看交互的 svg 图片）的堆栈。它提供了许多不间断（uninterruptible）代码路径的示例：

![out.offcputime_unint02.svg](/assets/img/out.offcputime_unint02.svg)

如果您不熟悉 off-CPU 火焰图：您可以单击方框以放大，检查显示为方框的完整堆栈。x 轴大小与关闭 CPU 所花费的时间成正比，并且排序顺序（从左到右）没有实际意义。对于 off-CPU 的堆栈，颜色为蓝色（我对 CPU 上的堆栈使用暖色），并且饱和度具有随机差异以区分方框。

我使用 [bcc](https://github.com/iovisor/bcc) 的 offcputime 工具（此工具需要 Linux 4.8+ 的 eBPF 功能）和我的[火焰图](https://github.com/brendangregg/FlameGraph)软件生成了此文件：

```bash
# ./bcc/tools/offcputime.py -K --state 2 -f 60 > out.stacks
# awk '{ print $1, $2 / 1000 }' out.stacks | ./FlameGraph/flamegraph.pl --color=io --countname=ms > out.offcpu.svgb>
```

我正在使用 awk 将输出从微秒更改为毫秒。offcputime 的“--state 2”在 TASK_UNINTERRUPTIBLE 上匹配（请参阅 sched.h），这是我为此博文添加的一个选项。Facebook 的 Josef Bacik 首先使用他的 [kernelscope](https://github.com/josefbacik/kernelscope) 工具做到了这一点，该工具还使用了 bcc 和火焰图。在我的示例中，我只是显示内核堆栈，但是 offcputime.py 也支持显示用户堆栈。

对于上面的火焰图：显示 60 秒钟只有 926 毫秒用于不间断的睡眠。这仅会使我们的平均负载增加 0.015。现在是时候在某些 cgroup 路径中了，但是该服务器没有做太多的磁盘 I/O。

这是一个更有趣的示例，这次仅跨 10 秒（[SVG](http://www.brendangregg.com/blog/images/2017/out.offcputime_unint01.svg)）：

![out.offcputime_unint01.svg](/assets/img/out.offcputime_unint01.svg)

右边的宽塔在 proc_pid_cmdline_read() 中显示 `systemd-journal` （对 /proc/PID/cmdline），被阻塞，并为平均负载贡献 0.07。左侧有一个较宽的页面故障塔，该塔也以 rwsem_down_read_failed() 结尾（将平均负载加 0.23）。我已经使用火焰图搜索功能以洋红色突出显示了这些功能。这是 rwsem_down_read_failed() 的摘录：

```c
    /* wait to be given the lock */
    while (true) {
        set_task_state(tsk, TASK_UNINTERRUPTIBLE);
        if (!waiter.task)
            break;
        schedule();
    }
```

这是使用 TASK_UNINTERRUPTIBLE 的锁获取代码。Linux 具有互斥和不间断版本的互斥锁获取功能（例如，mutex_lock() 与 mutex_lock_interruptible() 以及信号量的 down() 和 down_interruptible()）。可中断版本允许任务被信号中断，然后在获取锁之前唤醒以处理该任务。不间断的锁睡眠时间通常不会增加平均负载，但是在这种情况下，它们增加了 0.30。如果这要高得多，则值得分析一下是否可以减少锁争用（例如，我将开始研究 systemd-journal 和 proc_pid_cmdline_read()！），这将提高性能并降低平均负载。

将这些代码路径包含在平均负载中是否有意义？是的，我会这样说。这些线程在工作中，碰巧阻塞了一个锁。他们不是闲着，它们是对系统的需求，尽管需要软件资源而不是硬件资源。

# 分解 Linux 平均负载

Linux 平均负载的值可以完全分解为组件吗？这是一个示例：在闲置的 8 CPU 系统上，我启动了 `tar` 来归档一些未缓存的文件。它花费几分钟的时间，其中大部分时间是读取磁盘被阻塞。以下是从 3 个不同的终端窗口收集的统计信息：

```bash
terma$ pidstat -p `pgrep -x tar` 60
Linux 4.9.0-rc5-virtual (bgregg-xenial-bpf-i-0b7296777a2585be1)     08/01/2017  _x86_64_    (8 CPU)

10:15:51 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
10:16:51 PM     0     18468    2.85   29.77    0.00   32.62     3  tar

termb$ iostat -x 60
[...]
avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           0.54    0.00    4.03    8.24    0.09   87.10

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
xvdap1            0.00     0.05   30.83    0.18   638.33     0.93    41.22     0.06    1.84    1.83    3.64   0.39   1.21
xvdb            958.18  1333.83 2045.30  499.38 60965.27 63721.67    98.00     3.97    1.56    0.31    6.67   0.24  60.47
xvdc            957.63  1333.78 2054.55  499.38 61018.87 63722.13    97.69     4.21    1.65    0.33    7.08   0.24  61.65
md0               0.00     0.00 4383.73 1991.63 121984.13 127443.80    78.25     0.00    0.00    0.00    0.00   0.00   0.00

termc$ uptime
 22:15:50 up 154 days, 23:20,  5 users,  load average: 1.25, 1.19, 1.05
[...]
termc$ uptime
 22:17:14 up 154 days, 23:21,  5 users,  load average: 1.19, 1.17, 1.06
```

我还收集了不间断状态（[SVG](http://www.brendangregg.com/blog/images/2017/out.offcputime_unint08.svg)）的 off-CPU 火焰图：

![out.offcputime_unint08.svg](/assets/img/out.offcputime_unint08.svg)

最后 1 分钟的平均负载为 1.19。让我分解一下：

* 0.33 来自 tar 的 CPU 时间（pidstat）
* 0.67 是从 tar 的不间断磁盘读取得出的（off-CPU 火焰图在 0.69 处有此推断，我怀疑是因为它稍后开始收集并跨越了稍有不同的时间范围）
* 0.04 来自其他 CPU 使用者（iostat 用户 + 系统，从 pidstat 减去 tar 的 CPU）
* 0.11 是来自内核工作人员的不间断磁盘 I/O 时间，刷新磁盘是写操作（off-CPU火焰图，左边的两个塔）

总计为 1.15。我仍然缺少 0.04，其中有些可能是舍入和测量间隔偏移误差，但是很多可能是由于平均负载是指数衰减的移动总和，而我正在使用的其他平均值（pidstat、iostat）是正常平均值。在 1.19 之前，1 分钟的平均值为 1.25，因此其中有些仍然会拖累我们。多少？根据我以前的图标，在 1 分钟标记处，指标的 62% 是从那分钟开始的，其余则是较旧的。因此 0.62 x 1.15 + 0.35 x 1.25 = 1.18。这是非常接近前面的 1.19。

这是一个系统，其中有一个线程（tar）加上更多线程（有时在内核工作线程中）正在工作，Linux 报告的平均负载为 1.19，这很有意义。如果它在测量“CPU 平均负载值”，则系统将报告 0.37（从 mpstat 的摘要中推断），这仅对 CPU 资源正确，但掩盖了一个线程需要大量工作的事实。

我希望这个例子表明数字确实意味着某些故意的事情（CPU + 不间断），并且您可以分解它们并弄清楚。

# 了解 Linux 平均负载

我一直在使用平均负载意味着平均 CPU 负载的操作系统，因此 Linux 版本一直困扰我。也许一直以来的真正问题是，“平均负载”一词与“I/O”一样模棱两可。哪种类型的 I/O？磁盘 I/O？文件系统 I/O？网络 I/O？……同样，平均负载是多少？平均 CPU 负载？系统平均负载？这样澄清一下，我就可以理解：

* 在 Linux 上，平均负载是（或试图成为）整个系统的“系统平均负载”，用于衡量正在工作并等待工作的线程数（CPU、磁盘、不间断锁）。换句话说，它测量未完全空闲的线程数。优势：包括对不同资源的需求。
* 在其他操作系统上，平均负载为“CPU 平均负载”，用于衡量 CPU 运行数量 + CPU 可运行线程的数量。有点：可以更容易理解和推理（仅适用于 CPU）。

请注意，还有另一种可能的类型：“物理资源平均负载”，它将仅包含物力资源（CPU + 磁盘）的负载。

也许有一天，我们将为 Linux 添加其他平均负载，然后用户选择他们想要使用的负载：单独的“CPU 平均负载”、“磁盘平局负载”、“网路平均负载”等。或者只是完全使用不同的指标。

# 什么是“好”或“坏”的平均负载？

![vectorloadavg.png](/assets/img/vectorloadavg.png)
> *使用现代工具测得的平均负载*


有些人发现了似乎对他们的系统和工作负载有用的价值：他们知道，当负载超过 X 时，应用程序延迟很高，客户开始抱怨。但这并没有真正的规则。

使用 CPU 平均负载，可以将该值除以 CPU 计数，然后说如果该比率超过 1.0，则说明您正在饱和运行，这可能导致性能问题。这有点模棱两可，因为它是一个长期平均值（至少 1 分钟），可以隐藏变化。一个比略为 1.5 的系统可能运行良好，而在 1.5 分钟内突然爆裂的另一个系统可能性能不佳。

我曾经管理过一个两核 CPU 的电子邮件服务器，该服务器在一天中的平均 CPU 负载为 11 到 16（比率为 5.5 到 8）。延迟是可以接受的，没有人抱怨。这是一个极端的例子：大多数系统的 load/CPU 比率仅为 2。

对于 Linux 的系统平均负载：由于它们涵盖了不同的资源类型，因此它们甚至更加模棱两可，因此您不能仅除以 CPU 数量。对于*相对*比较而言，它更有用：如果您知道系统在 20 的负载下运行良好，而现在是 40，那么该是时候深入研究其他指标以了解发生了什么。

# 更好的指标

当 Linux 平均负载增加时，您就会知道对资源（CPU、磁盘和一些锁）的需求更高，但是您不确定哪一个。您可以使用其他指标进行澄清。例如，对于 CPU：

* **每个 CPU 利用率**：例如，使用 `mpstat -P ALL 1`
* **每个进程的 CPU 利用率**：例如，`top`、`pidstat 1` 等。
* **每个线程运行队列（调度程序）延迟**：例如，在 /proc/PID/schedstats、delaystats，`perf sched`
* **CPU 运行队列延迟**：列如，在 /proc/schedstat，`perf shed`、我的 [runqlat](http://www.brendangregg.com/blog/2016-10-08/linux-bcc-runqlat.html) [bcc](https://github.com/iovisor/bcc) 工具中。
* **CPU 运行队列长度**：例如，使用 `vmstat 1` 和“r”列，或使用我的 runqlen bcc 工具。

前两个是利用率指标，后 3 个是饱和度指标。利用率指标可用于表征工作负载，饱和度指标可用于识别性能问题。最佳的 CPU 饱和度度量标准是运行队列（或调度程序）延迟的度量：任务/线程处于可运行状态但必须等待其运行的时间。这些允许您计算性能问题的严重程度，例如，线程在调度程序延迟中花费的时间百分比。相反，通过测量运行队列长度可以表明存在问题，但是估计幅度更难。

schedstats 工具在 Linux 4.6 中已成为内核可调参数（`sysctl kernel.sched_schedstats`），默认情况下已更改为关闭。延迟计算公开了与 [cpustat](https://github.com/uber-common/cpustat) 中相同的调度程序延迟指标，我只是建议也将其添加到 [htop](https://github.com/hishamhm/htop/issues/665) 中，因为将使人们更容易使用。比从（未记录的）/proc/sched_debug 输出中抓取等待时间（调度程序延迟）度量标准更容易：

```bash
$ awk 'NF > 7 { if ($1 == "task") { if (h == 0) { print; h=1 } } else { print } }' /proc/sched_debug
            task   PID         tree-key  switches  prio     wait-time             sum-exec        sum-sleep
         systemd     1      5028.684564    306666   120        43.133899     48840.448980   2106893.162610 0 0 /init.scope
     ksoftirqd/0     3 99071232057.573051   1109494   120         5.682347     21846.967164   2096704.183312 0 0 /
    kworker/0:0H     5 99062732253.878471         9   100         0.014976         0.037737         0.000000 0 0 /
     migration/0     9         0.000000   1995690     0         0.000000     25020.580993         0.000000 0 0 /
   lru-add-drain    10        28.548203         2   100         0.000000         0.002620         0.000000 0 0 /
      watchdog/0    11         0.000000   3368570     0         0.000000     23989.957382         0.000000 0 0 /
         cpuhp/0    12      1216.569504         6   120         0.000000         0.010958         0.000000 0 0 /
          xenbus    58  72026342.961752       343   120         0.000000         1.471102         0.000000 0 0 /
      khungtaskd    59 99071124375.968195    111514   120         0.048912      5708.875023   2054143.190593 0 0 /
[...]
         dockerd 16014    247832.821522   2020884   120        95.016057    131987.990617   2298828.078531 0 0 /system.slice/docker.service
         dockerd 16015    106611.777737   2961407   120         0.000000    160704.014444         0.000000 0 0 /system.slice/docker.service
         dockerd 16024       101.600644        16   120         0.000000         0.915798         0.000000 0 0 /system.slice/
[...]
```

除了 CPU 指标外，您还可以查找磁盘设备的利用率饱和度指标。我将重点放在 [USE 方法](http://www.brendangregg.com/usemethod.html) 中的此类指标上，并有一个 [Linux 清单](http://www.brendangregg.com/USEmethod/use-linux.html)。

尽管有更明确的指标，但这并不意味着平均负载没有用。它们已成功用于云计算微服务的扩展策略以及其他指标。这有助于微服务响应不同类型的负载增加、CPU 或磁盘 I/O。使用这些策略，错误地扩容（花钱）比不扩容（消耗客户）更安全，因此，希望包含更多信号。如果规模太大，我们将在第二天调试原因。

我一直使用平均负载的一个原因是它们的历史信息。如果要求我在云上检查性能不佳的实例，然后登陆并发现平均 1 分钟的负载比 15 分钟的平均负载低的多，这是一个很大的提示，我可能来不及看到性能问题的存活。但是在转向其他指标之前，我只花了几秒钟时间来考虑平均负载。

# 结论

1993 年，一位 Linux 工程师发现了一个不直观的情况，即平均负载，并且通过 3 行程序补丁将它们从“CPU 平均负载”永久更改为“系统平均负载”。他的更爱包含了处于不间断状态的任务，因此平均负载反映了对磁盘资源的需求，而不仅仅是 CPU。这些系统平均负载对工作和等待工作的线程数进行计数，并总结为指数衰减的移动求和平均值的 3 元组，其在等式中使用 1、5 和 15 分钟作为常数。这个 3 元组数字使您可以查看负载是增加还是减少，并且它们的最大值可能是它们自己的相对比较。

从那以后，不中断状态的使用在 Linux 内核中得到了发展，如今包括不间断的锁原语。如果平均负载是根据运行和等待线程的需求（而不是严格地说需要硬件资源的线程）衡量的，那么它们仍然按照我们希望的方式工作。

在这篇文章中，我挖掘了 1993 年以来的 Linux 平均负载补丁 —— 很难找到它 —— 包含了作者的原始解释。我还使用了现代 Linux 系统上的 bcc/eBPF 在不间断状态下测量了堆栈跟踪和时间，并将这段时间可视化为 off-CPU 火焰图。该可视化提供了许多不间断睡眠的示例，并且可以在需要时生成以解释异常高的平均负载。我还提出了其他指标，可以用来更详细地了解系统负载，而不是平均负载。

最后，我将引用 Linux 源码中调度程序维护者 Peter Zijlstra 的 [kernel/sched/loadavg.c](https://github.com/torvalds/linux/blob/master/kernel/sched/loadavg.c) 顶部的注释作为结尾：

```c
* 改文件包含计算全局负载平均值所需的魔术位。
* 这是一个愚蠢的数字，但人们认为它很重要好。
* 我们要竭尽全力使其在大型计算机和无滴答的内核上运行。
```

# 参考文献

* Saltzer, J., and J. Gintell. “[The Instrumentation of Multics](http://web.mit.edu/Saltzer/www/publications/instrumentation.html),” CACM, August 1970 (explains exponentials).
* Multics [system_performance_graph](http://web.mit.edu/multics-history/source/Multics/doc/privileged/system_performance_graph.info) command reference (mentions the 1 minute average).
* [TENEX](https://github.com/PDP-10/tenex) source code (load average code is in SCHED.MAC).
* [RFC 546](https://tools.ietf.org/html/rfc546) "TENEX Load Averages for July 1973" (explains measuring CPU demand).
* Bobrow, D., et al. “TENEX: A Paged Time Sharing System for the PDP-10,” Communications of the ACM, March 1972 (explains the load average triplet).
* Gunther, N. "UNIX Load Average Part 1: How It Works" [PDF](http://www.teamquest.com/import/pdfs/whitepaper/ldavg1.pdf) (explains the exponential calculations).
* Linus's email about [Linux 0.99 patchlevel 14](http://www.linuxmisc.com/30-linux-announce/4543def681c7f27b.htm).
* The load average change email is on [oldlinux.org](http://oldlinux.org/Linux.old/mail-archive/) (in alan-old-funet-lists/kernel.1993.gz, and not in the linux directories, which I searched first).
* The Linux kernel/sched.c source before and after the load average change: [0.99.13](http://kernelhistory.sourcentral.org/linux-0.99.13/?f=/linux-0.99.13/S/449.html%23L332), [0.99.14](http://kernelhistory.sourcentral.org/linux-0.99.14/?f=/linux-0.99.14/S/323.html%23L412).
* Tarballs for Linux 0.99 releases are on [kernel.org](https://www.kernel.org/pub/linux/kernel/Historic/v0.99/).
* The current Linux load average code: [loadavg.c](https://github.com/torvalds/linux/blob/master/kernel/sched/loadavg.c), [loadavg.h](https://github.com/torvalds/linux/blob/master/kernel/sched/loadavg.c)
* The bcc analysis tools includes my [offcputime](http://www.brendangregg.com/blog/2016-01-20/ebpf-offcpu-flame-graph.html), used for tracing TASK_UNINTERRUPTIBLE.
* [Flame Graphs](http://www.brendangregg.com/flamegraphs.html) were used for visualizing uninterruptible paths.

感谢 Deirdre Straughan 的编辑。

---

# 备注

* 原文：[http://www.brendangregg.com/blog/2017-08-08/linux-load-averages.html](http://www.brendangregg.com/blog/2017-08-08/linux-load-averages.html)

