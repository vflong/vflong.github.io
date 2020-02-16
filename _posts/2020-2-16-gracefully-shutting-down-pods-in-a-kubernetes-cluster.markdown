---
layout: post
title:  "平滑关闭 Kubernetes 集群中的 Pod"
date:   2020-2-16 16:31:18 +0800
categories: sre k8s
---

    原文：https://blog.gruntwork.io/gracefully-shutting-down-pods-in-a-kubernetes-cluster-328aecec90d

![gracefully-shutting-down-pods-in-a-kubernetes-cluster-1](/assets/img/gracefully-shutting-down-pods-in-a-kubernetes-cluster-1.png)
> 平滑停止 Kubernetes 中的容器

这是实现 Kubernetes 集群零停机时间更新[旅程]({% post_url 2020-2-16-zero-downtime-server-updates-for-your-kubernetes-cluster %})的第二部分。在[本系列的第一部分]({% post_url 2020-2-16-zero-downtime-server-updates-for-your-kubernetes-cluster %})中，我们提出了原生的 drain 集群中节点的问题和挑战。在本文中，我们将介绍如何解决这些问题中的一个：平滑关闭 Pod。

# Pod 驱逐生命周期

默认情况下，`kubectl drain` 将以某种方式驱逐 Pod，以遵从 Pod 生命周期。实际上，这意味着它将遵循以下流程：

* `drain` 将向控制平面发出删除目标节点上的 Pod 的请求。随后，这将通知目标节点上的 `kubelet` 开始关闭 Pod。
* 节点上的 `kubelet` 将调用 Pod 中的 `preStop` 勾子。
* 一旦 `preStop` 勾子完成，节点上的 `kubelet` 将向 Pod 容器中正在运行的应用程序发出 `TERM` 信号。
* 节点上的 `kubelet` 将等待最多宽限期（在 Pod 上指定，或从命令行传递；默认为 30 秒）以关闭容器，然后强行终止进程（使用 `SIGKILL`）。请注意，此宽限期包括执行 `preStop` 勾子的时间。

基于此流程，您可以利用应用程序容器中的 `preStop` 勾子和信号处理来正常关闭应用程序，以便在最终终止应用程序之前对其进行“清理”。例如，如果您有一个工作进程从队列中流式传输任务，则可以让您的应用程序捕获 `TERM` 信号，以指示该应用程序应停止接受新工作，并在所有当前工作完成后停止运行。或者，如果您运行的应用程序无法修改以捕获 `TERM`信号（例如第三方应用程序），则可以使用 `preStop` 勾子来实现该服务提供的自定义 API，以便正常关闭应用。

在我们的示例中，Nginx 默认情况下不会平滑地处理 `TERM` 信号，从而导致现有的服务请求失败。因此我们将改为依靠 preStop 勾子正常停止 Nginx。我们将修改资源清单，在容器 spec 中添加 `lifecycle` 指令。`lifecycle` 指令如下所示：

```yaml
lifecycle:
  preStop:
    exec:
      command: [
        # Gracefully shutdown nginx
        "/usr/sbin/nginx", "-s", "quit"
      ]
```

使用此配置，在将 `SIGTERM` 发送到容器中的 Nginx 进程之前，关闭序列将发出命令 `/usr/sbin/nginx -s quit`。请注意，由于该命令将正常停止 Nginx 进程和 Pod，因此 `TERM` 信号实际上并没有生效。

这应该是嵌套在 Nginx 容器 spec 下。当包含此内容时，`Deployment` 的完整配置如下所示：

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.15
        ports:
        - containerPort: 80
        lifecycle:
          preStop:
            exec:
              command: [
                # Gracefully shutdown nginx
                "/usr/sbin/nginx", "-s", "quit"
              ]
```

# 关机后的持续流量

平滑关闭 Pod 可以确保 Nginx 在关闭之前以服务现有流量的方式停止。然而，您可能会发现，尽管有最佳意图，但 Nginx 容器在关闭后仍会继续接收流量，从而导致服务停机。

要了解这可能会带来什么问题，让我们通过实例 deployment 逐步介绍一个示例。对于此示例，我们将假定节点已从客户端接收流量。这将在应用程序中产生一个工作线程来处理请求。我们将在 Pod 容器中用圆圈表示该线程：

![gracefully-shutting-down-pods-in-a-kubernetes-cluster-2](/assets/img/gracefully-shutting-down-pods-in-a-kubernetes-cluster-2.png)

假设此时，集群操作员决定对节点 1 进行维护。为此，操作员执行命令 `kubectl drain node-1`，使节点上的 `kubelet` 进程执行 `preStop` 勾子，从而开始正常关闭 Ngnix 进程：

![gracefully-shutting-down-pods-in-a-kubernetes-cluster-3](/assets/img/gracefully-shutting-down-pods-in-a-kubernetes-cluster-3.png)

由于 Nginx 仍在为原始请求提供服务，因此它不会立即终止。但是，当 Nginx 启动正常关闭时，它将出错并拒绝随之而来的其他流量。

此时，假设有一个新的服务请求进入我们的服务。由于 Pod 仍在服务中注册，因此 Pod 仍可以接收流量。如果这样做，这将返回错误，因为 Nginx 服务正在关闭：

![gracefully-shutting-down-pods-in-a-kubernetes-cluster-4](/assets/img/gracefully-shutting-down-pods-in-a-kubernetes-cluster-4.png)

为了完成序列，最终 Nginx 将完成对原始请求的处理，这将终止 Pod，节点将完成 drain：

![gracefully-shutting-down-pods-in-a-kubernetes-cluster-5](/assets/img/gracefully-shutting-down-pods-in-a-kubernetes-cluster-5.png)
![gracefully-shutting-down-pods-in-a-kubernetes-cluster-6](/assets/img/gracefully-shutting-down-pods-in-a-kubernetes-cluster-6.png)

在此示例中，当应用程序 Pod 在启动关闭序列后接收到流量时，第一个客户端将受到来自服务器的响应。但是，第二个客户端收到一个错误，该错误将被视为停机。

那么为什么会这样呢？对于在关闭序列期间最终连接到服务器的客户端，您如何减少潜在的停机时间？在[本系列的下一部分]({% post_url 2020-2-16-delaying-shutdown-to-wait-for-pod-deletion-propagation %})中，我们将更详细地介绍 Pod 驱逐生命周期，并描述如何在 `preStop` 勾子中引入延迟，以减轻来自 `Service` 的持续流量的影响。