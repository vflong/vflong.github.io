---
layout: post
title:  "Kubernetes 网络模型指南"
date:   2020-2-29 21:42:47 +0800
categories: sre k8s
---

Kubernetes 旨在在一组机器上运行分布式系统。分布式系统的本质是网络成为 Kubernetes 部署的核心和必要组成部分，并且了解 Kubernetes 网络模型将使您能够正确运行、监控和故障排除在 Kubernetes 上运行的应用程序。

网络是一个拥有许多成熟技术的广阔空间。对于不熟悉这种情况的人来说，这可能会感到不舒服，因为大多数人已经对网络有了先入为主的观念，并且有很多新旧概念需要了解并将其融合为一个整体。这个不太全面的列表可能包括诸如网络名称空间、虚拟接口、IP 转发和网络地址转换之类的技术。本指南旨在通过讨论每种与 Kubernetes 相关的技术以及如何使用这些技术来启用 Kubernetes 网络模型的描述，来揭开 Kubernetes 网络神秘感。

本指南相当长，分为几个部分。

我们首先讨论一些基本的 Kubernetes 术语，以确保在整个指南中正确使用术语，然后讨论 Kubernetes 的网络模型以及它所施加的设计和实现决策。

接下来是本指南中最长，最有趣的部分：关于使用几种不同用例如何在 Kubernetes 中路由流量的深入讨论。

如果您不熟悉任何网络术语，则本指南附有网络术语词汇表。


# 1 Kubernetes 基础

Kubernetes 是基于一些核心概念构建的，这些核心概念被组合为越来越强大的功能。本节列出了这些概念中的每一个，并提供了简要概述以帮助促进讨论。Kubernetes 的功能远不止这里列出的内容，但本节应作为入门，并使读者可以在后续部分中继续学习。如果您已经熟悉 Kubernetes，请随时跳过本节。

## 1.1 Kubernetes API Server

在 Kubernetes 中，一切都是由 *Kubernetes API Server*（`kube-apiserver`）服务的 API 调用。API 服务器是通往 [etcd]() 数据存储的网关，该存储维护应用程序集群的所需状态。要更新 Kubernetes 集群的状态，您需要对 API 服务器进行 API 调用，以描述所需的状态。

## 1.2 Controllers

*Controllers* 是用于构建 Kubernetes 的核心抽象。使用 API 服务器声明集群的所需状态后，控制器会不断观察 API 服务器的状态并对任何更改做出反应，以确保集群的当前状态与所需状态匹配。控制器使用一个简单的循环进行操作，该循环不断地根据集群的期望状态检查集群的当前状态。如果存在任何差异，则控制器执行任务以使当前状态与所需状态匹配。用伪代码：

```java
while true:
  X = currentState()
  Y = desiredState()

  if X == Y:
    return  # Do nothing
  else:
    do(tasks to get to Y)
```

例如，当您使用 API 服务器创建新的 Pod 时，Kubernetes 调度程序（控制器）会注意到更改，并决定将 Pod 防止在集群中的位置。然后，它使用 API server（由 etcd 支持）写入状态更改。然后，`kubelet`（控制器）会注意到新的更改，并设置所需的网络功能以使 Pod 在集群中可访问。在这里，连个单独的控制器对两个单独的状态更改做出反应，以使集群的实际情况与用户的期望相匹配。

## 1.3 Pods

Pod 是 Kubernetes 的原子单位 —— 构建应用程序的最小可部署对象。单个 Pod 代表集群中正在运行的工作负载，并封装一个或多个 Docker 容器，任何必须的存储以及唯一的 IP 地址。构成 Pod 的容器设计为在同一机器上位于同一位置并进行调度。

## 1.4 Nodes

节点是运行 Kubernetes 集群的机器。这些可以是裸机、虚拟机或其他任何东西。主机一词通常与节点互换使用。我将尝试使用属于“具有已执行的节点”，但有时会根据上下文使用“虚拟机”一词来指代节点。

## 2 Kubernetes 网络模型

Kubernetes 对 Pod 的网络方式进行了明智的选择。特别是，Kubernetes 对任何网络实施都规定了以下要求：

* 所有 Pod 都可以与所有其他 Pod 通信，而无需使用网络地址转换（NAT）。
* 所有节点都可以在没有 NAT 的情况下与所有 Pod 通信。
* Pod 所看到的自己的 IP 与其他 Pod 或节点看到的相同。

考虑到这些限制，我们剩下 4 个不同的网络问题要解决。

1. Container-to-Container 网络
1. Pod-to-Pod 网络
1. Pod-to-Service 网络
1. Internet-to-Service 网络

本指南的其余部分将依次讨论这些问题及其解决方案。

# 3. Container-to-Container 网络

通常，我们将虚拟机中的网络通信视为直接与以太网设备进行交互，如图 1 所示。

![understanding-kubernetes-networking-model-1](/assets/img/understanding-kubernetes-networking-model-1.png)
> 图 1，以太网设备

