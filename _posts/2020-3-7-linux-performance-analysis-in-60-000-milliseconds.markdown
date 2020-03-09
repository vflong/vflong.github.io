---
layout: post
title:  "【译文】60,000 毫秒内的 Linux 性能分析"
date:   2020-3-7 21:18:01 +0800
categories: sre linux
---

您登录到具有性能问题的 Linux 服务器：第 1 分钟要检查什么？

在 Netflix 我们有大量 EC2 Linux 云服务器，数不清的性能分析工具用于监控分析它的性能。其中包括用于云监控的 [Altas](https://medium.com/@Netflix_Techblog/introducing-atlas-netflixs-primary-telemetry-platform-bd31f4d8ed9a) 和用于按需实例分析的 [Vector](https://medium.com/@Netflix_Techblog/introducing-vector-netflixs-on-host-performance-monitoring-tool-c0d3058c3f6f)。尽管这些工具能够帮助我们解决大部分问题，但我们有时候需要登录实例并运行一些标准的 Linux 性能分析工具。

# 前 60 秒：总结

在本文中，Netflix Performance Engineering 团队将使用您应该可用的标准 Linux 工具，在命令行中向您展示优化性能调查的前 60 秒。在 60 秒内，您可以通过运行以下 10 个命令来全面了解系统资源的使用情况和运行进程。寻找错误和饱和度指标，因为它们既易于解释，又易于资源利用。饱和度是资源资源负载超出其处理能力的地方，并且可以作为请求队列的长度或等待时间来公开。

```bash
uptime
dmesg | tail
vmstat 1
mpstat -P ALL 1
pidstat 1
iostat -xz 1
free -m
sar -n DEV 1
sar -n TCP,ETCP 1
top
```

其中一些命令需要安装 sysstat 软件包。这些命令公开的度量标准将帮助您完成 [USE 方法](http://www.brendangregg.com/usemethod.html)：一种定位性能瓶颈的方法。这涉及检查所有资源（CPU、内存、磁盘等）的利用率，饱和度和错误度量。还应注意检查和免除资源的时间，因为通过消除过程，这会缩小研究目标的范围，并指导后续调查。

以下各节通过生产系统中的实例总结了这些命令。有关这些工具的更多信息，请参考其 man 手册。

# 1. uptime 

```bash
$ uptime 
23:51:26 up 21:31, 1 user, load average: 30.02, 26.43, 19.02
```

这是查看平均负载的快速方法，该平均负载指示要运行任务（进程）的数量。在 Linux 系统上，这些数字包括要在 CPU 上运行的进程以及在不间断 I/O（通常是磁盘 I/O）中阻塞的进程。这提供了资源负载（或需求）的高级概念，但是没有其他工具就无法正确理解。值得一看。

这 3 个数字是具有 1 分钟、5 分钟和 15 分钟常数的指数阻尼移动求和平均值。这 3 个数字使我们对负载如何随时间变化有所了解。例如，如果要求您检查问题服务器，而 1 分钟的值比 15 分钟的值低很多，则您可能登录得太晚而错过了问题。

在上面的示例中，平均负载显示了最近的增加，1 分钟值达到 30，而 15 分钟值达到 19。数字如此之大意味着很多：可能是 CPU 需求；vmstat 或 mpstat 将确认，依次为命令 3 和 4。

# 2. dmesg | tail

```bash
$ dmesg | tail
[1880957.563150] perl invoked oom-killer: gfp_mask=0x280da, order=0, oom_score_adj=0
[...]
[1880957.563400] Out of memory: Kill process 18694 (perl) score 246 or sacrifice child
[1880957.563408] Killed process 18694 (perl) total-vm:1972392kB, anon-rss:1953348kB, file-rss:0kB
[2320864.954447] TCP: Possible SYN flooding on port 7001. Dropping request.  Check SNMP counters.
```

如果有消息，它将查看最近的 10 条系统消息。查找可能导致性能问题的错误。上面的示例包括 oom-killer 和 TCP 丢弃请求。

不要错过这一步！dmesg 始终值得检查。

# 3. vmstat 1

```bash
$ vmstat 1
procs ---------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
34  0    0 200889792  73708 591828    0    0     0     5    6   10 96  1  3  0  0
32  0    0 200889920  73708 591860    0    0     0   592 13284 4282 98  1  1  0  0
32  0    0 200890112  73708 591860    0    0     0     0 9501 2154 99  1  0  0  0
32  0    0 200889568  73712 591856    0    0     0    48 11900 2459 99  0  0  0  0
32  0    0 200890208  73712 591860    0    0     0     0 15898 4840 98  1  1  0  0
^C
```

vmstat（8）是虚拟内存状态的缩写，是一种常用工具（几十年前首次为 BSD 创建）。它在每一行上打印关键服务统计信息的总结。

vmstat 使用参数 1 运行，用来显示 1 秒钟的总结。输出的第一行（在此版本的 vmstat 中）具有一些列，这些列显示自启动以来的平均值，而不是前一列。现在请跳过第一行，除非您想学习并记住哪一列是哪一列。

**要检查的列：**

* **r**：在 CPU 上运行并等待转向的进程数。由于它不包括 I/O，因此它比确定 CPU 饱和的平均负载提供更好的信号。解释：大于 CPU 技术的“r”值是饱和度。
* **free**：可用内存（以千字节为单位）。如果要计数的位数太多，则您有足够的可用内存，包括在命令 7 中的“free -m”命令可以更好地说明空闲内存的状态。
* **si, so**：Swap-ins 和 swap-outs。如果这些为非零值，则表示您的内存不足。
* **us, sy, id, wa, st**：这些是平均所有 CPU 的 CPU 时间细分。它们分别是用户时间、系统时间（内核）、空闲、等待 I/O 和被盗时间（其他 guest 账户，或使用 guest 商户自身隔离的驱动程序域 Xen）。

通过计算用户 + 系统时间，CPU 时间细分将确认 CPU 是否繁忙。恒定程度的等待 I/O 指向磁盘瓶颈，这是 CPU 空闲的地方，因为任务被等待磁盘 I/O 的进程阻塞。您可以将等待 I/O 视为 CPU 空闲的另一种形式，它提供了有关为什么它们处于空闲状态的线索。

I/O 处理需要系统时间。更高的平均系统时间（超过 20%）可能会很有趣，需要进一步研究：也许内核在低效地处理 I/O。

在上面的示例中，CPU 时间几乎完全处于用户级别，并指向应用程序级别的使用。CPU 的平均利用率也远远超过 90%。这并不一定是问题；使用“r”列检查饱和度。

# 4. mpstat -P ALL 1

```bash
$ mpstat -P ALL 1
Linux 3.13.0-49-generic (titanclusters-xxxxx)  07/14/2015  _x86_64_ (32 CPU)

07:38:49 PM  CPU   %usr  %nice   %sys %iowait   %irq  %soft  %steal  %guest  %gnice  %idle
07:38:50 PM  all  98.47   0.00   0.75    0.00   0.00   0.00    0.00    0.00    0.00   0.78
07:38:50 PM    0  96.04   0.00   2.97    0.00   0.00   0.00    0.00    0.00    0.00   0.99
07:38:50 PM    1  97.00   0.00   1.00    0.00   0.00   0.00    0.00    0.00    0.00   2.00
07:38:50 PM    2  98.00   0.00   1.00    0.00   0.00   0.00    0.00    0.00    0.00   1.00
07:38:50 PM    3  96.97   0.00   0.00    0.00   0.00   0.00    0.00    0.00    0.00   3.03
[...]
```

此命令显示每个 CPU 的 CPU 时间明细，可用于检查不平衡情况。单个热 CPU 可以证明是单线程应用程序。

# pidstat 1

```bash
$ pidstat 1
Linux 3.13.0-49-generic (titanclusters-xxxxx)  07/14/2015    _x86_64_    (32 CPU)

07:41:02 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
07:41:03 PM     0         9    0.00    0.94    0.00    0.94     1  rcuos/0
07:41:03 PM     0      4214    5.66    5.66    0.00   11.32    15  mesos-slave
07:41:03 PM     0      4354    0.94    0.94    0.00    1.89     8  java
07:41:03 PM     0      6521 1596.23    1.89    0.00 1598.11    27  java
07:41:03 PM     0      6564 1571.70    7.55    0.00 1579.25    28  java
07:41:03 PM 60004     60154    0.94    4.72    0.00    5.66     9  pidstat

07:41:03 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
07:41:04 PM     0      4214    6.00    2.00    0.00    8.00    15  mesos-slave
07:41:04 PM     0      6521 1590.00    1.00    0.00 1591.00    27  java
07:41:04 PM     0      6564 1573.00   10.00    0.00 1583.00    28  java
07:41:04 PM   108      6718    1.00    0.00    0.00    1.00     0  snmp-pass
07:41:04 PM 60004     60154    1.00    4.00    0.00    5.00     9  pidstat
^C
```

Pidstat 有点像 top 的每个进程总结，但是会打印滚动总结，而不是清除屏幕。这对于观察一段时间内的模式以及将所看到的内容（拷贝并粘贴）记录到调查记录中很有用。

上面示例将两个 Java 进程标识为消耗 CPU 的主要进程。%CPU 列是所有 CPU 的总数；1591% 表明 Java 进程消耗了将近 16 个 CPU。

# 6. iostat -xz 1

```bash
$ iostat -xz 1
Linux 3.13.0-49-generic (titanclusters-xxxxx)  07/14/2015  _x86_64_ (32 CPU)

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
          73.96    0.00    3.73    0.03    0.06   22.21

Device:   rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
xvda        0.00     0.23    0.21    0.18     4.52     2.08    34.37     0.00    9.98   13.80    5.42   2.44   0.09
xvdb        0.01     0.00    1.02    8.94   127.97   598.53   145.79     0.00    0.43    1.78    0.28   0.25   0.25
xvdc        0.01     0.00    1.02    8.86   127.79   595.94   146.50     0.00    0.45    1.82    0.30   0.27   0.26
dm-0        0.00     0.00    0.69    2.32    10.47    31.69    28.01     0.01    3.23    0.71    3.98   0.13   0.04
dm-1        0.00     0.00    0.00    0.94     0.01     3.78     8.00     0.33  345.84    0.04  346.81   0.01   0.00
dm-2        0.00     0.00    0.09    0.07     1.35     0.36    22.50     0.00    2.55    0.23    5.62   1.78   0.03
[...]
^C
```

这是了解设备（磁盘）、应用的工作负载的性能的绝佳工具。查找：

* **r/s, w/s, rkB/s, wkB/s**：这些是每秒向设备交付的读取、写入、读取千字节和写入千字节。使用这些表征工作负载。性能问题可能仅仅是由于施加了过多的负载。
* **await**：I/O 平均时间（以毫秒为单位）。这是应用程序运行的时间，因为它包括排队时间和服务时间。大于预期平均时间可能表示设备饱和或设备出现问题。
* **avgqu-sz**：发给设备的平均请求数。值大于 1 可以表明已达到饱和状态（尽管设备通常可以并行处理请求，尤其是在多个后端磁盘前端的虚拟设备）。
* **%util**：设备利用率。这确实是一个繁忙的百分比，它表示设备每秒工作的时间。值大于 60% 通常会导致性能不佳（应在等待状态中看到），尽管它取决于设备。接近 100% 的值通常表示饱和。

如果存储设备是位于许多后端磁盘前面的逻辑磁盘设备，则 100% 的利用率可能仅意味着 100% 的时间正在处理某些 I/O，但是，后端磁盘可能还远远不够饱和，并且可能能够处理更多工作。

请记住，性能不佳的磁盘 I/O 不一定是应用程序的问题。通常使用许多技术来异步执行 I/O，以便应用程序不会阻塞并直接遇到延迟（例如，预读用于读取，缓冲用于写入）。

# 7. free -m

```bash
$ free -m
             total       used       free     shared    buffers     cached
Mem:        245998      24545     221453         83         59        541
-/+ buffers/cache:      23944     222053
Swap:            0          0          0
```

右边两列显示：

* **buffers**：用于缓冲区高速缓存，用于块设备 I/O。
* **cached**：对于页面缓存，由文件系统使用。

我们只想检查一下它们的大小是否接近零，这可能导致磁盘 I/O 更高（使用 iostat 进行确认），并且导致性能下降。上面的示例看起来不错，每个示例中都有许多兆字节。

“-/+ buffers/cache” 为医用和可用内存提供较少令人迷惑的值。Linux 将可用内存用于高速缓存，但是如果应用程序需要使用它可用快速回收它。因此，应以某种方式将魂村的内存包括在“空闲内存”列中（此行会这样做）。甚至还有一个关于此疑惑的网站 [linuxatemyram](http://www.linuxatemyram.com/)。

如果使用 Linux 上的 ZFS（就像某些服务一样），可能还会造成疑惑，因为 ZFS 具有自己的文件系统缓存，而 `free -m` 无法反映该文件系统。似乎系统的可用内存不足，而实际上该内存可根据需要从 ZFS 魂村中使用。

# sar -n DEV 1

```bash
$ sar -n DEV 1
Linux 3.13.0-49-generic (titanclusters-xxxxx)  07/14/2015     _x86_64_    (32 CPU)

12:16:48 AM     IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s   %ifutil
12:16:49 AM      eth0  18763.00   5032.00  20686.42    478.30      0.00      0.00      0.00      0.00
12:16:49 AM        lo     14.00     14.00      1.36      1.36      0.00      0.00      0.00      0.00
12:16:49 AM   docker0      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00

12:16:49 AM     IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s   %ifutil
12:16:50 AM      eth0  19763.00   5101.00  21999.10    482.56      0.00      0.00      0.00      0.00
12:16:50 AM        lo     20.00     20.00      3.25      3.25      0.00      0.00      0.00      0.00
12:16:50 AM   docker0      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
^C
```

使用此工具来检查网络接口吞吐量：rxkB/s 和 txkB/s，作为工作量的度量标准，还可以检查是否已达到任何限制。在上面的示例中，eth0 接收达到 22 Mbytes/s 即 176 Mbits/sec（远低于 1 Gbit/sec 的限制）。

此版本还具有 %ifutil，用于设备利用率（全双工的双向最大），这也是我们使用 Brendan 的 [nicstat 工具](https://github.com/scotte/nicstat) 进行测量的结果。与 nicstat 一样，这很难解决，并且宅本利（0.00）中似乎不起作用。

# 9. sar -n TCP,ETCP 1

这是一些关键 TCP 指标的摘要视图。这些包括：

* **active/s**：每秒本地启动的 TCP 连接数（例如，通过 connect()）。
* **passive/s**：每秒远程启动的 TCP 连接数（例如，通过 accept()）。
* **retrans/s**：每秒 TCP 重传的次数。

主动和被动计数通常可以用作服务器负载的粗略度量：新接受的连接数（被动）和下游连接数（主动）。将主动视为出站，将被动视为入站可能会有所帮助，但这并不严格（例如，将 localhost 连接到 localhost）。

重新传输是网络或服务器问题的迹象；它可能是不可靠的网络（例如，公共 Internet），也可能是由于服务器超载并丢弃了数据包。上面的示例仅显示每秒一个新的 TCP 连接。

# 10. top

```bash
$ top
top - 00:15:40 up 21:56,  1 user,  load average: 31.09, 29.87, 29.92
Tasks: 871 total,   1 running, 868 sleeping,   0 stopped,   2 zombie
%Cpu(s): 96.8 us,  0.4 sy,  0.0 ni,  2.7 id,  0.1 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem:  25190241+total, 24921688 used, 22698073+free,    60448 buffers
KiB Swap:        0 total,        0 used,        0 free.   554208 cached Mem

   PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 20248 root      20   0  0.227t 0.012t  18748 S  3090  5.2  29812:58 java
  4213 root      20   0 2722544  64640  44232 S  23.5  0.0 233:35.37 mesos-slave
 66128 titancl+  20   0   24344   2332   1172 R   1.0  0.0   0:00.07 top
  5235 root      20   0 38.227g 547004  49996 S   0.7  0.2   2:02.74 java
  4299 root      20   0 20.015g 2.682g  16836 S   0.3  1.1  33:14.42 java
     1 root      20   0   33620   2920   1496 S   0.0  0.0   0:03.82 init
     2 root      20   0       0      0      0 S   0.0  0.0   0:00.02 kthreadd
     3 root      20   0       0      0      0 S   0.0  0.0   0:05.35 ksoftirqd/0
     5 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 kworker/0:0H
     6 root      20   0       0      0      0 S   0.0  0.0   0:06.94 kworker/u256:0
     8 root      20   0       0      0      0 S   0.0  0.0   2:38.05 rcu_sched
```

top 命令包括我们之前检查的许多指标。可以很方便地运行它以查看是否有任何与以前的命令看起来截然不同，这表明负载是可变的。

不利的一面是，随着时间的推移很难看到模式，这在提供滚动输出的 vmstat 和 pidstat 之类的工具中可能更清楚。如果您没有足够快地暂停输出（Ctrl-S 暂停，Ctrl-Q 继续），并且屏幕清楚、间歇性问题的证据也可能丢失。

# 后续分析

您可以应用更多命令和方法来进行更深入的研究。请参阅 Velocity 2015 的 Brendan 的 [Linux 性能工具教程](https://medium.com/@Netflix_Techblog/netflix-at-velocity-2015-linux-performance-tools-51964ddb81cf)，该教程可处理 40 多个命令，涉及可观察性、基准测试、调优、静态性能调优和跟踪。

# 备注

* 原文：[https://netflixtechblog.com/linux-performance-analysis-in-60-000-milliseconds-accc10403c55](https://netflixtechblog.com/linux-performance-analysis-in-60-000-milliseconds-accc10403c55)

