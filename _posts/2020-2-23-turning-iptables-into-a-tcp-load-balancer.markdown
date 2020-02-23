---
layout: post
title:  "将 IPTables 变成 TCP 负载均衡器"
date:   2020-2-23 14:25:42 +0800
categories: sre network
---

> 在对 Linux 网络安全配置使用工具 iptables 的技术深入研究中，我们将了解为什么以及如何构建适合于处理 IoT 应用程序流量的复杂 TCP 路由器和负载均衡器。

大多数平台中的服务服务仅限于可通过 HTTP 协议访问的 Web 应用程序托管。但是在内存、CPU 或电池受限的环境中（例如 IoT 时间），人们不会使用 HTTP。通常，首选自定义，快速且轻量级的基于 TCP 的协议。

考虑一下，应用程序的“构建”和“运行”阶段与 Web 应用程序的阶段非常相似。编程语言（尤其是 NodeJS）和数据库通常也被共享。因此，PaaS 托管物联网应用程序的唯一限制因素是拥有 TCP 路由层。

此 TCP 路由层必须能够执行以下操作：

* 将原始 TCP 数据包路由到正确的应用程序
* 跨多个容器负载均衡那些连接

对于 HTTP 路由，Scalingo 使用 OpenResty。但是，它不能用于 TCP 路由（或只是我们这样认为，请参见结论）。因此，我们选择了另一种基于 iptables 的方法。

# 网络基础设施和目标

首先，让我们定义不同的网络。在本文中，我们将考虑两个不同的网络：

* 我们的公共网络：`192.168.1.0/24` 客户端所在的位置
* 我们的私有网络：`10.0.0.0/24` 服务器所在位置（托管应用程序容器）

公共网络有一个带有 IP：`192.168.1.2` 的客户端，私有网络有 3 个服务器，IP 地址分别为：`10.0.0.2`、`10.0.0.3` 和 `10.0.0.4`。

设置的最后部分是一台前端服务器，该服务器在两个网络之间建立 IP 地址为 `10.0.0.1` 和 `192.168.1.1` 的链接。

![turning-iptables-into-a-tcp-load-balancer-1](/assets/img/turning-iptables-into-a-tcp-load-balancer-1.svg)

在以下各节中，除非另有说明，否则我们将假定每个操作和命令都在前端服务器上进行。

# NAT

首先，我们尝试将所有进入 `192.168.1.1` IP 上的 TCP 端口 27017 的流量重定向到专用网络中 `10.0.0.2` 服务器的端口 1234。

这是通过称为网络地址转换（或 NAT）的过程完成的。在本文中，我们将重点介绍两种不同的 NAT 方法：DNAT 和 SNAT。

## DNAT

DNAT 方法更改 IP 和 TCP 数据包的 **Destination** 头部。

在此，应重写 IP 和 TCP 头部。因此，应将数据包的目标 IP 重写为 `10.0.0.2`，并将目标端口重写为 `1234`。

发生以下转换：

```
   PACKET RECEIVED                   PACKET FORWARDED
|---------------------|           |---------------------|
|    IP PACKET        |           |    IP PACKET        |
|                     |           |                     |
| SRC: 192.168.1.2    |           | SRC: 192.168.1.2    |
| DST: 192.168.1.1    |           | DST: 10.0.0.2       |
| |---------------|   |           | |---------------|   |
| |   TCP PACKET  |   | =(DNAT)=> | |   TCP PACKET  |   |
| | DPORT: 27017  |   |           | | DPORT: 1234   |   |
| | SPORT: 23456  |   |           | | SPORT: 23456  |   |
| | ... DATA ...  |   |           | | ... DATA ...  |   |
| |---------------|   |           | |---------------|   |
|---------------------|           |---------------------|
```

为此，我们将需要在 iptables 的 nat 表中使用 PREROUTING 链。

```bash
iptables \
  -A PREROUTING    # Append a rule to the PREROUTING chain
  -t nat           # The PREROUTING chain is in the nat table
  -p tcp           # Apply this rules only to tcp packets
  -d 192.168.1.1   # and only if the destination IP is 192.168.1.1
  --dport 27017    # and only if the destination port is 27017
  -j DNAT          # Use the DNAT target
  --to-destination # Change the TCP and IP destination header
     10.0.0.2:1234 # to 10.0.0.2:1234
```

就这样。现在，如果我们尝试连接到端口 27017 上的 iptables 主机，我们的流量将被重定向到我们的服务器。

如果我们在客户端上尝试：

```bash
user@client ~ $ echo "Hi from client" | nc 192.168.1.1 27017
```

该命令挂起，服务器不显示任何内容。

通过查看 `Server 1` 收到的数据包，我们可以看到 iptables 规则有效，并且流量已重定向到正确的目的地。

