---
layout: post
title:  "【译文】Kubernetes 中的长连接伸缩和负载均衡"
date:   2020-2-21 18:20:34 +0800
categories: sre k8s
---

> Kubernetes 不能处理长连接的负载均衡，并且某些 Pod 可能比其他 Pod 收到更多的请求。如果您使用 HTTP/2、gRPC、RSockets、AMQP 或任何其他长连接（例如数据库连接），则可能需要考虑客户端负载均衡。

Kubernetes 提供了两种方便的抽象来部署应用程序：Service 和 Deployment。

Deployment 描述了在给定时间应运行哪种类型以及多少个应用程序副本的方法。

每个应用程序都部署为 Pod，并为其分配了 IP 地址。

另一方面，Service 类似于负载均衡器。

它们旨在将流量分配给一组 Pod。

1. 在此图中，您有一个包含 3 个实例的应用程序和一个负载均衡器。
![kubernetes-long-lived-connections-1](/assets/img/kubernetes-long-lived-connections-1.svg)

1. 负载均衡器被称为 Service，并具有 IP 地址。任何传入的请求都会分发到其中一个 Pod。
![kubernetes-long-lived-connections-2](/assets/img/kubernetes-long-lived-connections-2.svg)

1. Deployment 定义了创建更多相同 Pod 实例的方法。您几乎永远不会单独部署 Pod。
![kubernetes-long-lived-connections-3](/assets/img/kubernetes-long-lived-connections-3.svg)

1. 为 Pod 分配了 IP 地址。
![kubernetes-long-lived-connections-4](/assets/img/kubernetes-long-lived-connections-4.svg)

将 Service 视为 IP 地址的集合通常很有用。

每次您对 Serivce 发出请求时，都会从该列表中选择一个 IP 地址并将其用作目的地。

1. 想象一下，向 Service 发出了诸如 `curl 10.96.45.152` 之类的请求。
![kubernetes-long-lived-connections-5](/assets/img/kubernetes-long-lived-connections-5.svg)

1. 该 Service 选择 3 个 Pod 之一作为目的地。
![kubernetes-long-lived-connections-6](/assets/img/kubernetes-long-lived-connections-6.svg)

1. 流量转发到该实例。
![kubernetes-long-lived-connections-7](/assets/img/kubernetes-long-lived-connections-7.svg)

如果您有两个应用程序（例如前端和后端），则可以为每个应用程序使用 Deployment 和 Service，然后将它们部署在集群中。

当前端应用发起请求时，不需要知道有多少 Pod 连接到后端服务。

它可能是一个 Pod，数十个或数百个。

前端应用程序也不知道后端应用程序的各个 IP 地址。

当它想要发出请求时，该请求将发送到 IP 不变的后端服务。

1. 红色 Pod 向内部（米色）组件发出请求。红色 Pod 不会选择 Pod 之一作为目的地，而是向 Service 发出请求。
![kubernetes-long-lived-connections-8](/assets/img/kubernetes-long-lived-connections-8.svg)

1. Service 选择准备就绪的 Pod 之一作为目的地。
![kubernetes-long-lived-connections-9](/assets/img/kubernetes-long-lived-connections-9.svg)

1. 流量从红色 Pod 流向浅棕色 Pod。
![kubernetes-long-lived-connections-10](/assets/img/kubernetes-long-lived-connections-10.svg)

1. 请注意，红色 Pod 不知道 Service 背后隐藏了多少个 Pod。
![kubernetes-long-lived-connections-11](/assets/img/kubernetes-long-lived-connections-11.svg)

但是，该服务的负载平衡策略是什么？

是 round-robin 算法，对吗？

差不多。

# Kubernetes Service 中的负载均衡

Kubernetes Service 并不存在。

没有进程监听 Service 的 IP 地址和端口。

> 您可以通过访问 Kubernetes 集群中的任何节点并执行 `netstat -ntlp` 来检查是否存在这种情况。

甚至在任何地方都找不到 IP 地址。

Service 的 IP 地址由控制器管理器中的控制平面分配，并存储在数据库 etcd 中。

然后，另一个组件将使用相同的 IP 地址：kube-proxy。

Kube-proxy 读取所有 Service 的 IP 地址列表，并在每个节点中写入一组 iptables 规则。

这些规则的意思是：“如果看到此 Service IP 地址，则重写请求并选择其中一个 Pod 作为目的地”。

Service IP 地址仅用作占位符 —— 这就是为什么没有进程监听 IP 地址或端口的原因。

1. 假设存在一个具有 3 个节点的集群。每个节点上都部署了一个 Pod。
![kubernetes-long-lived-connections-12](/assets/img/kubernetes-long-lived-connections-12.svg)

