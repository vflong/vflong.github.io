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
* Ingress - 有关流量如何从集群外部流向您的服务的描述

下面是快速的回顾。

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

# 连接 Deployment 和 Service

令人惊讶的消息是 Service 和 Deployment 根本没有连接。

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

该标签属于 Deployment，Service 的选择器（selector）不使用它来路由流量。

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

# 连接 Service 和 Ingress

下一步暴露您的应用的步骤是配置 Ingress。

Ingress 必须知道如何获取到 Service 然后获取到 Pod 并路由流量到它们。

Ingress 获取根据名称和暴露的端口获取到正确的 Service。

Ingress 和 Service 中的两个属性应该匹配：

1. Ingress 的 `servicePort` 应该匹配 Service 中的 `port`
1. Ingress 中的 `serviceName`  应该匹配 Service 中的 `name`

下图总结了如何连接端口：

1. 您已经知道 Service 暴露了一个 `port`。
![troubleshooting-kubernetes-9](/assets/img/troubleshooting-kubernetes-9.svg)


1. Ingress 中有一个字段叫做 `servicePort`。
![troubleshooting-kubernetes-10](/assets/img/troubleshooting-kubernetes-10.svg)

1. Service 的 `port` 和 Ingress 的 `servicePort` 应该总是匹配。
![troubleshooting-kubernetes-11](/assets/img/troubleshooting-kubernetes-11.svg)

1. 如果您决定分配端口 80 给 service，您应该也修改 `servicePort` 为 80。
![troubleshooting-kubernetes-12](/assets/img/troubleshooting-kubernetes-12.svg)

在实践中，您应该查看以下几行：

`hello-world.yaml`

```yaml
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
        serviceName: my-service
        servicePort: 80
      path: /
```

您该如何测试 Ingress 的功能？

您可以对 `kubectl port-forward` 使用与之前相同的策略，但是应该连接到 Ingress 控制器，而不是连接到 Service。

首先，使用以下命令获取 Ingress 控制器的 Pod 名称：

```bash
$ kubectl get pods --all-namespaces
NAMESPACE   NAME                              READY STATUS
kube-system coredns-5644d7b6d9-jn7cq          1/1   Running
kube-system etcd-minikube                     1/1   Running
kube-system kube-apiserver-minikube           1/1   Running
kube-system kube-controller-manager-minikube  1/1   Running
kube-system kube-proxy-zvf2h                  1/1   Running
kube-system kube-scheduler-minikube           1/1   Running
kube-system nginx-ingress-controller-6fc5bcc  1/1   Running
```

找出 Ingress Pod（可能在其他 Namespace 中）并描述它以获取端口：

```bash
$ kubectl describe pod nginx-ingress-controller-6fc5bcc \
 --namespace kube-system \
 | grep Ports
Ports:         80/TCP, 443/TCP, 18080/TCP
```

最后，连接 Pod：

```bash
$ kubectl port-forward nginx-ingress-controller-6fc5bcc 3000:80 --namespace kube-system
```

此时，每次您访问计算机上的端口 3000 时，请求都会转发到 Ingress 控制器 Pod 上的端口 80。

