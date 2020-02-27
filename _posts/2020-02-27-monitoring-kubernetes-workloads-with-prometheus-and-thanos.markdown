---
layout: post
title:  "使用 Prometheus 和 Thanos 监控 Kubernetes 工作负载"
date:   2020-02-27 21:24:20 +0800
categories: sre k8s
---

# 介绍

恭喜！您已经成功说服您的项目经理、研发副总裁或者 CTO 使用 Kubernetes 上的容器将公司的工作负载歉意到微服务。

您非常高兴，一切都按计划进行，您创建了第一个 Kubernetes 集群（3 个主要的云服务商，Azure、AWS 和 GCP 都有一种简单的方法来配置托管或非托管 Kubernetes 平台），开发您的第一个容器化应用程序，并将其部署到集群中。那很容易，不是吗？

一段时间后，您意识到事情变的更加复杂，您有多个应用程序要部署到集群，因此需要一个 *Ingress Controller*，然后，在部署到生产环境之前，您需要了解工作负载的执行情况，所以开始需要监控解决方案，幸运的时，您找到了 [Prometheus](https://prometheus.io/)，部署它，添加 [Grafana](http://grafana.com/)，就完成了！

后来，您开始疑惑 —— 为什么 *Prometheus* 仅使用一个副本运行？如果容器重新启动会怎样？或者只是版本更新？*Prometheus* 可以存储我的指标多长时间？当集群出现故障时会发生什么？我是否需要用于 HA 和 DR 的另一个集群？我将如何对 Prometheus 进行集中查看？

好吧，继续阅读，聪明的人已经知道了。

# 典型的 Kubernetes 集群

下图是 Kubernetes 上的典型监控部署架构：

![monitoring-kubernetes-workloads-with-prometheus-and-thanos-1](/assets/img/monitoring-kubernetes-workloads-with-prometheus-and-thanos-1.jpeg)
> 典型的 Kubernetes 集群

部署架构包括 3 层：

1. 底层虚拟机 —— 主节点和工作节点
1. Kubernetes 基础架构应用
1. 用户应用

不同的组件在内部相互通信，通常使用 HTTP(s)（REST 或 gRPC），其中一些将 API 暴露给集群外部（Ingress），这些 API 主要用于：

1. 通过 Kubernetes API 服务器进行集群管理
1. 通过 Ingress Controller 暴露的用于应用程序交互

在某些情况下，应用程序可能会将流量发送到集群（Egress）之外以使用其他服务，例如 Azure SQL、Azure Blob 或任何第三方服务。

# 要监视什么？

# Thanos

# 部署 Thanos

# 其他选择

# 备注

* 原文：[https://itnext.io/monitoring-kubernetes-workloads-with-prometheus-and-thanos-4ddb394b32c](https://itnext.io/monitoring-kubernetes-workloads-with-prometheus-and-thanos-4ddb394b32c)


