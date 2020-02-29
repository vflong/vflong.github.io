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



# 备注

* 原文：[https://sookocheff.com/post/kubernetes/understanding-kubernetes-networking-model/](https://sookocheff.com/post/kubernetes/understanding-kubernetes-networking-model/)