如果您访问 [http://localhost:3000](http://localhost:3000)，则应该可以找到提供网页的应用程序。

# 关于 port 的总结回顾

快速回顾一下哪些端口和标签应该匹配：

1. Service selector 应该匹配 Pod 的 label
1. Service 的 `targetPort` 应该匹配 Pod 中容器的 `containerPort`
1. Service 的 port 可以是任意值。多个 Service 可以使用相同的 port，因为它们被分配了不通的 IP 地址
1. Ingress 的 `servicePort` 应该匹配 Service 的 `port`
1. Service 的 `name` 应该匹配 Ingress 的 `serviceName`

知道如何构造 YAML 定义只是故事的一部分。

出了问题怎么办？

Pod 可能无法启动，或者正在崩溃。

# 排查 Kubernetes deployment 故障的 3 个步骤

在深入 Debug 崩溃的 Deployment 之前，必须把 Kubernetes 的工作方式铭记于心。

由于每个 deployment 包含 3 个组件，因此您必须从底层开始依次 debug 这些组件。

1. 您应该确保 Pod 处于 running 状态，然后
1. 专注于让服务将流量路由到 Pod，然后
1. 检查 Ingress 配置是否正确

详细操作如下：

1. 您应该从底层开始对 Deployment 进行故障排查。首先检查 Pod 就绪并正在运行。
![troubleshooting-kubernetes-13](/assets/img/troubleshooting-kubernetes-13.svg)

1. 如果 Pod 已就绪，则应调查服务是否可以将流量分配到 Pod。
![troubleshooting-kubernetes-14](/assets/img/troubleshooting-kubernetes-14.svg)

1. 最后您应该检查 Service 与 Ingress 之间的连接
![troubleshooting-kubernetes-15](/assets/img/troubleshooting-kubernetes-15.svg)


## 1. 排查 Pod 故障

大部分情况下，问题出现在 Pod 自身。

您应该确认您的 Pod 已经处于 *Running* 和 *Ready* 状态。

您如何开始检查？

```bash
$ kubectl get pods
NAME                    READY STATUS            RESTARTS  AGE
app1                    0/1   ImagePullBackOff  0         47h
app2                    0/1   Error             0         47h
app3-76f9fcd46b-xbv4k   1/1   Running           1         47h

```

在上面的会话中，最后一个 Pod 已经处于 *Running* 和 *Ready* 状态 —— 然而，前两个 Pod 并未处于 *Running* 或 *Ready* 状态。

*您如何调查出了什么问题？*

有 4 个常用的命令可以用来排查 Pod 故障：

1. `kubectl logs <pod name>` 有助于获取 Pod  容器的日志
1. `kubectl describe pod <pod name>` 用于获取 Pod 相关的事件列表
1. `kubectl get po <pod name>` 用于提取存储在 Kubernetes 中的 Pod 的 YAML 定义
1. `kubectl exec -it <pod name> bash` 用于在 Pod 内的容器中运行一个交互式命令

您应该选择哪一个？

没有一个命令是万能的。

相反，您应该结合使用它们。

### 常见 Pod 错误

Pod 可能会出现启动或运行时错误。

启动错误如下：

* ImagePullBackOff
* ImageInspectError
* ErrImagePull
* ErrImageNeverPull
* RegistryUnavailavle
* InvalidImageName

运行时错误如下：

* CrashLoopBackOff
* RunContainerError
* KillContainerError
* VerifyNonRootError
* RunInitContainerError
* CreatePodSandboxError
* ConfigPodSandboxError
* KillPodSandboxError
* SetupNetworkError
* TeardownNetworkError

有些错误比其他错误更常见。

以下是最常见的错误以及如何修复它们的列表。

#### ImagePullBackOff

这个错误出现的原因是 Kubernetes 不能拉取到 Pod 中的一个容器的镜像。

有以下 3 中常见的原因：

1. 镜像名称无效 —— 例如，您错误输入了名称，或者镜像不存在
1. 您为镜像指定了一个不存在的 tag
1. 您试图拉取的镜像属于一个私有注册中心，并且 Kubernetes 没有权限访问它

前两种情况可以通过修正镜像名称和 tag 来解决。

对于最后一种情况，您应该将凭证添加到 Secret 中的私有注册中心，并在 Pod 中引用它。

[官方文档中有一个如何实现此目标的示例。](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/)

#### CrashLoopBackOff

如果容器无法启动，则 Kubernetes 将 CrashLoopBackOff 消息显示为状态。

通常，在以下情况下容器无法启动：

1. 应用程序中存在错误，导致无法启动
1. 您[未正确配置容器](https://stackoverflow.com/questions/41604499/my-kubernetes-pods-keep-crashing-with-crashloopbackoff-but-i-cant-find-any-lo)
1. Liveness 探针失败次数过多

您应该尝试从该容器中获取日志以调查其失败的原因。

如果由于容器重启太快而看不到日志，则可以使用以下命令：

```bash
$ kubectl logs <pod-name> --previous
```

以上命令会打印前一个容器错误信息。

#### RunContainerError

当容器无法启动时出现错误。

设置在容器内的应用程序启动之前。

该问题通常是由于配置错误，例如：

* 挂载不存在的卷，例如 ConfigMap 或 Secrets
* 将只读卷挂载为可读写

您应该使用 `kubectl describe pod <pod-name> ` 来收集并分析错误信息。

#### Pod 处于 *Pending* 状态

当您创建一个 Pod 时，该 Pod 保持处于 *Pending* 状态。

为什么？

假设您的调度程序组件运行良好，原因如下：

1. 集群没有足够的资源（例如 CPU 和内存）来运行 Pod
1. 当前的 Namespace 具有 ResourceQuota 对象，创建 Pod 将使 Namespace 超过配额
1. 该 Pod 绑定到了一个处于 *Pending* 状态的 PersistenVolumeClain

最好的选择是检查 `kubectl describe` 命令中的 *Events* 部分：

```bash
$ kubectl describe pod <pod name>
```

对于由于 ResourceQuotas 而造成的错误，可以使用以下方法检查集群日志：

```bash
$ kubectl get events --sort-by=.metadata.creationTimestamp
```

#### Pod 未处于 *Ready* 状态

如果一个 Pod 处于 *Running* 状态但并未 *Ready*，这说明 Readiness 探针失败了。

当 Readiness 探针失败时，Pod 未连接到 Service，并且没有流量转发到该实例。

Readiness 探针失败是特定于应用程序的错误，因此您应该检查 `kubectl describe` 中的 *Events* 部分以找出错误。

## 2. 排查 Service 故障

如果 Pod 处于 Running 状态并 *Ready*，但仍无法收到应用程序的相应，则应检查 Service 的配置是否正确。

Service 旨在根据 Pod 的标签将流量路由至 Pod。

因此，您应该检查的第一件事是 Service 匹配了多少 Pod。

您可以通过检查 Service 中的 Endpoint 来做到这一点：

```bash
$ kubectl describe service <service-name> | grep Endpoints
```

endpoint 是一对 <ip address: port>，并且在 Service（至少）有一个 Pod，当 Service 以 Pod 为目标时。

如果“Endpoints”部分为空，则有两种解释：

1. 您并没有运行带有正确标签的 Pod（提示：您应检查自己是否在正确的 namespace）
2. 您在 Service 的 `selector` 的标签中有错别字

如果您看到了 endpoints 列表，但仍然无法访问您的应用程序，则 service 中的 `targetPort` 可能是罪魁祸首。

您如何测试服务？

无论服务类型如何，都可以使用 `kubectl port-forward` 连接到它：

```bash
$ kubectl port-forward service/<service-name> 3000:80
```

其中：

* `<service-name>` 是 Service 的名称
* `3000` 是您希望在您的电脑上开放的端口
* `80` 是 Service 暴露的端口

## 3. 排查 Ingress

### 调试 Ingress Nginx


# 总结