1. 米色的 Pod 是 Service 的一部分。Service 不存在，因此该图的组件显示灰色。
![kubernetes-long-lived-connections-13](/assets/img/kubernetes-long-lived-connections-13.svg)

1. 红色的 Pod 想要向 Service 发起请求，并最终到达其中一个米色的 Pod。
![kubernetes-long-lived-connections-14](/assets/img/kubernetes-long-lived-connections-14.svg)

1. 但是 Service 不存在。没有进程监听 Service IP 地址。它是如何工作的？
![kubernetes-long-lived-connections-15](/assets/img/kubernetes-long-lived-connections-15.svg)

1. 在从节点分发请求之前，它会被 iptables 规则拦截。
![kubernetes-long-lived-connections-16](/assets/img/kubernetes-long-lived-connections-16.svg)

1. iptables 规则知道该 Service 不存在，并继续使用属于该 Service 的 Pod 的 IP 地址之一替换该 Service 的 IP 地址。
![kubernetes-long-lived-connections-17](/assets/img/kubernetes-long-lived-connections-17.svg)

1. 该请求具有一个真实 IP 地址作为目的地，并且可以正常运行。
![kubernetes-long-lived-connections-18](/assets/img/kubernetes-long-lived-connections-18.svg)

1. 取决于您的特定的网络设施，请求最终会到达 Pod。
![kubernetes-long-lived-connections-19](/assets/img/kubernetes-long-lived-connections-19.svg)

iptables 是否使用了 round-robin 算法？

没有，iptables 主要用于防火墙，并不是被设计为提供负载均衡功能。

