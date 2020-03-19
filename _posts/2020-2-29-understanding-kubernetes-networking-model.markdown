---
layout: post
title:  "【译文】Kubernetes 网络模型指南"
date:   2020-2-29 21:42:47 +0800
categories: sre k8s
---

Kubernetes 用于在一组机器上运行分布式系统。分布式系统的本质使网络成为 Kubernetes 部署的核心和必要组成部分，并且了解 Kubernetes 网络模型将使您能够正确运行、监控和故障排除在 Kubernetes 上运行的应用程序。

网络是一个拥有许多成熟技术的广阔空间。对于不熟悉这种情况的人来说，这可能会感到不舒服，因为大多数人已经对网络有了先入为主的观念，并且有很多新旧概念需要了解并将其融合为一个整体。这个不太全面的列表可能包括诸如网络命名空间、虚拟接口、IP 转发和网络地址转换之类的技术。本指南旨在通过讨论每种与 Kubernetes 相关的技术以及如何使用这些技术来启用 Kubernetes 网络模型的描述，来揭开 Kubernetes 网络神秘感。

本指南相当长，分为几个部分。我们首先讨论一些基本的 Kubernetes 术语，以确保在整个指南中正确使用术语，然后讨论 Kubernetes 的网络模型以及设计和它所采用的实现决策。接下来是本指南中最长且最有趣的部分：使用几种不同用例深入讨论如何在 Kubernetes 中路由流量。

如果您不熟悉任何网络术语，本指南结尾附有网络术语词汇表可供查阅。

# 1 Kubernetes 基础

Kubernetes 是基于一些核心概念构建的，这些核心概念被组合为越来越强大的功能。本节列出了这些概念中的每一个，并提供了简要概述以帮助促进讨论。Kubernetes 的功能远不止这里列出的内容，但本节应作为入门，并使读者可以在后续部分中继续学习。如果您已经熟悉 Kubernetes，可以随时跳过本节。

## 1.1 Kubernetes API Server

