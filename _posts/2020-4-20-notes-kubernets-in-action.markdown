---
layout: post
title:  "Kubernetes in action 笔记"
date:   2020-4-20 22:29:47 +0800
categories: sre k8s
---

    Kubernetes in action 笔记，温故而知新。

# VM vs Docker

* 运行在容器中的进程实际上像其他进程一样运行在宿主机操作系统中（不像 VM，进程运行在独立的操作系统中）。但是容器中的进程仍然与其他进程隔离。
* 虚拟机的主要优点在于它提供了完全的隔离，因为每个 VM 运行在自己的 Linux 内核，然而所有容器运行在同样的内核，这可能会存在潜在的安全风险。
* 基于 Docker 的容器镜像与 VM 镜像的一个显著区别：容器镜像是分层的，可以由多个镜像共享和重用。

# Docker

* Dockerfile 中 ENTRYPOINT vs CMD。
* ENTRYPOINT 的两种执行方式：shell vs exec。

# 容器

* 容器技术的基础，Linux Namespace 和 Linux Control Groups。
* 一个进程并不是属于一个命名空间，而是属于一组命名空间。
* 实现进程隔离的并不是 Docker，Docker 只是使这些特性更易用。
* 进程在容器中的 PID 与宿主机中的不同。

# Pod

* 包含一个或多个紧密联系的容器。
* 总是运行在同一个 worker 节点上。
* 属于同一个 namespace。
* 为什么需要 Pod？为什么不能直接使用容器？为什么需要运行多个容器？为什么不能将所有进程放在同一个容器中？
* 注解也是引入新特性的一种过渡手段。
* Pod 的终止过程：SIGTERM，等待 30s，SIGKILL。
* 退出码的涵义：128+x，x 为发送到进程的信号值。

# ReplicaSet

* ReplicaSet 的标签选择器更灵活，与 ReplicationController 相比。

# DaemonSet

* DaemonSet 会绕过 Scheduler！！！

# Service

* 使用一个不变的 IP 和 Port 来暴露多个不断变化 IP 的 Pod。
* 会话一致性，Kubernetes 只支持基于 `ClientIP`，不支持基于 `Cookie`。
* 你不能 ping 一个 Service IP，因为它是一个虚拟 IP，只有和 Port 一起使用才有效。
* `ExternalName` 类型经常被忽略，实际上它只是一个 `CNAME`。
* `LoadBalancer` 是 `NodePort` 类型的扩展，如果不支持 `LoadBalancer` 仍然会开启 `NodePort` 类型。

# Ingress

* `LoadBalancer` 与 Ingress 的一个区别在于，每个 `LoadBalancer` 都需要一个外网 IP，而 Ingress 只需要一个。
* Ingress 可以处理应用层的 HTTP 网络堆栈，故能提供基于 cookie 的会话一致性，而 Service 不能。
* Ingress 直接转发流量到 Pod，并不经过 Service。

# Volumes

* volumes 是 Pod 的一部分，并非 Kubernetes 顶级资源。
* `emptyDir` 是用于 Pod 中多个容器共享文件的一种临时磁盘。
* `hostPath` 类型的常见场景是收集 node 节点的日志。

# kubectl

* 双短线（`--`）表示命令的结束，防止后面命令的参数无效。

# 基础组件

## etcd

* etcd 是 Kubernetes 存储集群状态和元数据的唯一位置。
* Kubernetes API server 是与 etcd 直接交互的唯一组件。
* ectd 使用了 RAFT 算法实现分布式一致性。
* etcd 的实例数量为什么是奇数？

## Scheduler

* Scheduler 并不负责指定选定的 node 去运行 Pod，而是通过 API server 更新 Pod 定义并存储在 etcd 中，API server 来通知 Kubelet 运行 Pod。

## Controller Manager

* Controller Manager 包含多个 Controller。
* Controllers 实现系统的期望状态。
* Controller Manager （Replication、ReplicaSet、StatefulSet）通过创建新的 Pod 清单，POST 到 API server，并通知 Scheduler 和 Kubelet 实现调度和运行 Pod。
* Namespace controller 实现了删除 Namespace 资源时，同步删除 Namespace 中的所有资源。
* 所有的 Contoller 通过 API server 实现操作，并不直接与 Kubelet 或其他组件交流。

## Kubelet

* Kubelet 负责运行在 worker 节点的一切活动。
* Kubelet 既可以从 Kubernetes API server 获取 Pod 清单，也可以从指定的本地目录获取 Pod 清单（用于运行控制平面的组件）来运行 Pod。

## kube-proxy

* kube-proxy 负责将到达 service IP 与 port 的流量转发到后端的 Pod。