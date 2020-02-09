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

假设您希望部署一个简单的 Hello World 应用程序，则该应用程序的 YAML 应该类似于以下内容：
`hello-world.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
  labels:
    track: canary
spec:
  selector:
    matchLabels:
      any-name: my-app
  template:
    metadata:
      labels:
        any-name: my-app
    spec:
      containers:
      - name: cont1
        image: learnk8s/app:1.0.0
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  ports:
  - port: 80
    targetPort: 8080
  selector:
    name: app
---
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
  - http:
    paths:
    - backend:
        serviceName: app
        servicePort: 80
      path: /
```

定义很长，很容易忽略组件之间的相互关系。

例如：

* 何时使用 80 端口？何时使用 8080 端口？
* 是否应该为每个服务创建一个新端口，避免冲突？
* 标签名称重要吗？到处都应该一样吗？

在进行调试之前，让我们回顾一下这三个组件如何相互链接。

我们从 Deployment 和 Service 开始。