```bash
user@server-1 ~ $ tcpdump -i eth1
15:19:17.832609 IP 192.168.1.2.23456 > 10.0.0.2.1234: Flags [S],
  seq 37761180, win 29200, options [mss 1460,sackOK,
  TS val 21306607 ecr 0,nop,wscale 6], length 0
```

## SNAT

该命令挂起的原因是服务器不知道如何响应该客户端，因为源 IP 设置为不在其网络上的 `192.168.1.2`。

解决方案是还修改前端服务器上的源 IP 和源端口头部。这是使用 SNAT 方法完成的。

将发生以下转换：

```
  PACKET RECEIVED                                             PACKET FORWARDED
|-------------------|         |-------------------|         |-------------------|
|    IP PACKET      |         |     IP PACKET     |         |     IP PACKET     |
|                   |         |                   |         |                   |
| SRC: 192.168.1.2  |         | SRC: 192.168.1.2  |         | SRC: 10.0.0.1     |
| DST: 192.168.1.1  |         | DST: 10.0.0.2     |         | DST: 10.0.0.2     |
| |---------------| |         | |---------------| |         | |---------------| |
| |   TCP PACKET  | |=(DNAT)=>| |   TCP PACKET  | |=(SNAT)=>| |   TCP PACKET  | |
| | DPORT: 27017  | |         | | DPORT: 1234   | |         | | DPORT: 1234   | |
| | SPORT: 23456  | |         | | SPORT: 23456  | |         | | SPORT: 38921  | |
| | ... DATA ...  | |         | | ... DATA ...  | |         | | ... DATA ...  | |
| |---------------| |         | |---------------| |         | |---------------| |
|-------------------|         |-------------------|         |-------------------|
```

SNAT 在所有路由决定（包括我们的 DNAT 规则）做出后发生，因此我们需要在 nat 表的 `POSTROUTING` 链中添加 `SNAT` 规则。

```bash
iptables \
  -A POSTROUTING
  -t nat
  -p tcp
  -d 10.0.0.2    # Apply this rule if the packet is going to the IP 10.0.0.2
  --dport 1234   # and if the packet is going to port 1234
  -j SNAT        # Use the SNAT target
  --to-source 10.0.0.1 # To change the DST IP header to 10.0.0.1
```

iptables 将转换表保存在内存中，并自动处理从服务器返回的连接，并将其重定向到客户端。

通过重试上一个 `nc` 命令，我们得到：

```bash
user@client ~ $ echo "Hi from client" | nc 192.168.1.1 27017
Hi from server
```

通过查看 `Server 1` 收到的数据包，我们可以看到源 IP 和目标 IP 已被前端服务器更改。

```bash
user@server-1 ~ $ tcpdump -i eth1
15:29:37.384773 IP 10.0.0.1.38921 > 10.0.0.2.1234:
  Flags [S], seq 3215489734, win 29200, options [mss 1460,sackOK,
  TS val 21461495 ecr 0,nop,wscale 6], length 0
```

## 保护系统

iptables 通常用作防火墙。是时候使用其主要功能了，它添加了一些规则以丢弃每个未明确允许的转发数据包。

每个 iptables 链都有一个默认策略。此链中与规则不匹配的任何数据包都在使用此数据包。使用 `DROP` 默认策略，将删除任何未明确接受的连接。

```bash
iptables -t filter -P FORWARD DROP
```

先前编写的 SNAT 和 DNAT 规则仅会修改数据包头。过滤不受那些规则的影响。将默认策略设置为 DROP 后，我们现在需要显式接受来自 `Server 1` 的流量：

```bash
# Accept traffic to Server 1
iptables -t filter -A FORWARD -d 10.0.0.2 --dport 1234 -j ACCEPT
# Accept traffic from Server 1
iptables -t filter -A FORWARD -s 10.0.0.2 --sport 1234 -j ACCEPT
```

现在，我们可以将流到前端服务器的 TCP 端口 27017 的流量转发到承载**单节点应用程序**的服务器。

# 负载均衡

现在的下一步是在承载我们应用程序的多个节点之间分发连接。

为了在多个主机之间实现负载均衡，一种解决方案是更改 DNAT 规则，以便它不总是将客户端重定向到单个节点，而是将它们分布在多个节点上。

为了在 `Server 1`、`Server 2` 和 `Server 3` 之间分发这些连接，我们可以尝试定义以下规则：

```bash
iptables -A PREROUTING -t nat -p tcp -d 192.168.1.1 --dport 27017 \
         -j DNAT --to-destination 10.0.0.2:1234
iptables -A PREROUTING -t nat -p tcp -d 192.168.1.1 --dport 27017 \
         -j DNAT --to-destination 10.0.0.3:1234
iptables -A PREROUTING -t nat -p tcp -d 192.168.1.1 --dport 27017 \
         -j DNAT --to-destination 10.0.0.4:1234
```