但是，你可以[制定一套聪明的规则，使 iptables 像负载均衡器一样工作](https://scalingo.com/blog/iptables#load-balancing)。

而这正是 Kubernetes 中发生的事情。

如果您有 3 个 Pod，则 kube-proxy 将写入以下规则：

1. 选择 Pod 1 作为目的地，可能性为 33%。否则，移至下一条规则
1. 选择 Pod 2 作为目的地，概率为 50%。否则，请移至以下规则
1. 选择 Pod 3 作为目的地（没有其他情况）

复合概率是 Pod 1、Pod 2 和 Pod 3 都有三分之一的机会（33%）被选中。

![kubernetes-long-lived-connections-20](/assets/img/kubernetes-long-lived-connections-20.svg)

同样，不能保证在 Pod 1 之后选择 Pod 2 作为目的地。

> iptables 使用带有随机模式的[统计模块](http://ipset.netfilter.org/iptables-extensions.man.html#lbCD)。因此，负载算法是随机的。

现在您已经熟悉了 Service 的工作方式，下面让我们看一下更令人兴奋的场景。

# Kubernetes 中的长连接扩展不能开箱即用

从前端到后端启动每个 HTTP 请求时，都会打开和关闭一个新的 TCP 连接。

如果前端每秒向后端发出 100 个 HTTP 请求，则同时将打开和关闭 100 个不同的 TCP 连接。

如果打开 TCP 连接并重新用于任何后续的 HTTP 请求，则可以改善延迟并节省资源。

HTTP 协议具有称为 HTTP keep-alive 或 HTTP 连接重用的功能，则该功能使用单个 TCP 连接发送和接收多个 HTTP 请求和响应。

![kubernetes-long-lived-connections-21](/assets/img/kubernetes-long-lived-connections-21.svg)

但是该功能并不是开箱即用，服务器和客户端需要配置才能使用。

修改配置本身很简单，并且可以在大多数语言和框架中使用。

以下是一些有关如何使用不同语言实现 keep-alive 的示例：

* [Node.js 中使用 keep-alive](https://medium.com/@onufrienkos/keep-alive-connection-on-inter-service-http-requests-3f2de73ffa1)
* [Sping boot 中使用 keep-alive](https://www.baeldung.com/httpclient-connection-management)
* [Python 中使用 keep-alive](https://blog.insightdatascience.com/learning-about-the-http-connection-keep-alive-header-7ebe0efa209d)
* [.NET 中使用 keep-alive](https://docs.microsoft.com/en-us/dotnet/api/system.net.httpwebrequest.keepalive?view=netframework-4.8)

*当对 Kubernetes Service 使用 keep-alive 时会发生什么？*

让我们假设前后端均支持 keep-alive。

前端只有一个实例，而后端有 3 个副本。

前端向后端发出第一个请求，并打开 TCP 连接。

该请求到达 Service，并且其中一个 Pod 被选择为目的地。

后端 Pod 回复，前端接收到响应。

但是，它不会关闭 TCP 连接，而是对后续的 HTTP 请求保持打开状态。

*当前端发出更多请求时，会发生什么？*

它们将会被发送到同一 Pod。

*iptables 不应该分配流量吗？*

应该。

打开了一个 TCP 连接，第一次使用了 iptables 规则。

3 个 Pod 中的 1 个被选为目的地。

由于所有后续请求都是通过相同的 TCP 连接传输的，因此 [iptables 不再被调用](https://scalingo.com/blog/iptables#load-balancing)。

1. 红色 Pod 向 Service 发出请求。
![kubernetes-long-lived-connections-22](/assets/img/kubernetes-long-lived-connections-22.svg)

1. 您已经知道接下来会发生什么。Service 不存在，但是 iptables 规则会拦截请求。
![kubernetes-long-lived-connections-23](/assets/img/kubernetes-long-lived-connections-23.svg)

1. 属于 Service 的 Pod 之一被选择作为目的地。
![kubernetes-long-lived-connections-24](/assets/img/kubernetes-long-lived-connections-24.svg)

1. 最终，请求到达 Pod，此时，两个 Pod 之间已建立持久连接。
![kubernetes-long-lived-connections-25](/assets/img/kubernetes-long-lived-connections-25.svg)

1. 红色 Pod 发出的任何子请求都将重用现有的开放连接。
![kubernetes-long-lived-connections-26](/assets/img/kubernetes-long-lived-connections-26.svg)

因此，您现在已经获得了更好的延迟和吞吐量，但是您失去了扩展后端的能力。

即使您有两个可用于接收来自前端 Pod 的请求的后端 Pod，也只能使用其中一个。

*这种情况可以修复吗？*

由于 Kubernetes 不知道如何对持久性连接进行负载均衡，因此您可以自己接入并对其进行修复。

Service 是 IP 地址和端口集合，通常称为端点（endpoint）。

您的应用程序可以从 Service 中检索端点列表，并决定如何分发请求。

首次尝试时，您可以打开与每个 Pod 的持久连接，并对其进行 round-robin 请求。

或者，您可以[使用更复杂的负载均衡算法](https://blog.twitter.com/engineering/en_us/topics/infrastructure/2019/daperture-load-balancer.html)。

执行负载均衡的客户端代码应遵循以下逻辑：

1. 从 Service 中检索端点列表
1. 为每一个端点打开一个连接并保持打开状态
1. 当您需要发出请求时，选择一个打开的连接
1. 定期刷新端点列表并删除或添加新连接

下面为示意图：

1. 不必让红色 Pod 向您的 Service 发出请求，您可以在客户端对请求进行负载均衡。
![kubernetes-long-lived-connections-27](/assets/img/kubernetes-long-lived-connections-27.svg)

1. 您可以编写一些代码来查询哪些 Pod 是 Service 的一部分。
![kubernetes-long-lived-connections-28](/assets/img/kubernetes-long-lived-connections-28.svg)

1. 获得该列表之后，您可以将其存储在本地并用于连接 Pod。
![kubernetes-long-lived-connections-29](/assets/img/kubernetes-long-lived-connections-29.svg)

1. 您负责控制负载均衡算法。
![kubernetes-long-lived-connections-30](/assets/img/kubernetes-long-lived-connections-30.svg)

*此问题仅适用于 HTTP keep-alive 吗？*

# 客户端负载均衡

HTTP 并不是唯一可以受益于 TCP 长连接的协议。

如果您的应用程序使用数据库，则每次您要检索记录或文档时都不会打开或关闭连接。

相反，TCP 连接仅建立一次并保持打开状态。

如果使用 Service 将数据库部署在 Kubernetes 中，则可能会遇到与上一个示例相同的问题。

数据库中有一个副本的利用率要高于其他副本。

kube-proxy 和 Kubernetes 并不能均衡持久的连接。

相反，您应该注意负载均衡对数据库的请求。

根据用于连接数据库的库，您可能有不同的选择。

以下示例来自从 Node.js 调用的 MySQL 数据库集群：

`index.js`
```js
var mysql = require('mysql');
var poolCluster = mysql.createPoolCluster();

var endpoints = /* retrieve endpoints from the Service */

for (var [index, endpoint] of endpoints) {
  poolCluster.add(`mysql-replica-${index}`, endpoint);
}

// Make queries to the clustered MySQL database
```

可以想象，其他几种协议也工作于 TCP 长连接之上。例如：

* WebSockets 和安全的 WebSockets
* HTTP/2
* gRPC
* RSockets
* AMQP

您可能认识上面的大多数协议。

*因此，如果这些协议非常流行，为什么没有标准的负载均衡解决方案呢？*

*为什么必须将逻辑移入客户端？*

*Kubernetes 中是否有原生解决方案？*

kube-proxy 和 iptables 旨在涵盖 Kubernets 集群中最流行的 deployment 用例。

但是它们大多是为了方便。

如果您使用的是公开 REST API 的 Web 服务，那么您很幸运 —— 这种情况通常不会重用 TCP 连接，并且可以使用任何 Kubernetes Service。

但是，一旦开始使用持久的 TCP 连接，就应该研究如何将负载平均分配到后端。

Kubernetes 并没有为这种情况提供开箱即用的解决方案。

然而，有些东西可能会有所帮助。

# Kubernetes 中对长连接采用负载均衡

Kubernetes 中有 4 中不同的 Service：

* ClusterIP
* NodePort
* LoadBalancer
* Headless

前 3 种服务都有一个虚拟 IP 地址， kube-proxy 使用它来创建 iptables 规则。

但是无头 Service 是所有 Service 的基本组成部分。

无头 Service 没有分配 IP 地址，而只是一种收集 Pod IP 地址和端口（也称为端点）列表的机制。

所有其他 Service 都建立在无头服务之上。

ClusterIP Service 是具有一些附加功能的无头 Service：

* 控制平面为其分配 IP 地址
* kube-proxy 遍历所有 IP 地址并创建 iptables 规则

因此，您可以一起忽略 kube-proxy，而始终使用无头 Service 收集的端点列表来平衡客户端的请求。

*但是您能想象将逻辑添加到集群中部署的所有应用程序吗？*

如果您有一个现有的应用程序群，这听起来像是不可能完成的任务。

但是还有另一种选择。

# 服务网格

您可能已经注意到，客户端负载均衡策略是非常标准的。

当应用启动时，应该

* 从 Service 中检索 IP 地址列表
* 打开并维护连接池
* 通过添加和删除端点来定期刷新池

想要提出请求时，应：

* 使用诸如 round-robin 的预定义逻辑选择可用连接之一
* 发起请求

上面的步骤对 WebSockets 连接以及 gRPC 和 AMQP 有效。

您可以将该逻辑提取到一个单独的库中，并与所有应用共享。

您可以使用 Istio 或 Linkerd 之类的服务网格来代替从头开始编写库。

服务网格通过以下新过程增强了您的应用程序；

* 自动发现 IP 地址服务
* 检查 WebSockets 和 gRPC 等连接
* 使用正确的协议进行负载均衡请求

服务网格可以帮助您管理集群内部的流量，但是它们并不是完全轻量级的。

其他选择包括选择使用诸如 Netflix Ribbon 之类的库、诸如 Envoy 之类的可编程代理，或者只是忽略它。

*如果您忽略它会怎样？*

您可以忽略负载均衡，并且仍然不会注意到任何更改。

您应该考虑几种情况。

如果您的客户端多余服务器，则应该有有限的问题。

假设您有 5 个客户端打开了到两个服务器的持久连接。

即使没有负载均衡，两个服务器也可能被利用。

![kubernetes-long-lived-connections-31](/assets/img/kubernetes-long-lived-connections-31.svg)

连接可能分布不均（也许最终有 4 个连接到同一台服务器），但是总体而言，很有可能同时利用了这两个服务器。

更有问题的相反的情况。

如果客户端更少，服务器更多，则可能有一些资源未充分利用和潜在的瓶颈。

加入有两个客户端和 5 个服务器。

最多只能打开到两个服务器的两个持久连接。

其余服务器完全不被使用。

![kubernetes-long-lived-connections-32](/assets/img/kubernetes-long-lived-connections-32.svg)

如果两个服务器无法处理客户端生成的流量，则水平扩展将无济于事。

# 总结

Kubernetes Service 旨在涵盖 Web 应用程序的最常见用途。

但是，一旦您开始使用持久性 TCP 连接的应用程序协议（例如数据库、gRPC 或 WebSockets），它们就会崩溃。

Kubernetes 不提供任何内置机制来平衡 TCP 长连接的负载。

相反，您应该对应用程序进行编码，以便它可以在客户端上游检索和负载均衡。

非常感谢 [Daniel Weibel](https://medium.com/@weibeld)、[Gergely Risko](https://github.com/errge) 和 [Salman lqbal](https://twitter.com/soulmaniqbal) 提供了一些宝贵的建议。

并感谢提出关于 iptables 规则如何工作的详细解释（和流程图）的 [ChrisHanson](https://twitter.com/CloudNativChris)。

# 备注

* 原文：https://learnk8s.io/kubernetes-long-lived-connections