在 Kubernetes 中，一切都是由 *Kubernetes API Server*（`kube-apiserver`）的 API 调用。API Server 是通往 [etcd](https://github.com/coreos/etcd) 数据存储的网关，该存储维护应用程序集群的期望状态。要更新 Kubernetes 集群的状态，您需要对 API Server 进行 API 调用，并描述期望状态。

## 1.2 Controllers

*Controllers* 是用于构建 Kubernetes 的核心抽象。使用 API Server 声明集群的期望状态后，控制器会不断监听 API Server 的状态并对任何更改做出响应，以确保集群的当前状态与期望状态匹配。控制器使用一个简单的循环进行操作，该循环不断地根据集群的期望状态检查集群的当前状态。如果存在任何差异，则控制器执行任务以使当前状态与期望状态匹配。用伪代码表示如下：

```java
while true:
  X = currentState()
  Y = desiredState()

  if X == Y:
    return  # Do nothing
  else:
    do(tasks to get to Y)
```

例如，当您使用 API Server 创建新的 Pod 时，Kubernetes 调度程序（控制器）会注意到更改，并决定将 Pod 放置在集群中的位置。然后，它使用 API Server（由 etcd 支持）写入状态变更。然后，`kubelet`（控制器）会注意到新的变更，并设置所需的网络功能以使 Pod 在集群中可访问。在这里，两个单独的控制器对两个单独的状态更改做出响应，以使集群的实际情况与用户的期望相匹配。

## 1.3 Pods

Pod 是 Kubernetes 的原子单位 —— 构建应用程序的最小可部署对象。单个 Pod 代表集群中正在运行的工作负载，并封装一个或多个 Docker 容器，任何必须的存储以及唯一的 IP 地址。构成 Pod 的容器设计为在同一机器上位于同一位置并进行调度。

## 1.4 Nodes

Node 节点是运行 Kubernetes 集群的机器。这些可以是裸机、虚拟机或其他任何东西。主机一词通常与节点互换使用。我将尝试使用术语“具有一致性的节点”，但有时会根据上下文使用“虚拟机”一词来指代节点。

# 2 Kubernetes 网络模型

Kubernetes 对 Pod 的网络方式进行了明智的选择。特别是，Kubernetes 对任何网络实施都规定了以下要求：

* 所有 Pod 都可以与所有其他 Pod 通信，而无需使用网络地址转换（NAT）。
* 所有节点都可以在没有 NAT 的情况下与所有 Pod 通信。
* Pod 所看到的自己的 IP 与其他 Pod 或节点看到的相同。

考虑到这些限制，我们剩下 4 个不同的网络问题要解决。

1. Container-to-Container 网络
1. Pod-to-Pod 网络
1. Pod-to-Service 网络
1. Internet-to-Service 网络

本指南的后续部分将依次讨论这些问题及其解决方案。

# 3 Container-to-Container 网络

通常，我们将虚拟机中的网络通信视为直接与以太网设备进行交互，如图 1 所示。

![understanding-kubernetes-networking-model-1](/assets/img/understanding-kubernetes-networking-model-1.png)
> 图 1，以太网设备

实际上，情况比这更微妙。在 Linux 中，每个正在运行的进程都在[网络命名空间](http://man7.org/linux/man-pages/man8/ip-netns.8.html)中进行通信，该命名空间为逻辑网络堆栈提供了自己的路由、防火墙规则和网络设备。本质上，网络命名空间为命名空间内的所有进程提供了全新的网络堆栈。

作为 Linux 用户，可以使用 `ip` 命令创建网络命名空间。例如，以下命令将创建一个名为 `ns1` 的新网络命名空间。

```bash
$ ip netns add ns1
```

创建命名空间后，将在 `/var/run/netns` 下创建该名称的挂载点，即使没有附加任何进程，命名空间也可以保留。

您可以通过列出 `/var/run/netns` 下的所有挂载点，或使用 `ip` 命令列出可用的命名空间。

```bash
$ ls /var/run/netns
ns1
$ ip netns
ns1
```

默认情况下，Linux 将每个进程分配给根（root）网络命名空间，以提供对外部环境的访问，如图 2 所示：

![understanding-kubernetes-networking-model-2](/assets/img/understanding-kubernetes-networking-model-2.png)
> 图 2，根（root）网络命名空间

就 Docker 结构而言，一个 Pod 被建模为一组共享网络命名空间的 Docker 容器。Pod 中的容器都具有通过分配给 Pod 的网络命名空间分配的相同 IP 地址和端口空间，并且可以通过 `localhost` 相互找到，因为它们位于同一命名空间中。我们可以为虚拟机上的每个 Pod 创建一个网络命名空间。这是使用 Docker 作为“Pod 容器”实现的，该容器使网络命名空间保持打开状态，而“应用程序容器”（用户指定的内容）使用 Docker 的参数 `-net=container:function` 加入该命名空间。图 3 显示每个 Pod 如何由共享命名空间中的多个 Docker 容器（`ctr*`）组成。

![understanding-kubernetes-networking-model-3](/assets/img/understanding-kubernetes-networking-model-3.png)
> 图 3，每个 Pod 的网络命名空间

Pod 中的应用程序还可以访问共享卷，这些共享卷定义为 Pod 的一部分，并且可以挂载到每个应用程序的文件系统中。

# 4 Pod-to-Pod 网络

在 Kubernetes 中，每个 Pod 都有一个真实的 IP 地址，每个 Pod 都使用该 IP 地址与其他 Pod 通信。当前的任务是了解 Kubernetes 如何使用真实 IP 启用 Pod 到 Pod 的通信，无论 Pod 是部署在集群中的同一物理节点还是不同的节点上。我们通过考虑驻留在同一台机器上的 Pod 来开始讨论，以避免通过内部网络进行跨节点通信的复杂性。

从 Pod 的角度看，它存在于自己的以太网命名空间中，该命名空间需要与同一节点上的其他网络命名空间进行通信。值得庆幸的是，可以使用 Linux [虚拟以太网设备](http://man7.org/linux/man-pages/man4/veth.4.html)或由两个虚拟接口组成的 *veth 对*来连接命名空间，这两个虚拟接口可以分布在多个命名空间上。

要连接 Pod 命名空间，我们可以将 veth 对的一侧分配给根（root）网络命名空间，另一侧分配给 Pod 的网络命名空间。每对 veth 都像跳线一样工作，将两侧连接起来并允许流量在它们之间流动。此设置可以复制到与计算机上一样多的 Pod。图 4 显示了将 VM 上的每个 Pod 连接到根（root）命名空间的 veth 对。

![understanding-kubernetes-networking-model-4](/assets/img/understanding-kubernetes-networking-model-4.png)
> 图 4，每个 Pod 中的 veth 对

至此，我们已经将 Pod 设置为每个都有自己的网络命名空间，以便它们认为自己具有自己的以太网设备和 IP 地址，并且已连接到 Node 的根（root）命名空间。

现在，我们希望 Pod 通过根（root）命名空间相互通信，为此，我们使用了网桥。

Linux 以太网桥是一种虚拟的第 2 层网络设备，用于将两个或多个网段结合在一起，透明地将两个网络连接在一起。该网桥通过维护源和目标之间的转发表实现，而转发表检查通过它的数据包的目的地，并确定是否将是否将数据包传递到连接网桥的其他网段。桥接代码通过查看网络中每个以太网设备唯一的 MAC 地址来决定桥接数据还是丢弃数据。

网桥使用 [ARP](https://en.wikipedia.org/wiki/Address_Resolution_Protocol) 协议以发现与给定 IP 地址关联的链路层 MAC 地址。当网桥接收到数据帧时，网桥会将帧广播到所有连接到的设备（原始发送方式除外），并且将对帧进行相应的的设备存储在查找表中。具有相同 IP 地址的后续流量将使用查找表来发现正确的 MAC 地址，以将数据包转发到该地址。

![understanding-kubernetes-networking-model-5](/assets/img/understanding-kubernetes-networking-model-5.png)
> 图 5，使用桥接连接命名空间

## 4.1 数据包的生命周期：Pod-to-Pod，同一节点

给定将每个 Pod 隔离到自己的网络堆栈的网络命名空间，将每个命名空间连接到根（root）命名空间的虚拟以太网设备，以及将命名空间连接在一起的网桥，我们终于可以在同一节点上的 Pod 之间发送流量了。如图 6 所示：

![understanding-kubernetes-networking-model-6](/assets/img/understanding-kubernetes-networking-model-6.gif)
> 图 6，数据包在同一节点上的 Pod 之间传输

在图 6 中，Pod 1 将数据包发送到其自己的以太网设备 `eth0`，该设备可作为 Pod 的默认设备。对于 Pod 1，`eth0` 通过虚拟以太网设备连接到根（root）命名空间 `veth0`（1）。网桥 `cbr0` 配置有 `veth0` 附加到它的网段。数据包到达网桥后，网桥解析正确的网络段，以使用 ARP 协议（3）将数据包发送到 `veth1`。当数据包到达虚拟设备 `veth1` 时，它将直接转发到 Pod 2 的命名空间和该命名空间中的 `eth0` 设备（4）。在整个流量中，每个 Pod 仅与 `localhost` 上的 `eth0` 通信，并且流量被路由到正确的 Pod。使用网络的开发经验是开发人员期望的默认行为。

Kubernetes 的网络模型规定 Pod 必须通过其跨节点的 IP 地址才能访问。即，一个 Pod 的 IP 地址始终对网络中的其他 Pod 可见，并且每个 Pod 都将自己的 IP 地址视为与其他 Pod 看到的 IP 地址相同。现在我们来讨论在不同节点上的 Pod 之间路由流量的问题。

## 4.2 数据包的生命周期：Pod-to-Pod，跨节点

在确定了如何在同一节点上的 Pod 之间路由数据包之后，我们继续在不同节点上的 Pod 之间路由流量。Kubernetes 网络模型要求 Pod IP 可以在整个网络上访问，但并未指定必须如何完成。实际上，这是特定于网络的，但是已经建立了一些模式来简化此过程。

通常集群中的每个节点都分配有一个 CIDR 块，该块指定了该节点上运行 Pod 可用的 IP 地址。一旦发往 CIDR 区块的流量到达节点，则节点有责任将流量转发到正确的 Pod。图 7 说明了两个节点之间的流量流，*假设网络可以将 CIDR 块中的流量路由到正确的节点*。

![understanding-kubernetes-networking-model-7](/assets/img/understanding-kubernetes-networking-model-7.gif)
> 图 7，数据包在不同节点上的 Pod 之间传输

图 7 从与图 6 相同的请求开始，除了目标 Pod（以绿色突出显示）与源 Pod（以蓝色突出显示）位于不同的 Node 上。数据包首先通过 Pod 1 的以太网设备发送，该设备与根（root）命名空间（1）中的虚拟以太网设备配对。最终，数据包最终到达根（root）命名空间的网络桥（2）。ARP 将在网桥上失败，因为没有与数据包的 MAC 地址匹配的设备连接到该网桥。失败之后，网桥会将数据包发送出默认路由 —— 根（root）命名空间的 `eth0` 设备。此时，路由离开节点并进入网络（3）。现在我们假设网络可以根据分配给节点（4）的 CIDR 块将数据包路由到正确的节点。数据包进入目标节点的根（root）命名空间（VM 2 上的 `eth0`），在此它通过网桥路由到正确的虚拟以太网设备（5）。最后，通过流经 Pod 4 命名空间（6）中的虚拟以太网设备对来完成路由。一般而言，每个节点都知道如何将数据包传递到其中运行的 Pod。数据包到达目标节点后，数据包的流动方式与在同一节点上的 Pod 之间路由流量的方式相同。

为了便于理解，我们回避了如何配置网络，以将 Pod IP 的流量路由到负责这些 IP 的正确节点。这是特定于网络的，但是查看特定示例将提供有关所涉及问题的一些见解。例如，对于 AWS，Amazon 为 Kubernetes 维护了一个容器网络插件，该插件允许使用[容器网络接口（CNI）插件](https://github.com/aws/amazon-vpc-cni-k8s)在 Amazon VPC 环境中运行节点到节点网络。

容器网络接口（CNI）提供了用于将容器连接到外部网络的通用 API。作为开发人员，我们想知道 Pod 可以使用 IP 地址与网络通信，并且我们希望此操作的机制透明。由 AWS 开发的 CNI 插件试图满足这些需求，同时通过 AWS 提供的现有 VPC，IAM 和安全组功能提供安全和可管理的环境。解决方案是使用弹性网络接口。

在 EC2 中，每个实例都绑定到一个弹性网络接口（ENI），并且所有 ENI 都连接在 VPC 内 —— ENI 可以相互访问，而无需付出额外的努力。默认情况下，每个 EC2 实例都部署有一个 ENI，但是您可以随意创建多个 ENI 并将它们部署到您认为合适的 EC2 实例中。用于 Kubernetes 的 AWS CNI 插件通过为部署到节点的每个 Pod 创建新的 ENI 来利用这种灵活性。由于 VPC 中的 ENI 已在现有的 AWS 基础设施中连接，因此，每个 Pod 的 IP 地址都可以在 VPC 中本地寻址。将 CNI 插件部署到集群后，每个节点（EC2 实例）都会创建多个弹性网络接口，并为这些实例分配 IP 地址，从而为每个节点形成一个 CIDR 块。部署 Pod 时，作为 DaemonSet 部署到 Kubernetes 集群的小型二进制文件会收到来自节点本地 `kubelet` 进程的将 Pod 添加到网络的任何请求。该二进制文件从 Node 的可用 ENI 池中选择一个可用 IP 地址，并通过连接虚拟以太网设备和 Linux 内核中的网桥，并将其分配给 Pod，如在同一 Node 中对 Pod 进行网络连接时所述。有了这个，Pod 流量就可以跨集群中的节点进行路由。

# 5 Pod-to-Service 网络

我们已经展示了如何在 Pod 及其关联的 IP 地址之间的路由流量。在我们需要应对变化之前，这非常有效。Pod IP 地址不是持久性的，并且会随着扩容或缩容，应用程序崩溃或节点重启而出现并消失。这些事件中的每一个都可以使 Pod IP 地址更改而不会发出警告。*Service* 已内置到 Kubernetes 中以解决此问题。

Kubernetes *Service* 管理一组 Pod 的状态，使您可以跟踪随时间动态变化的一组 Pod IP 地址。Service 充当 Pod 的抽象，并为一组 Pod IP 分配一个虚拟 IP 地址。寻址到 Service 的虚拟 IP 的任何流量都将被路由到与虚拟 IP 关联的 Pod 集。这样一来，与 Service 相关联的 Pod 集群可以随时更改 —— 客户端只需要知道 Service 的虚拟 IP（不变）即可。

创建新的 Kubernetes Service 时，将代表您创建一个新的的虚拟 IP（也称为 cluster IP）。在集群中的任何位置，寻址到虚拟 IP 的流量都将负载均衡到与该 Service 关联的一组后端 Pod。实际上，Kubernetes 会自动创建并维护一个分布式集群内的负载均衡器，该负载均衡器会将流量分配给与 Service 相关的健康 Pod。让我们仔细看看它是如何工作的。

（译者注：在使用 Spring Cloud 的微服务架构中，可以考虑使用 `Service` 取代 `Netflix Eureka`）

## 5.1 netfilter 和 iptables

为了在集群内执行负载均衡，Kubernetes 依赖于 Linux 内置的网络框架 `netfilter`。Netfilter 是 Linux 提供的框架，它允许以自定义处理程序的形式实现各种与网络相关的操作。Netfilter 提供了各种功能和操作，用于数据包过滤，网络地址转换和端口转换，这些功能和操作提供了通过网络定向数据包所需的功能，以及提供禁止数据包到达计算机网络内敏感位置的功能。

`iptables` 是一个用户空间程序，它提供了一个基于表的系统，用于定义使用 netfilter 框架处理和转换数据包的规则。在 Kubernetes 中，iptables 规则由 kube-proxy 控制器配置，该控制器监视 Kubernetes API Server 的更改。当对 Service 或 Pod 的更改更新了服务的虚拟 IP 地址或 Pod 的 IP 地址时，将更新 iptables 规则以将针对 Service 的流量正确地路由到后备 Pod。iptables 规则会监视发往 Service 虚拟 IP 的流量，并在匹配时监视，从可用 Pod 的集合中选择一个随机 Pod IP 地址，并且 iptables 规则将数据包的目标 IP 地址从 Service 的虚拟 IP 更改为所选 Pod 的 IP。随着 Pod 启停，iptables 规则集将更新以反映集群状态的变化。换句话说，iptables 已在计算机上完成了负载均衡，以将定向到 Service IP 的流量传输到实际 Pod 的 IP。

在返回的路径上，IP 地址来自目标 Pod。在这种情况下，iptables 再次重写 IP 表头，以用该 Service 的 IP 替换 Pod IP，以便 Pod 认为它一直在与该服务的 IP 进行通信。

## 5.2 IPVS

Kubernetes 的最新版本（1.11）加入了用于集群内负载均衡的第二个选项：IPVS。IPVS（IP 虚拟服务器），也建立在 netfilter 之上，并作为 Linux 内核的一部分实现传输层负载均衡。IPVS 被集成到 LVS（Linux 虚拟服务器）中，在此服务器上运行，并充当真实服务器集群之前的负载均衡器。IPVS 可以将对基于 TCP 和 UDP 的服务的请求定向到真实服务器，并使真实服务器的服务在单个 IP 地址上显示为虚拟服务。这使得 IPVS 非常适合 Kubernetes Service。

在声明 Kubernetes 服务时，您可以指定是否要使用 iptables 或 IPVS 实现集群内负载均衡。IPVS 专为负载均衡而设计，并使用更有效的数据结构（哈希表），与 iptables 相比，几乎可以无限扩展。创建使用 IPVS 的 Service 负载均衡时，会发生三件事：

1. 在节点上创建虚拟 IPVS 接口
1. 将服务的 IP 地址绑定到虚拟 IPVS 接口
1. 并为每个服务 IP 地址创建 IPVS 服务器

将来，希望 IPVS 成为集群内负载均衡的默认方式。此更改仅影响集群内的负载均衡，在本指南的后续部分中，您可以使用 IPVS 安全地替换 iptables 以进行集群内的负载均衡，而不会影响后续的讨论。

## 5.3 数据包的生命周期：Pod-to-Service

![understanding-kubernetes-networking-model-8](/assets/img/understanding-kubernetes-networking-model-8.gif)
> 图 8，数据包在 Pod 和 Service 之间传输

在 Pod 和 Service 之间路由数据包时，旅程以与以前相同的方式开始。数据包首先通过连接到 Pod 网络命名空间（1）的 `eth0` 接口离开 Pod。然后，它通过虚拟以太网设备到达网桥（2）。在网桥上运行的 ARP 协议不了解服务，因此它通过默认路由 `eth0`（3）传送数据包。在这里，发生了一些不同的事情。在被 `eth0` 接受之前，该数据包通过 iptables 进行过滤。收到数据包之后， iptables 会使用 kube-proxy 安装在节点上的规则来响应服务或 Pod 事件，以将数据包的目标从 Service IP 重写到特定的 Pod IP（4）。现在，数据包将到达 Pod 4，而不是 Service 的虚拟 IP。iptables 充分利用了 Linux 内核的 `conntrack` 实用程序，以记住做出的 Pod 选择，以便将来的流量被路由到相同的 Pod（不包括任何扩缩容事件）。本质上，iptables 直接在节点上完成了集群内负载均衡。然后，使用我们已经检查过的 Pod-to-Pod 路由将流量流到 Pod。

## 5.4 数据包的生命周期：Service-to-Pod

![understanding-kubernetes-networking-model-9](/assets/img/understanding-kubernetes-networking-model-9.gif)
> 图 9，数据包在 Service 和 Pod 之间传输

接收到此数据包的 Pod 将作出响应，将源 IP 标识为自己的 IP，将目标 IP 标识为最初发送该数据包的 Pod（1）。进入节点后，数据包流经 iptables，后者使用 `conntrack` 记住先前所做的选择，并将数据包的源重写为 Service 的 IP，而不是 Pod 的 IP（2）。数据包从此处通过网桥流到与 Pod 的命名空间配对的虚拟以太网设备（3），再到我们之前看到的 Pod 的以太网设备（4）。

## 5.5 使用 DNS

Kubernetes 可以选择使用 DNS，以避免必须将 Service 的 Cluster IP 地址硬编码到您的应用程序中。Kubernetes DNS 作为在集群上计划的常规 Kubernetes 服务运行。它配置在每个节点上运行的 `kubelet`，以便容器使用 DNS 服务的 IP 来解析 DNS 名称。为集群中定义的每个服务（包括 DNS 服务器本身）分配一个 DNS 名称。DNS 记录根据您的需要将 DNS 名称解析为 Service 的 cluster IP 或 Pod 的 IP。SRV 记录用于指定运行 Service 的特定命名端口。

DNS Pod 由 3 个单独的容器组成：

* `kubedns`：监视 Kubernetes master 节点以了解 Service 和 Endpoint 的更改，并维护内存中的查找结构以服务 DNS 请求。
* `dnsmasq`：添加 DNS 缓存以提高性能。
* `sidecar`：提供单个运行状态检查点，以执行 `dnsmasq` 和 `kubedns` 的运行状况检查。

DNS Pod 本身作为 Kubernetes Service 暴露，具有静态 cluster IP，该 IP 在启动时会传递给每个正在运行的容器，以便每个容器都可以解析 DNS 条目。通过维护内存中 DNS 表示形式的 `kubedns` 系统解析 DNS 条目。`etcd` 是用于集群状态的后端存储系统，而 `kubedns` 使用一个库，该库在必要时将 `etcd` 键值存储转换为 DNS 整体，以重建内存中 DNS 查找结构的状态。

CoreDNS 的工作方式与 `kubedns` 相似，但其使用的插件体系结构使其更加灵活。从 Kubernetes 1.11 开始，CoreDNS 是 Kubernetes 的默认 DNS 实现。

# 6 Internet-to-Service 网络

到目前为止，我们已经研究了如何在 Kubernetes 集群中路由流量。一切都很好，但不幸的是，与外界隔离你的应用程序无济于事，无法实现任何销售目标 —— 有时您将需要向外部流量公开您的服务。这种需求突出了两个相关的问题：

1. 将来自 Kubernetes Service 的流量引出到 Internet
1. 将来自 Internet 的流量引入到您的 Kubernetes Service

本节将依次处理这些问题。

## 6.1 Egress —— 将流量路由到 Internet

从节点到公共 Internet 的流量路由是特定于网络的，并且实际上取决于网络配置为发布流量的方式。为了使本节更具体，我将使用 AWS VPC 讨论任何特定细节。

在 AWS 中，Kubernetes 集群在 VPC 内运行，其中为每个节点分配了一个私有 IP 地址，该地址可从 Kubernetes 集群内访问。要使流量可以从集群外部访问，请将 Internet 网关连接到 VPC。Internet 网关有两个目的：在 VPC 路由表中为可路由到 Internet 的流量提供目标，并对已分配了公共 IP 地址的任何实例执行网络地址转换（NAT）。NAT 转换负责将节点专用于集群的内部 IP 地址更改为公共 Internet 上可用的外部 IP 地址。

有了 Internet 网关后，VM 可以自由地将流量路由到 Internet。不幸的是，仍然存在一个小问题。Pod 拥有自己的 IP 地址，该 IP 地址与托管 Pod 的节点的 IP 地址不同，并且 Internet 网关上的 NAT 转换仅适用于 VM IP 地址，因为它不了解哪些 Pod 在哪些 VM 上运行 —— 网关并不知道容器的存在。让我们看看 Kubernetes 如何使用 iptables 解决这个问题（再次）。

## 6.1.1 数据包的生命周期：Node-to-Internet

在下图中，数据包起源于 Pod 命名空间（1），并经过连接到根（root）命名空间的 veth 对（2）。一旦进入根（root）命名空间，数据包就会从网桥移动到默认设备，因为数据包上的 IP 与连接到网桥的任何网段都不匹配。在到达根（root）命名空间的以太网设备（3）之前，iptables 会处理数据包（3）。在这种情况下，数据包的源 IP 地址是 Pod，并且如果我们将源保留为 Pod，则 Internet 网关将拒绝它，因为网关 NAT 仅了解连接到 VM 的 IP 地址。解决方案是让 iptables 执行源 NAT（更改数据包源），以便数据包看起来是来自 VM 而不是 Pod。使用正确的源 IP 后，数据包现在可以离开 VM（4）并到达 Internet 网关（5）。Internet 网关将执行另一个 NAT，将源 IP 从 VM 内部 IP 重写为外部 IP。最终，数据包将到达公共 Internet（6）。在返回过程中，数据包遵循相同的路径，并且任何源 IP 处理都将被撤销，因此系统的每一层都将接收其能够理解的 IP 地址：节点或 VM 级别的 VM 内部，以及 Pod 命名空间内的 Pod IP。

![understanding-kubernetes-networking-model-10](/assets/img/understanding-kubernetes-networking-model-10.gif)
> 图 10，数据包从 Pod 路由到 Internet

## 6.2 Ingress —— 将 Internet 流量路由到 Kubernetes

Ingress —— 将流量引入集群 —— 是一个非常棘手的难题。同样，这特定于您正在运行的网络，但是通常，Ingress 分为两个可在网络堆栈的不同部分上运行的解决方案：

1. Service 负载均衡器
1. Ingress 控制器

### 6.2.1 第四层 Ingress：负载均衡器

创建 Kubernetes Service 时，可以选择指定一个负载均衡器来配合它。*云控制器*提供了负载均衡器的实现，该控制器知道如何为您的服务创建负载均衡器。创建 Service 后，它将为负载均衡器通告 IP 地址。作为最终用户，您可以开始将流量定向到负载均衡器，以开始与 Service 进行通信。

借助 AWS，负载均衡器可以了解其目标组中的节点，并将均衡集群中所有节点上的流量。流量到达节点后，先前在整个集群中为您的 Service 安装的 iptables 规则将确保流量到达您感兴趣的 Service 的 Pod。

### 6.2.2 数据包的生命周期：LoadBalancer-to-Service

让我们看看这在实践中是如何工作的。部署服务后，您正在使用的云供应商将为您创建一个新的负载均衡器（1）。由于负载均衡器意识不到容器的存在，因此，一旦流量到达负载均衡器，它就会分发到组成您的集群的所有 VM 上（2）。每个 VM 上的 iptables 规则会将来自负载均衡器的传入流量定向到正确的 Pod（3）—— 这些是 Service 创建期间制定的 IP 表规则，前面已经讨论过。Pod 对客户端的响应将返回 Pod 的 IP，但客户端需要具有负载均衡器的 IP 地址。如前所述，iptables 和 `conntrack` 用于在返回路径上正确重写 IP。

下图显示了托管 Pod 的 3 个 VM 前面的网络负载均衡器。传入流量（1）指向 Service 的负载均衡器。一旦负载均衡器接收到数据包（2），它就会随机选择一个 VM。在这种情况下，我们从病理上选择了没有运行 Pod 的 VM2（3）。此时，在 VM 上运行的 iptables 规则将使用通过 kube-proxy 安装到集群中的内部负载均衡规则将数据包定向到正确的 Pod。iptables 执行正确的 NAT，并将数据包转发到正确的 Pod（4）。

![understanding-kubernetes-networking-model-11](/assets/img/understanding-kubernetes-networking-model-11.gif)
> 图 11，数据包从 Internet 发送到 Service

### 6.2.3 第七层 Ingress：Ingress Controller

第 7 层网络入口在网络堆栈的 HTTP/HTTPS 协议范围内运行，并建立在服务之上。启用 Ingress 第一步是使用 Kubernetes 中的 `NodePort` Service 类型在 Service 上打开端口。如果将 Service 的类型字段设置为 NodePort，Kubernetes master 将在您指定的范围内分配一个端口，并且每个节点都会将该端口（每个节点上的相同端口号）代理到您的 Service 中。也就是说，使用 iptables 规则，任何定向到 Node 端口的流量都将转发到该 Service。此 Service 到 Pod 路由遵循了将流量从 Service 路由到 Pod 时已经讨论过的相同内部集群负载均衡模式。

要将节点的端口暴露到 Internet，请使用 Ingress 对象。Ingress 是将 HTTP 请求映射到 Kubernetes Service 的高级 HTTP 负载均衡器。Ingress 方法将有所不同，具体取决于 Kubernetes 云供应商控制器如何实现。HTTP 负载均衡器（如第 4 层网络负载均衡器）仅了解节点 IP（而非 Pod IP），因此流量路由类似地利用 kube-proxy 在每个节点上安装的 iptables 顾泽提供的内部负载均衡。

在 AWS 环境中，ALB Ingress Controller 使用 Amazon 的 7 层应用程序负载均衡器提供 Kubernetes Ingress。下图详细说明了此 Controller 创建的 AWS 组件。它还演示了 Ingress 流量从 ALB 到 Kubernetes 集群的路线。

![understanding-kubernetes-networking-model-12](/assets/img/understanding-kubernetes-networking-model-12.png)
> 图 12，Ingess Controller 设计图

创建后，（1）Ingress Controller 将监听来自 Kubernetes API Server 的 Ingress 事件。当发现满足其要求的 Ingress 资源时，它将开始创建 AWS 资源。AWS 将应用程序负载均衡器（ALB）（2）用于 Ingress 资源。负载均衡器与用于将请求路由到一个或多个已注册节点目标组一起工作。（3）在 AWS 中为 Ingress 资源描述的每个唯一的 Kubernetes Service 创建目标组。（4）侦听器是 ALB 进程，它使用您配置的协议和端口检查连接请求。侦听器是由 Ingress 控制器为 Ingress 资源注解中详细说明的每个端口创建的。最后，为 Ingress 资源中指定的每个路径创建目标组规则。这样可以确保将到特定路径的流量路由到正确的 Kubernetes Service（5）。

### 6.2.4 数据包的生命周期：Ingress-to-Service

流经 Ingress 的数据包的生命周期与 LoadBalancer 生命周期非常相似。主要区别在于 Ingress 知道 URL 的路径（允许并可以根据其路径将流量路由到服务），并且 Ingress 和 Node 之间的初始连接是通过 Node 上每个 Service 公开的端口进行的。

让我们看看这在实践中是如何工作的。部署服务后，您正在使用的云供应商将为您创建一个新的 Ingress 负载均衡器（1）。由于负载均衡器意识不到容器的存在，因此，一旦流量到达负载均衡器，它将通过为您的 Service 广播的端口在组成您的集群（2）的所有 VM 中进行分配。如前所述，每个 VM 上的 iptables 规则会将来自负载均衡器的传入流量定向到正确的 Pod（3）。Pod 对客户端的响应将返回 Pod 的 IP，但客户端需要具有负载均衡器的 IP 地址。如前所述，iptables 和 `conntrack` 用于在返回路径上正确重写 IP。

![understanding-kubernetes-networking-model-13](/assets/img/understanding-kubernetes-networking-model-13.gif)
> 图 13，数据包从 Ingress 流向 Service

第 7 层负载均衡器的优点之一是它们可以识别 HTTP，因此它们知道 URL 和路径。这使您可以按照 URL 路径细分 Service 流量。它们通常还会在 HTTP 请求的 `X-Forworded-For` 头部中提供原始客户端的 IP 地址。

# 7 总结

本指南为了解 Kubernetes 网络模型及其如何实现常见网络任务提供了基础。网络领域既广泛又深入，不可能涵盖这里的所有内容。本指南应为您提供一个起点，让您深入了解感兴趣的主题并想了解更多有关的主题。每当您陷入困境时，都可以利用 Kubernetes 文档和 Kubernetes 社区来帮助您找到解决的方法。

# 8 术语表

Kubernetes 依靠几种现有技术来构建可运行的集群。全面探索每种技术不在本指南的讨论范围内，但是本节将对每种技术进行足够详细的介绍，以供讨论之用。您可以随意浏览、完全跳过本节或者在感到困惑或需要复习时根据需要引用它。

## 第 2 层网络

第 2 层是提供 Node-to-Node 数据传输的数据链路层。它定义了两个在物理连接的设备之间建立和终止连接的协议。它还定义了它们之间的流量控制协议。

## 第 4 层网络

传输层通过流控制来控制给定链路的可靠性。在 TCP/IP 中，此层是指用于在不可靠的网络上交换数据的 TCP 协议。

## 第 7 层网络

应用层是最接近最终用户的层，这意味着应用层和用户都直接与应用程序交互。该层与实现通信组件的应用程序交互。通常，第 7 层网络是指 HTTP。

## NAT —— 网络地址转换

NAT 或 *网络地址转换* 是将一个地址空间的 IP 转换为另一个地址空间的 IP。映射是通过在数据包通过流量路由设备传输时修改其 IP 报头中的网络地址信息实现的。

NAT 的基本功能是从一个 IP 地址到另一个 IP 地址的简单映射。更常见的场景是，NAT 用于将多个私有 IP 地址映射到一个公网 IP 地址。通常，本地网络使用私有 IP 地址空间，并且该网络上的路由器会在该空间中获得私有地址。然后，路由器将使用公网 IP 地址连接到 Internet。当流量从本地网络传递到 Internet 时，每个数据包的源地址都是从私有地址转换为公网地址，从而使请求似乎是直接来自路由器。路由器保持连接跟踪，以将答复转发到本地网络上的正确私有地址。

NAT 的另一个好处是允许大型专用网络使用单个公网 IP 地址连接到 Internet，从而节省了公网 IP 地址的数量。

### SNAT —— 源网络地址转换

SNAT 指仅修改 IP 数据包的源地址的 NAT 过程。这是上述 NAT 的典型行为。

### DNAT —— 目的网络地址转换

DNAT 是指修改 IP 数据包的目的地址的 NAT 过程。DNAT 用于将驻留在私有网络中的服务发布到可公开寻址的 IP 地址。

## 网络命名空间（Network Namespace）

在网络中，每台计算机（真实或虚拟）都具有一个以太网设备（我们将其称为 `eth0`）。流入和流出计算机的所有流量都与该设备关联。实际上，LInux 将每个以太网设备与一个*网络命名空间*相关联 —— 整个网络堆栈的逻辑副本，以及它自己的路由、防火墙规则和网络设备。最初，所有进程都从 init 进程共享相同的默认网络命名空间，称为根（root）命名空间。默认情况下，进程从其父级继承其网络命名空间，因此，如果不进行任何更改，所有网络流量都将流经为根（root）网络命名空间指定的以太网设备。

## veth —— 虚拟以太网设备对

计算机系统通常由一个或多个与物理网络适配器相关联的联网设备 eth0、eth1 等组成，该物理网络是配置负责将数据包放置到物理线路上。Veth 设备是始终在互连中创建的虚拟网络设备。它们可以充当网络命名空间之间的隧道，以创建到另一个命名空间中的物理网络设备的桥，但也可以用作独立的网络设备。您可以将 veth 设备视为设备之间的虚拟跳线 —— 从一端结束然后从另一端流出。

## bridge —— 网桥

网桥是一种从多个通信网络或网段创建单个聚合网络的设备。桥接两个独立的网络，就像它们是一个网络一样。网桥使用内部数据结构来记录每个数据包发送的位置，以优化性能。

## CIDR —— 无类域间路由

CIDR 是分配 IP 地址和执行 IP 路由的方法。对于 CIDR，IP 地址由两部分组成：网络前缀（标识整个网络或子网）和主机标识符（指定该网络或子网中主机的特定接口）。CIDR 使用 CIDR 表示法表示 IP 地址，其中地址或路由前缀带有后缀，以指示该前缀的位数，例如 IPv4 的 192.0.2.0/24。IP 地址是 CIDR 块的一部分，如果该地址的前 n 位和 CIDR 前缀相同，则称该 IP 地址属于 CIDR 块。

## CNI —— 容器网络接口

CNI（容器网络接口）是一个云原生计算基金会项目，由一个规范和库组成，用于编写在 Linux 容器中配置网络接口的插件。CNI 仅涉及容器的网络连接以及删除容器时删除分配的资源。

## VIP —— 虚拟 IP 地址

虚拟 IP 地址（即 VIP）是软件定义的 IP 地址，与实际的物理网络接口不符。

## netfilter —— Linux 的数据包过滤框架

netfilter 是 Linux 中的数据包过滤框架。实现此框架的软件负责数据包过滤，网络地址转换（NAT）和其他数据包处理。

netfilter、ip_tables、连接跟踪（ip_conntrack、nf_conntrack）和 NAT 子系统共同构成了框架的主要部分。

## iptables —— 数据包处理工具

iptables 是一个程序，允许 Linux 系统管理员配置 netfilter 以及它存储的链和规则。IP 表中的每个规则都由多个分类器（iptables 匹配项）和一个连接的动作（iptables 目标）组成。

## conntrack —— 连接跟踪

conntrack 是在 Netfilter 框架之上构建的用于处理连接的跟踪的工具。连接跟踪允许内核跟踪所有逻辑网络连接或会话，并将每个连接或会话的数据包定向到正确的发送者或接受者。NAT 依靠此信息以相同的方式转换所有相关的数据包，并且 iptables 可以使用此信息充当有状态防火墙。

## IPVS —— IP 虚拟服务器

IPVS 作为 Linux 内核的一部分实现了传输层负载均衡。

IPVS 是类似于 iptables 的工具。它基于 Linux 内核的 netfilter 钩子函数，但使用哈希表作为基础数据结构。这意味着，与 iptables 相比，IPVS 可以更快地重定向流量，在同步代理规则时具有更好的性能，并提供更多的负载均衡算法。

## DNS —— 域名系统

域名系统（DNS）是用于将系统名称与 IP 地址相关的分散式命名系统。它将域名转换为数字 IP 地址以定位计算机服务。

# 备注

* 原文：[https://sookocheff.com/post/kubernetes/understanding-kubernetes-networking-model/](https://sookocheff.com/post/kubernetes/understanding-kubernetes-networking-model/)