但是 iptables 引擎是确定性的，将始终使用第一个匹配规则。在此示例中，`Server 1` 将获得所有连接。

为了解决此问题，iptables 包含一个名为 `statistic` 的模块，该模块根据某些统计条件跳过或接受规则。

statistic 模块支持两种不同的模式：

* `random`：根据概率跳过规则
* `net`：根据 round-robin 算法跳过规则

**请注意，负载均衡将仅在 TCP 协议的连接阶段进行。建立连接后，连接将始终被路由到同一服务器。**

## 随机均衡

为了真正在 3 台不同的服务器上负载均衡流量，前面的三个规则变为：

```bash
iptables -A PREROUTING -t nat -p tcp -d 192.168.1.1 --dport 27017 \
         -m statistic --mode random --probability 0.33            \
         -j DNAT --to-destination 10.0.0.2:1234
iptables -A PREROUTING -t nat -p tcp -d 192.168.1.1 --dport 27017 \
         -m statistic --mode random --probability 0.5             \
         -j DNAT --to-destination 10.0.0.3:1234
iptables -A PREROUTING -t nat -p tcp -d 192.168.1.1 --dport 27017 \
         -j DNAT --to-destination 10.0.0.4:1234
```

请注意，定义了 3 个不同的概率，而不是在各处都定义为 0.33。原因是规则是顺序执行的。

以 0.33 的概率，第一个规则将在 33％ 的时间执行，而跳过 66％ 的时间。

概率为 0.5 时，第二条规则将在 50％ 的时间执行，而跳过 50％ 的时间。但是，由于此规则位于第一个规则之后，因此将仅在 66％ 的时间执行。因此，此规则将仅应用于 `50%*66%=33%` 个请求。

由于只有 33％ 的流量达到了最后一条规则，因此必须始终应用它。

您可以根据规则（n）的数量和规则索引（i）（从 1 开始）计算具有（p = frac {1} {n-i + 1} ）。

## Round Robin

另一种方法是使用 `nth` 算法。该算法实现了 [round robin 算法](https://en.wikipedia.org/wiki/Round-robin_scheduling)。

该算法采用两个不同的参数：`every` (`n`) and `packet`(`p`)。从数据包 `p` 开始，每 `n` 个数据包将评估该规则。

要在三个不同主机之间进行负载平衡，您将需要创建这三个规则：

```bash
iptables -A PREROUTING -t nat -p tcp -d 192.168.1.1 --dport 27017 \
         -m statistic --mode nth --every 3 --packet 0              \
         -j DNAT --to-destination 10.0.0.2:1234
iptables -A PREROUTING -t nat -p tcp -d 192.168.1.1 --dport 27017 \
         -m statistic --mode nth --every 2 --packet 0              \
         -j DNAT --to-destination 10.0.0.3:1234
iptables -A PREROUTING -t nat -p tcp -d 192.168.1.1 --dport 27017 \
         -j DNAT --to-destination 10.0.0.4:1234
```

## 允许流量通过

由于我们在过滤器表的 FORWARD 链上有一个 DROP 默认策略，因此我们需要允许三个远程服务器。这可以通过 6 个 iptables 规则来完成：

```bash
iptables -t filter -A FORWARD -d 10.0.0.2 --dport 1234 -j ACCEPT
iptables -t filter -A FORWARD -d 10.0.0.3 --dport 1234 -j ACCEPT
iptables -t filter -A FORWARD -d 10.0.0.4 --dport 1234 -j ACCEPT
iptables -t filter -A FORWARD -s 10.0.0.2 --sport 1234 -j ACCEPT
iptables -t filter -A FORWARD -s 10.0.0.3 --sport 1234 -j ACCEPT
iptables -t filter -A FORWARD -s 10.0.0.4 --sport 1234 -j ACCEPT
```

现在，如果我们的客户端尝试联系我们的应用程序，我们将从客户端那里获得以下输出：

```bash
user@client ~ $ echo "Hi from client" | nc 192.168.1.1 27017
Hi from 10.0.0.2
user@client ~ $ echo "Hi from client" | nc 192.168.1.1 27017
Hi from 10.0.0.3
user@client ~ $ echo "Hi from client" | nc 192.168.1.1 27017
Hi from 10.0.0.4
user@client ~ $ echo "Hi from client" | nc 192.168.1.1 27017
Hi from 10.0.0.2
[...]
```

# 结论

在本文中，我们看到了如何基于 iptables 和 Linux 内核构建 TCP 负载平衡器。我们使用此方法来创建当前在生产 IoT 应用程序中使用的 TCP 网关。

# 备注

* 原文：[https://scalingo.com/blog/iptables](https://scalingo.com/blog/iptables)




