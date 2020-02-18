---
layout: post
title:  "Kubernetes 集群的零停机服务器更新"
date:   2020-2-16 11:02:14 +0800
categories: sre k8s
---

    原文：https://blog.gruntwork.io/zero-downtime-server-updates-for-your-kubernetes-cluster-902009df5b33

![zero-downtime-server-updates-for-your-kubernetes-cluster-1](/assets/img/zero-downtime-server-updates-for-your-kubernetes-cluster-1.png)
> Kubernetes 集群的滚动更新

在 Kubernetes 集群的生命周期中的某个时候，您将需要对基础节点执行维护。这可能包括程序包更新、内核升级或部署新的 VM 镜像。在 Kubernetes 中，被视为[“自愿中断”（Voluntary Disruption）](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/#voluntary-and-involuntary-disruptions)。

这是一个分为 4 部分的博客系列的一部分：

1. 本文
1. [优雅地关闭 Pod]({% post_url 2020-2-16-gracefully-shutting-down-pods-in-a-kubernetes-cluster %})
1. [延迟关闭以等待 Pod 删除传播]({% post_url 2020-2-16-delaying-shutdown-to-wait-for-pod-deletion-propagation %})
1. [使用 PodDisruptionBudge 避免中断]({% post_url 2020-2-16-avoiding-outages-in-your-kubernetes-cluster-using-poddisruptionbudgets %})

在本系列中，我们将介绍 Kubernetes 提供的所有工具，以实现集群中底层工作节点的零宕机时间更新。

# 说明问题

我们将从原生的方法开始，找出该方法的挑战和潜在风险，并逐步构建解决我们在整个系列中确定的每个问题。我们将完成一个配置，该配置利用生命周期勾子、就绪探针以及 Pod 中断预算来实现零停机时间部署。

首先，我们来看一个具体的例子。假设我们有一个两个节点的 Kubernetes 集群，该集群运行一个应用程序，其中两个 Pod 支持 Service 资源：

![zero-downtime-server-updates-for-your-kubernetes-cluster-2](/assets/img/zero-downtime-server-updates-for-your-kubernetes-cluster-2.png)
> 我们的起点是两个 Nginx Pod 和在两个节点 Kubernetes 集群上运行的 Service。

我们先要升级集群中两个底层工作程序节点的内核版本。我们该如何做？原生的方式是使用更新的配置启动新节点，然后在启动新节点后关闭旧节点。尽管这样可行，但是这种方法存在一些问题：

* 当关闭旧节点时，您将会同时将在旧节点上运行的 Pod 下线。如果 Pod 需要清理以正常关闭该怎么办？底层 VM 技术可能不会等待清理过程。
* 如果您同时关闭所有节点怎么办？将 Pod 重新启动到新节点时，您可能会短暂中断。

我们想要的是一种从旧节点上优雅迁移 Pod 的方法，以确保在对节点进行更改时，没有任何工作负载运行。或者，如实例中所示，如果要完全替换集群（例如替换 VM 镜像），我们希望将工作负载从旧节点迁移到新节点。在这两种情况下，我们都希望避免将新 Pod  调度到旧节点，并且将所有正在运行的 Pod 从其上逐出。我们可以使用 [kubectl drain](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/) 命令实现它。

# 重新调度节点上的 Pod

Drain 操作实现了将所有 Pod 重新调度到其他节点的目的。在 drain 操作期间，该节点被标记为不可调度（`NoSchedule` 污点）。这样可以防止新的 Pod 被调度到该节点。之后，drain 操作开始从节点驱逐 Pod，通过将 `TERM` 信号发送到 Pod 的底层容器来关闭当前在该节点上运行的容器。

尽管 `kubectl drain` 可以优雅处理 Pod 驱逐，但仍存在两个因素可能会在 drain 操作过程中导致服务中断：

* 您的应用程序服务需要能够优雅处理 TERM 信号。驱逐 Pod 时，Kubernetes 将 `TERM` 信号发送容器，然后在发出信号后将容器强制关闭之前等待可配置时间，以使用容器关闭。但是，如果您的容器无法正常处理信号，则在工作期间（例如提交数据库事务），您仍然可以不干净地关闭 Pod。
* 您将失去为应用程序提供服务的所有 Pod。在新节点上启动新容器时，您的服务可能会停机，或者，如果未使用控制器部署 Pod，则它们可能永远无法重启。

# 避免宕机

为了最大程度地减少因 drain 节点等自愿性中断而导致的停机时间，Kubernetes 提供以下中断处理功能：

* [优雅终止](https://kubernetes.io/docs/concepts/workloads/pods/pod/#termination-of-pods)
* [生命周期勾子](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/)
* [PodDisruptionBudgets](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/#how-disruption-budgets-work)

在本系列的其余部分中，我们将使用 Kubernetes 的这些功能来减轻驱逐时间对服务的干扰。为了使后续操作更容易，我们将在上面的示例中使用以下资源配置：

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
---
kind: Service
apiVersion: v1
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
  - protocol: TCP
    targetPort: 80
    port: 80
```

此配置是管理多个 Nginx Pod（在我们的示例中是两个）的 [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) 资源的最小示例。该资源将用于维护集群中的两个 Nginx Pod。此外，配置将提供可用于访问集群中 Nginx Pod 的 [Service](https://kubernetes.io/docs/concepts/services-networking/service/) 资源。

我们将在本系列的整个过程中逐步增加它，以构建最终配置，以实现 Kubernetes 提供的所有功能，以最大程度地减少维护操作期间的停机时间。这是我们的路线图：

1. [优雅地关闭 Pod]({% post_url 2020-2-16-gracefully-shutting-down-pods-in-a-kubernetes-cluster %})
1. [延迟关闭以等待 Pod 删除传播]({% post_url 2020-2-16-delaying-shutdown-to-wait-for-pod-deletion-propagation %})
1. [使用 PodDisruptionBudge 避免中断]({% post_url 2020-2-16-avoiding-outages-in-your-kubernetes-cluster-using-poddisruptionbudgets %})

继续阅读[下一篇文章]({% post_url 2020-2-16-gracefully-shutting-down-pods-in-a-kubernetes-cluster %})，了解如何利用生命周期勾子来正常关闭 Pod。