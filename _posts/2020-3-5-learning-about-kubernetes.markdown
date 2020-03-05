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

我今天画了几张“来自 kubernetes 的场景”草图，试图简要地解释诸如“添加新节点时会发生什么？”之类的事情。

![learning-about-kubernetes-1](/assets/img/learning-about-kubernetes-1.svg)
![learning-about-kubernetes-2](/assets/img/learning-about-kubernetes-2.svg)

# 从头开始的 Kubernetes

帮助我了解 Kubernetes 发生了什么的第一件事是 Kamal 的“从头开始的 Kubernetes”系列。

他基本上会引导您逐步了解所有Kubernetes组件如何相互配合 —— “看，您可以自己运行kubelet！而且，如果您有kubelet，则可以添加API服务器，然后自己运行这两件事！好的，太好了，现在让我们添加调度程序！”像这样呈现时，我发现它更容易理解。这是3个系列文章：

* 第1部分：[kubelet](http://kamalmarhubi.com/blog/2015/08/27/what-even-is-a-kubelet/) 
* 第2部分：[API 服务](http://kamalmarhubi.com/blog/2015/09/06/kubernetes-from-the-ground-up-the-api-server/)
* 第3部分：[调度程序](http://kamalmarhubi.com/blog/2015/11/17/kubernetes-from-the-ground-up-the-scheduler/)

基本上这些帖子告诉我：

* “kubelet”负责在节点上运行容器
* 如果您告诉 API 服务在节点上运行容器，它将告诉 kubelet 完成它（间接）
* 调度程序将“运行容器”转换为“在节点 X 上运行容器”

但是如果您想了解这些组件之间的相互作用，则应该详细阅读它们。Kubernetes的东西变化很快，但是我认为诸如“API 服务器如何与  kubelet 交互”之类的基本架构并没有太大变化，这是一个很好的起点。

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



# 备注