实际上，情况比这更微妙。在 Linux 中，每个正在运行的进程都在[网络命名空间](http://man7.org/linux/man-pages/man8/ip-netns.8.html)中进行通信，该命名空间为逻辑网络堆栈提供了自己的路由、防火墙规则和网络设备。本质上，网络命名空间为命名空间内的所有进程提供了全新的网络堆栈。

作为 Linux 用户，可以使用 `ip` 命令创建网络命名空间。例如，以下命令将创建一个名为 `ns1` 的新网络命名空间。

```bash
$ ip netns add ns1
```

创建命名空间后，将在 `/var/run/netns` 下创建该名称的挂载点，即使没有附加任何进程，名称空间也可以保留。

您可以通过列出 `/var/run/netns` 下的所有挂载点，或使用 `ip` 命令列出可用的命名空间。

```bash
$ ls /var/run/netns
ns1
$ ip netns
ns1
```

默认情况下，Linux 将每个进程分配给跟网络命名空间，以提供对外部环境的访问，如图 2 所示：

![understanding-kubernetes-networking-model-2](/assets/img/understanding-kubernetes-networking-model-2.png)
> 图 2，根网络命名空间

就 Docker 结构而言，一个 Pod 被建模为一组共享网络命名空间的 Docker 容器。Pod 中的容器都具有通过分配给 Pod 的网络命名空间分配的相同 IP 地址和端口空间，并且可以通过 `localhost` 相互找到，因为它们位于同一命名空间中。我们可以为虚拟机上的每个 Pod 创建一个网络命名空间。这是使用 Docker 作为“Pod 容器”实现的，该容器使网络命名空间保持打开状态，而“应用程序容器”（用户指定的内容）通过 Docker 是的 `-net=container:function` 加入该命名空间。图 3 显示每个 Pod 如何共享命名空间中的多个 Docker 容器（ctr*）组成。

![understanding-kubernetes-networking-model-3](/assets/img/understanding-kubernetes-networking-model-3.png)
> 图 3，每个 Pod 的网络命名空间

Pod 中的应用程序还可以访问共享卷，这些共享卷定义为 Pod 的一部分，并且可以挂载到每个应用程序的文件系统中。

# 4. Pod-to-Pod 网络

在 Kubernetes 中，每个 Pod 都有一个真实的 IP 地址，每个 Pod 都使用该 IP 地址与其他 Pod 通信。当前的任务是了解 Kubernetes 如何使用真实 IP 启用 Pod 到 Pod 的通信，无论 Pod 是部署在集群中的同一物理节点还是不同的节点上。我们通过考虑驻留在同一台机器上的 Pod 来开始讨论，以避免通过内部网络进行跨节点通信的复杂性。

从 Pod 的角度看，它存在于自己的以太网命名空间中，该命名空间需要与同一节点上的其他网络命名空间进行通信。值得庆幸的是，可以使用 Linux [虚拟以太网设备](http://man7.org/linux/man-pages/man4/veth.4.html)或由两个虚拟接口组成的 *veth 对* 来连接命名空间，这两个虚拟接口可以分布在多个命名空间上。

要连接 Pod 命名空间，我们可以将 veth 对的一侧分配给根网络命名空间，另一侧分配给 Pod 的网络命名空间。每对 veth 都想跳线一样工作，将两侧连接起来并允许流量再它们之间流动。此设置可以复制到与计算机上一样多的 Pod。图 4 显示了将 VM 上的每个 Pod 连接到根命名空间的 veth 对。

![understanding-kubernetes-networking-model-4](/assets/img/understanding-kubernetes-networking-model-4.png)
> 图 4，每个 Pod 中的 veth 对

至此，我们已经将 Pod 设置为每个都有自己的网络命名空间，以便它们认为自己具有自己的以太网设备和 IP 地址，并且已连接到 Node 的根命名空间。

现在，我们希望 Pod 通过跟命名空间相互通信，为此，我们使用了网桥。

Linux 以太网桥是一种虚拟的第 2 层网络设备，用于将两个或多个网段结合在一起，透明地将两个网络连接在一起。该网桥通过维护源和目标之间的转发表实现，而转发表检查通过它的数据包的目的地，并确定是否将是否将数据包传递到连接网桥的其他网段。桥接代码通过查看网络中每个以太网设备唯一的 MAC 地址来决定桥接数据还是丢弃数据。

网桥使用 [ARP](https://en.wikipedia.org/wiki/Address_Resolution_Protocol) 协议以发现与给定 IP 地址关联的链路层 MAC 地址。当网桥接收到数据帧时，网桥会将帧广播到所有连接到的设备（原始发送方式除外），并且将对帧进行相应的的设备存储在查找表中。具有相同 IP 地址的后续流量将使用查找表来发现正确的 MAC 地址，以将数据包转发到该地址。

![understanding-kubernetes-networking-model-5](/assets/img/understanding-kubernetes-networking-model-5.png)
> 图 5，使用桥接连接命名空间

## 4.1 数据包的生命周期：Pod-to-Pod，同一节点

给定将每个 Pod 隔离到自己的网络堆栈的网络命名空间，将每个命名空间连接到跟命名空间的虚拟以太网设备，以及将命名空间连接在一起的网桥，我们终于可以在同一节点上的 Pod 之间发送流量了。如图 6 所示：

![understanding-kubernetes-networking-model-6](/assets/img/understanding-kubernetes-networking-model-6.gif)
> 图 6，数据包在同一节点上的 Pod 之间传输

在图 6 中，Pod 1 将数据包发送到其自己的以太网设备 `eth0`，该设备可作为 Pod 的默认设备。对于 Pod 1，`eth0` 通过虚拟以太网设备连接到根命名空间 `veth0`（1）。网桥 `cbr0` 配置有 `veth0` 附加到它的网段。数据包到达网桥后，网桥解析正确的网络段，以使用 ARP 协议（3）将数据包发送到 `veth1`。当数据包到达虚拟设备 `veth1` 时，它将直接转发到 Pod 2 的命名空间和该命名空间中的 `eth0` 设备（4）。在整个流量中，每个 Pod 仅与 `localhost` 上的 `eth0` 通信，并且流量被路由到正确的 Pod。使用网络的开发经验是开发人员期望的默认行为。

Kubernetes 的网络模型规定 Pod 必须通过其跨节点的 IP 地址才能访问。即，一个 Pod 的 IP 地址始终对网络中的其他 Pod 可见，并且每个 Pod 都将自己的 IP 地址视为与其他 Pod 看到的 IP 地址相同。现在我们来讨论在不同节点上的 Pod 之间路由流量的问题。

## 4.2 数据包的生命周期：Pod-to-Pod，跨节点

在确定了如何在同一节点上的 Pod 之间路由数据包之后，我们继续在不同节点上的 Pod 之间路由流量。Kubernetes 网络模型要求 Pod IP 可以在整个网络上访问，但并未指定必须如何完成。实际上，这是特定于网络的，但是已经建立了一些模式来简化此过程。

通常集群中的每个节点都分配有一个 CIDR 块，该块指定了该节点上运行 Pod 可用的 IP 地址。一旦发往 CIDR 区块的流量到达节点，则节点有责任将流量转发到正确的 Pod。图 7 说明了两个节点之间的流量流，*假设网络可以将 CIDR 块 中的流量路由到正确的节点*。

![understanding-kubernetes-networking-model-7](/assets/img/understanding-kubernetes-networking-model-7.gif)
> 图 7，数据包在不同节点上的 Pod 之间传输

图 7 从与图 6 相同的请求开始，除了这次，目标 Pod（以绿色突出显示）与源 Pod（以蓝色突出显示）位于不同的 Node 上。数据包首先通过 Pod 1 的以太网设备发送，该设备与根命名空间（1）中的虚拟以太网设备配对。最终，数据包最终到达根命名空间的网络桥（2）。ARP 将在网桥上失败，因为没有与数据包的 MAC 地址匹配的设备连接到该网桥。失败之后，网桥会将数据包发送出默认路由 —— 根命名空间的 `eth0` 设备。此时，路由离开节点并进入网络（3）。现在我们假设网络可以根据分配给节点（4）的 CIDR 块将数据包路由到正确的节点。数据包进入目标节点的根命名空间（VM 2 上的 `eth0`），在此它通过网桥路由到正确的虚拟以太网设备（5）。最后，通过流经 Pod 4 命名空间（6）中的虚拟以太网设备对来完成路由。一般而言，每个节点都知道如何将数据包传递到其中运行的 Pod。数据包到达目标节点后，数据包的流动方式与在同一节点上的 Pod 之间路由流量的方式相同。

我们方便地回避了如何配置网络，以将 Pod IP 的流量路由到负责这些 IP 的正确节点。这是特定于网络的，但是查看特定示例将提供有关所涉及问题的一些见解。例如，对于 AWS，Amazon 为 Kubernetes 维护了一个容器网络插件，该插件允许使用[容器网络接口（CNI）插件](https://github.com/aws/amazon-vpc-cni-k8s)在 Amazon VPC 环境中运行节点到节点网络。

容器网络接口（CNI）提供了用于将容器连接到外部网络的通用 API。作为开发人员，我们想知道 Pod 可以使用 IP 地址与网络通信，并且我们希望此操作的机制透明。由 AWS 开发的 CNI 插件师徒满足这些需求，同时通过 AWS 提供的现有 VPC，IAM 和安全组功能提供安全和可管理的环境。解决方案是使用弹性网络接口。












# 备注

* 原文：[https://sookocheff.com/post/kubernetes/understanding-kubernetes-networking-model/](https://sookocheff.com/post/kubernetes/understanding-kubernetes-networking-model/)

