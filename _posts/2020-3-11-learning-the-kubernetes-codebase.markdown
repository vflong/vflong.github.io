---
layout: post
title:  "【译文】学习 Kubernetes 源码"
date:   2020-3-11 22:06:05 +0800
categories: sre k8s
---

Kubernetes 是一款仅需不到 [200 万行代码](https://www.openhub.net/p/kubernetes)的大型软件。入门可能会令人生畏，但是一旦掌握了窍门，您会惊讶于一切努力都变得有意义，并能够快速识别并修复 bug！

# 开始

如果您刚刚开始 Kubernetes 之旅，我建议您阅读[官方文档](https://kubernetes.io/docs/concepts/overview/components/)并对主要组成部分有较深入的了解。要使用 Kubernetes 集群，它需要列出的大部分组件至少在一个节点上运行。非常方便的是，这些组件的所有代码都位于 [Kubernetes monorepo](https://github.com/kubernetes/kubernetes) 中。

# 第一步

现在您已经知道每个组件的功能，您可以选择一个听起来有趣的组件。如果您在挑选组件时遇到麻烦，我想做的一件事就是挑选离我的舒适范围最远的东西。例如，如果网络方面是您的薄弱点，请选择 `kube-proxy`。如果您遇到了网络问题，但实际上却与 web 服务无关，请选择 `kube-apiserver`。

# 一步一步来

每个码农都需要一个工具，即一个好的代码编辑器。读取代码是对我最大帮助的功能是跳转到定义。我使用 [VSCode](https://code.visualstudio.com/) 与典型的 Go 插件一起使用，满足了我的要求。您可以使用 Vim、Goland 或 Emacs，只要您对该工具感到满意，并且可以在函数之间有效地跳转，就可以开始使用了。

# 一小步

Kubernetes 源码托管在 [https://github.com/kubernetes/kubernetes](https://github.com/kubernetes/kubernetes)。同创，托管在 GitHub 上的 Go 代码的 import 路径是不带 scheme 的 URL（https://）。因此，您可能希望 `go -d github.com/kubernetes/kubernetes` 能够正常个工作，但这会使您的工具混乱。软件包名称实际上是 `k8s.io/kubernetes`。因此，继续使用此命令克隆 repo `go get -d k8s.io/kubernetes`。如果您尚未安装 go，请按照 [golang.org](https://golang.org/) 上的说明进行操作。`kube-proxy` 的[挂载点](https://github.com/kubernetes/kubernetes/blob/release-1.11/cmd/kube-proxy/proxy.go)在这里。`cmd/<component>/<component>.go` 的这种模式在每个组件上都几乎相同。

# 另一小步

使用您喜欢的代码编辑器打开 Kubernetes 目录，看看 `cmd/`。在这里，您可以找到 Kubernetes 内部所有组件的*挂载点*。Kubernetes 中的每个组件都作为命令程序运行，并且存在相对应的在线[手册](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/)。

# 永远重复

现在开始浏览文件！您在 `cmd/<component>/<component>.go` 下打开的第一个文件可能只是一小步。通读每一行。您知道它是用于做什么的吗？如果您在[这行](https://github.com/kubernetes/kubernetes/blob/release-1.11/cmd/kube-proxy/proxy.go#L38)上跳转到函数调用的定义，您将了解到该代码的深度。您还有许多源码要阅读。

现在执行此操作，直到您对自己的理解感到满意为止。这将需要一段时间，而且这是完全正常的。这些东西不容易理解。因此，您可以发表一两篇关于您发现有用，有趣的兔子洞的文章，以及有关 Kubernetes 代码库的令人惊讶的事情！

如果您想涉足 Kubernetes 开发，那么没有比阅读大量源码更好的方法了！

# 备注

* 原文：[https://dev.to/chuck_ha/learning-the-kubernetes-codebase-1324](https://dev.to/chuck_ha/learning-the-kubernetes-codebase-1324)