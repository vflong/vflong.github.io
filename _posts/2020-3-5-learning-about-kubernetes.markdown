---
layout: post
title:  "【译文】关于 Kubernetes 的几点理解"
date:   2020-3-5 12:41:57 +0800
categories: sre k8s
---

我最近在学习有关 [Kubernetes](https://kubernetes.io/) 的知识，我在大约 6 个月前才开始认真考虑它 —— 我的伙伴 Kamal 对 Kubernetes 已经感到兴奋好几年了（他：“julia！你可以运行程序，而不必担心他们在什么计算机上运行！太酷了！”，我：“我不明白，那怎么可能”），但我现在对它的了解要好的多。

这不是全面的解释，而是我在学习过程中学到的一些东西，可以帮助我了解正在发生的事情。

这些天，我实际上是在建立一个集群，而不只是在互联网上阅读它，就像是“那是什么？？”所以我学得更快了:)

我不会尝试解释什么是 Kubernetes。我喜欢 Kelsey Hightower 在 Strange Loop 上的介绍性演讲，名为“[使用 CoreOS 和 Kubernetes 大规模管理容器](https://www.youtube.com/watch?v=pozC9rBvAIs)”，而且，如果您想了解更多，Kelsey 多年来已经为 TONS 进行了许多很棒的 Kubernetes 演讲。

基本上，Kubernetes 是在计算机上运行程序（容器）的分布式系统。您告诉它要运行什么，它将会调度到您的计算机上。

# 几个草图

我今天画了几张“来自 Kubernetes 的场景草图”，试图简要地解释诸如“添加新节点时会发生什么？”之类的事情。

![learning-about-kubernetes-1](/assets/img/learning-about-kubernetes-1.svg)
![learning-about-kubernetes-2](/assets/img/learning-about-kubernetes-2.svg)

# 从头开始的 Kubernetes

帮助我了解 Kubernetes 发生了什么的第一件事是 Kamal 的“从头开始学 Kubernetes”系列。

他基本上会引导您逐步了解所有 Kubernetes 组件如何相互配合 —— “看，您可以自己运行 kubelet！而且，如果您有 kubelet，则可以添加 API server，然后自己运行这两件事！好的，太好了，现在让我们添加调度程序！”像这样呈现时，我发现它更容易理解。这是3个系列文章：

* 第1部分：[kubelet](http://kamalmarhubi.com/blog/2015/08/27/what-even-is-a-kubelet/) 
* 第2部分：[API 服务](http://kamalmarhubi.com/blog/2015/09/06/kubernetes-from-the-ground-up-the-api-server/)
* 第3部分：[调度程序](http://kamalmarhubi.com/blog/2015/11/17/kubernetes-from-the-ground-up-the-scheduler/)

基本上这些帖子告诉我：

* “kubelet”负责在节点上运行容器
* 如果您告诉 API 服务在节点上运行容器，它将告诉 kubelet 完成它（间接）
* 调度程序将“运行容器”转换为“在节点 X 上运行容器”

但是如果您想了解这些组件之间的相互作用，则应该详细阅读它们。Kubernetes 的东西变化很快，但是我认为诸如“API server 如何与  kubelet 交互”之类的基本架构并没有太大变化，这是一个很好的起点。

# etcd：Kubernetes 的大脑

真正帮助我了解 Kubernetes 的工作的下一件事是更好地了解 etcd 的作用。

Kubernetes 中的每个组件（API 服务、调度程序、kubelet、控制器管理器等等）都是无状态的。所有状态都存储在名为 etcd 的键值存储中，并且组件之间的通信通常通过 etcd 进行。

例如！假设您要在机器 X 上运行容器。您不要求该机器 X 上的 kubelet 运行容器。那不是 Kubernetes 的方式！而是，发生这种情况：

1. 您在 etcd 中写道：“此 pod 应该在机器 X 上运行”。 （从技术上讲，您永远不会直接写到 etcd，而是通过 API 服务完成的，但稍后我们会学到此过程）
1. 机器 X上的 kublet 看着 etcd 并想：“天哪！它说 Pod 应该正在运行，而我不在运行！我现在就开始！！”

同样，如果您想将容器放在**某处**，但不在乎：

1. 您写到 etcd “此 pod 应该在某个地方运行”
1. 调度程序看了看，然后想“哎呀！有一个计划外的 Pod！这必须解决！”。它为 Pod 分配了要运行的机器（机器Y）
1. 像以前一样，机器 Y 上的 kubelet 看到了这一点并认为“天啊！预定在我的机器上运行！最好现在就做！！”

当我了解到 Kubernetes 中的所有工作基本上都通过观察 etcd 要做的事情，将其做完然后将新状态写回到 etcd 中而起作用时，Kubernetes 对我来说意义更大。

# API 服务器负责将内容放入 etcd

了解 etcd 还帮助我更好地了解了 API 服务的角色！ API 服务具有一组非常简单的职责：

1. 您告诉它把东西放到 etcd 中
1. 如果您说的话没有道理（与正确的架构不匹配），则拒绝
1. 否则它会完成此指令

那也不是很难理解！

它还管理身份验证（“谁被允许把什么东西放进 etcd？”），这很重要，但是我在这里不做讨论。关于 [Kubernetes 身份验证的页面](https://kubernetes.io/docs/admin/authentication/)非常有用，尽管有点像“您可能会使用的 9 种不同的身份验证策略”。据我所知，常用的方式是 X509 客户端证书。

# 控制器管理器做了很多事情

我们讨论了调度程序如何使用“这是一个应该在某个地方运行的 pod”并将其转换为“该 pod 应该在机器 X上运行”。

还有很多这样的翻译：

* Kubernetes daemonsets 说“在每台机器上运行”。有一个“daemonset 控制器”，当它在 etcd 中看到一个 daemonset 时，它将在具有该pod 配置的每台机器上创建一个 pod。
* 您可以创建一个 relicas “运行其中的 5 个”。当“replicas 控制器”在 etcd 中看到 replicas 时，将创建 5 个Pod，然后由调度程序进行调度。

控制器管理器基本上将一堆具有不同作业的不同程序捆绑在一起。

# 哪里工作异常？找出哪个控制器负责并查看其日志

今天我的 Pod 之一没有被调度。我想了一会儿，以为“嗯，调度程序（scheduler）负责调度 Pod”，然后去查看调度程序（scheduler）的日志。原来，我错误地重新配置了调度程序（scheduler），因此它不再启动了！

我使用 k8s 的次数越多，在遇到问题时就更容易确定哪个组件可能引起问题！

# Kubernetes 组件可以在 Kubernetes 内部运行

令我震惊的一件事是 —— 相对核心的 Kubernetes 组件（例如 DNS 系统和 overlay 网络系统）可以在 Kubernetes 内部运行！ “嘿，Kubernetes，请启动 DNS！”

这基本上是因为要在 Kubernetes 中运行程序，您只需要运行 5 个工具：

* 调度器（scheduler）
* API server
* etcd
* 每个节点上的 kubelet（实际管理容器）
* 控制器管理器（由于要设置 daemonset，因此需要控制器管理器）

一旦拥有了这 5 件工具，就可以安排容器在节点上运行！因此，如果您想运行另一个 Kubernetes 组件（例如您的 overlay 网络，DNS 服务器或其他任何东西），您只需要求 API server（通过 `kubectl apply -f your-configuration-file.yaml`）为您调度它，它将运行它！

还有一个 [bootkube](https://github.com/Kubernetes-incubator/bootkube)，您可以在其中运行 Kubernetes（甚至是 API server）中的*所有* Kubernetes 组件，但是今天还不能 100％ 投入生产。在 Kubernetes 内部运行 DNS 服务器之类的东西似乎很正常。

# Kubernetes 网络：并非不可能理解

[该页面](https://kubernetes.io/docs/concepts/cluster-administration/networking/)在解释 Kubernetes 网络模型“每个容器都获得 IP”方面做得很好，但是了解如何真正实现这一目标并非易事！

当我开始学习 Kubernetes 网络时，我对如何为每个容器分配IP地址感到非常困惑。回想一下，我认为这是因为我还不了解其中涉及的一些基本网络概念（要了解什么是“overlay 网络”及其工作原理，您实际上需要了解有关计算机网络的许多知识！
）。

现在，我觉得大多数时候都可以建立一个容器网络系统，并且就像我知道足够的概念来调试遇到的问题一样！我写了[一个容器网络概述](https://jvns.ca/blog/2016/12/22/container-networking/)，试图总结一些我学到的知识。

Sophie 如何调试 kube-dns 问题（[与 kube-dns 相关的故障](http://blog.sophaskins.net/blog/misadventures-with-kube-dns/)）的摘要是一个很好的例子，说明了如何理解基本的网络概念可以帮助您调试问题。

# 了解网络确实有帮助

这是我现在知道的一些事情，比如说两年前我不太了解。了解网络概念确实可以帮助我调试 Kubernetes 集群中的问题 —— 如果我不得不在不了解很多网络基础知识的情况下调试 Kubernetes 集群中的网络问题，我想我只是在搜索错误消息，然后尝试随机修复它们，这将是很痛苦的。

* overlay 网络（我去年写了[一个容器网络概述](https://jvns.ca/blog/2016/12/22/container-networking/)）
* 网络命名空间（通常了解命名空间对于使用容器确实很有帮助）
* DNS（因为 Kubernetes 具有 DNS 服务器）
* 路由表，如何运行 `ip route list` 和 `ip link list`
* 网络接口
* 封装（vxlan/UDP）
* 有关如何使用 iptables 和读取 iptables 配置的基础知识
* TLS、服务器证书、客户端证书、证书颁发机构

# Kubernetes 源代码似乎很容易阅读

Kubernetes源代码全是 Go，这很棒。该项目进展很快，因此不一定总是文档齐全（并且您有时会阅读 github issues 以了解问题的当前状态），但总的来说，我发现 Go 代码易于阅读，并且让我欣慰的是，我正在使用以我能理解的语言编写的东西。

# Kubernetes slack 小组很棒

您可以通过访问 [http://slack.kubernetes.io](http://slack.kubernetes.io) 加入 slack 的组织。我通常会尝试自己解决问题，而不是去那里，但是当我问问题时，那里的人非常有帮助。

# 备注

* 原文：[https://jvns.ca/blog/2017/06/04/learning-about-kubernetes/](https://jvns.ca/blog/2017/06/04/learning-about-kubernetes/)