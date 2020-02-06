---
layout: post
title:  "Kubernetes deployments 排障指南"
date:   2019-12-17 18:40:41 +0800
categories: sre
---

    原文：https://learnk8s.io/troubleshooting-deployments
    一份排障指南流程图。

![troubleshooting-kubernetes](/assets/img/troubleshooting-kubernetes.jpg)

在 Kubernetes 中部署一个应用，需要定义 3 个组件：

* Deployment - 这是创建名为 Pod 的应用程序副本的菜谱
* Service - 内部负载均衡器，将流量路由到 Pods
* Ingress - 有关流量如何从集群外部流向你的服务的描述

下面是快速的回放。

1. 在 Kubernetes 中，应用程序通过两层负载均衡暴露：内部和外部。
![troubleshooting-kubernetes-1](/assets/img/troubleshooting-kubernetes-1.svg)

1. 内部的负载平衡被称为 Service，而外部的被称为 Ingress。
![troubleshooting-kubernetes-2](/assets/img/troubleshooting-kubernetes-2.svg)

1. Pod 并不是被直接部署的，而是通过 Deployment 创建 Pod 并监测。
![troubleshooting-kubernetes-3](/assets/img/troubleshooting-kubernetes-3.svg)
