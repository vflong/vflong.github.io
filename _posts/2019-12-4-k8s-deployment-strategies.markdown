---
layout: post
title:  "Kubernetes 部署策略"
date:   2019-12-05 22:51:33 +0800
categories: sre
---

    https://github.com/ContainerSolutions/k8s-deployment-strategies
    这个项目的一些概念摘要，不包含操作过程。

### Kubernetes 部署策略

> 在Kubernetes中，几乎没有其他方法可以发布应用程序，因此您必须仔细选择正确的策略以使基础架构具有弹性。

* 重建：终止旧版本并发布新版本
* 滚动：以滚动更新的方式发布新版本
* 蓝/绿：与旧版本一起发布新版本，然后切换流量
* 金丝雀：向一部分用户发布新版本，然后继续全面推出
* A/B 测试：以精确的方式（HTTP 头部，Cookie，权重等）向一部分用户发布新版本。Kubernetes 并不是开箱即用的，这意味着需要额外的工作设置一个更智能的负载均衡系统（Istio、Linkerd、Traeffik、自定义 nginx/haproxy 等）。
* 影子：与旧版本一起发布新版本。传入流量已镜像到新版本，不会影响响应。
