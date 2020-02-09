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

在进行调试之前，让我们回顾一下这三个组件如何相互连接。

我们从 Deployment 和 Service 开始。

### 连接 Deployment 和 Service

令人惊讶的消息是服务和部署根本没有连接。

相反，Service 端直接连接了 Pod，完全跳过了 Deployment。

所以你应该关心 Pod 和 Serivce 之前是如何相互关联的。

你应该注意以下 3 点：

* Service 选择器（selector）应该匹配至少一个 Pod 的标签（label）
* Service `targetPort` 应该匹配 Pod 内部容器内的的 `containerPort`
* Service `port` 可以是任何数字。因为 Serivce 有不同的 IP 地址，所以多个 Service 可以使用相同的端口。

下图总结了如何连接端口：

1. 通过 Service 暴露下面的 Pod。
![troubleshooting-kubernetes-4](/assets/img/troubleshooting-kubernetes-4.svg)

1. 创建 Pod 时，应为 Pod 中的每个容器定义 `containerPort`。
![troubleshooting-kubernetes-5](/assets/img/troubleshooting-kubernetes-5.svg)

1. 创建服务时，您可以定义 `port` 和 `targetPort`，但是您应该选择连接哪一个容器？
![troubleshooting-kubernetes-6](/assets/img/troubleshooting-kubernetes-6.svg)

1. `tagetPort` 和 `containerPort` 应该总是匹配。
![troubleshooting-kubernetes-7](/assets/img/troubleshooting-kubernetes-7.svg)

1. 如果您的容器暴露了端口 3000，那么 `targetPort` 应该总是匹配这个数字。 
![troubleshooting-kubernetes-8](/assets/img/troubleshooting-kubernetes-8.svg)

如果您查看 YAML 文件，可以发现标签和 `ports`/`targetPort` 应该匹配：
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
    any-name: my-app
```

Deployment 部分中的 `track: canary` 是什么意思？

那也应该匹配吗？

该标签属于 Deployment，Service 的选择器（selector）不适用它来路由流量。

换句话说，您可以安全地删除或为其分配其他值。

那么 `matchLabels` 选择器呢？

**它始终必须匹配 Pod 标签**，因为它是 Deployment 用来追踪 Pod 的。

假设您做出了正确的更改，如何测试它？

您可以使用一下命令检查 Pod 是否具有正确的标签：

```bash
$ kubectl get pods --show-labels
```

或者您有属于多个应用的 Pod：

```bash
$ kubectl get pods --selector any-name=my-app --show-labels
```

其中 `any-time=my-app` 表示标签 `any-name: my-app`。

仍然存在问题？

您可以直接连接 Pod！

您可以在 kubectl 中使用 `port-forward` 命令连接 Service 并测试连接。

```bash
$ kubectl port-forward service/<service_name> 3000:80
```

其中：

* `service/<service_name>` 是 service 的名称 - 在当前 YAML 中是“my-service”。
* 3000 是您想要在您的电脑中开放的端口
* 80 是 Service 中 `port` 字段暴露的端口

如果您能够连接，说明设置是正确的。

如果您不能，那么您很可能配错了一个标签或者端口不匹配。

### 连接 Service 和 Ingress