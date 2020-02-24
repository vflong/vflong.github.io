---
layout: post
title:  "Kubernetes Pod 安全策略最佳实践"
date:   2020-02-24 21:46:54 +0800
categories: sre k8s
---

![kubernetes-pod-security-policy-1](/assets/img/kubernetes-pod-security-policy-1)

在 [Kubernetes](https://kubernetes.io/) 中，Pod 安全策略是用于减轻安全风险并[在 Kubernetes 环境中强制执行安全配置的强大工具](https://resources.whitesourcesoftware.com/blog-whitesource/kubernetes-security-best-practices)。但是，使用 Kubernetes Pod 安全策略最大程度地发挥作用需要花费一些精力。您的默认Pod安全策略（实际上默认情况下已设置）不太可能很好地满足您的需求。

考虑到这一挑战，本文说明了如何充分利用 [Kubernetes Pod 安全策略](https://kubernetes.io/docs/concepts/policy/pod-security-policy/)。首先，我们将简要说明策略的运作方式以及其目标是什么。然后，我们将逐步介绍最佳实践，以确保 Kubernetes Pod 安全策略针对您的环境进行了优化。

# Kubernetes Pod 安全策略是什么？

简而言之，Pod 安全策略是定义 Kubernetes Pod 必须满足哪些与安全相关的条件才能被接受到集群的配置。它们调节 Pod 如何与网络和存储等资源进行交互。它们还可以用于配置基于角色的访问控制。

看待 Kubernetes Pod 安全策略的另一种方式是将其描述为在 [Kubernetes 集群](https://medium.com/google-cloud/kubernetes-101-pods-nodes-containers-and-clusters-c1509e409e16)上运行自动一致性测试的一种方法。在安全领域，一致性测试是一种策略，用于确保环境或应用程序在允许运行之前符合预定义的安全标准。Kubernetes Pod 安全策略通过阻止 Pod 运行（除非它们满足策略中定义的安全要求）来执行相同的操作。（要清楚，您不必触发一致性测试来执行Kubernetes pod安全策略。只要启用并配置了这些策略，Kubernetes就会自动执行它们。）

## Kubernetes Pod 安全策略是集群级别资源

需要澄清的一点是：不要让 *Pod 安全策略* 一词使您感到困惑。尽管这个名称听起来像Pod安全策略定义了特定Pod的安全设置，但实际上相反。

Kubernetes Pod 安全策略是集群级别的资源，这意味着它们定义了适用于整个群集而不是单个 Pod 的安全策略。

## 定义和激活 Pod 安全策略

Pod 安全策略一般在 YAML 文件中被指定。如果您正在阅读本文并了解 Kubernetes 是什么，那么您可能也知道 YAML 文件是什么，因此我们在这里不再详细介绍。有关可以在 Pod 安全策略和一些示例策略文件中定义的参数的详细信息，请参阅 [Kubernetes 文档](https://kubernetes.io/docs/concepts/policy/pod-security-policy/)。

编写 Pod 安全策略后，您必须使用 kubectl 激活它。这是一个例子：

```bash
kubectl create -f your-new-policy.yaml
```

当然，将 *your-new-policy.yaml* 替换为您创建的策略文件的名称。

# 充分利用 Kubernetes Pod 安全策略

如何充分利用 Kubernetes Pod安全策略？当然，您的结果会因您的特定配置和需求而异，但是请考虑以下使用 Pod 安全策略的最佳实践。

## 启用它们！

## 禁用特权容器

## 只读文件系统

## 防止特权升级

## 防止容器以 root 身份运行

## 将策略合并在一起

# 总结

# 备注

* 原文：[https://resources.whitesourcesoftware.com/blog-whitesource/kubernetes-pod-security-policy](https://resources.whitesourcesoftware.com/blog-whitesource/kubernetes-pod-security-policy)
