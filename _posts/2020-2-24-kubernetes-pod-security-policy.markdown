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

需要澄清的一点是：不要让 *Pod 安全策略* 一词使您感到困惑。尽管这个名称听起来像 Pod 安全策略定义了特定 Pod 的安全设置，但实际上相反。

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

首先，请确保已启用 Kubernetes Pod 安全策略。这似乎不是最佳实践，但由于某些 Kubernetes 发行版默认情况下不启用 Pod 安全策略，因此请务必确保这样做。

某些 Kubernetes 发行版根本不支持 Pod 安全策略。在这种情况下，您不妨考略切换到其他发行版。在其他情况下，系统会支持 Pod 安全策略，但在 Kubernetes 启动时默认不会启用该策略。在您需要的情况下，请查看供应商文档以确定在 Kubernetes 启动时如何启用 Pod 安全策略。通常，您可以使用命令行参数来执行此操作。

## 禁用特权容器

在 Kubernetes Pod 中，容器可以选择以“特权（privileged）”模式运行，这意味着容器内的进程几乎可以不受限制地访问主机系统上的资源。尽管在某些情况下需要这种级别的访问，但通常来说，让您的容器执行此操作会带来安全风险。

通过在 Kubernetes Pod 安全策略中包含以下内容，可以轻松阻止特权容器运行：

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: prevent-privileged-containers
spec:
  privileged: false
  # The rest fills in some required fields.
  seLinux:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  runAsUser:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  volumes:
  - '*'
```

> 译者注：原文缺少了一些必填字段，导致无法直接使用此策略，译文已添加。

## 只读文件系统

Kubernetes 另一种安全最佳实践是要求容器与只读文件系统一起运行。这对于实施不可变基础设施策略非常有用，因为更新文件系统不可变的容器必须要更新容器。这样可以减轻恶意进程在容器内部存储或操作数据的风险。

您可以使用以下 Kubernetes Pod 安全配置来强制执行只读文件系统策略：

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: read-only-fs
spec:
  readOnlyRootFilesystem: true
  # The rest fills in some required fields.
  seLinux:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  runAsUser:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  volumes:
  - '*'
```

> 译者注：原文缺少了一些必填字段，导致无法直接使用此策略，译文已添加。

可以肯定的是，只读文件系统并非在所有情况下都可行，但如果您没有充分的里有允许对容器的内部数据进行写访问，请考虑使用只读文件系统。

## 防止特权升级

对于大多数管理员来说，*特权升级*是个坏词。在极少数情况下，您希望允许某个应用程序或进程获取比首次启动时更多的权限。

因此，您可能会认为 Kubernetes 默认情况下会阻止特权升级，但是您错了。为了禁止您的集群上的特权升级，您必须使用 Kubernetes Pod 安全策略，如下所示：

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: no-privilege-escalation
spec:
  allowPrivilegeEscalation: false
  # The rest fills in some required fields.
  seLinux:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  runAsUser:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  volumes:
  - '*'
```

> 译者注：原文缺少了一些必填字段，导致无法直接使用此策略，译文已添加。

## 防止容器以 root 身份运行

以 root 用户身份运行程序是另一种使大多数管理员不寒而栗的做法。通常，没有充分的理由需要以 root 身份运行程序。这几乎与键入 `chmod -R 777` 并按 `Enter` 一样。

幸运的是，Kubernetes Pod 安全策略提供了一种防止容器以 root 身份运行的简单方法：

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: must-run-as-nonroot
spec:
  # The rest fills in some required fields.
  seLinux:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  runAsUser:
    rule: MustRunAsNonRoot
  fsGroup:
    rule: RunAsAny
  volumes:
  - '*'
```

> 译者注：原文缺少了一些必填字段，导致无法直接使用此策略，译文已添加。

## 将策略合并在一起

我们上面查看的 Kubernetes Pod 安全策略是作为单个文件编写的。您可以根据需要以这种方式配置集群，但是生产部署中，将所有设置组合到一个文件中更有意义。这样，您只有一个文件可以写入和激活。

指定多个安全指令的文件如下所示：

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: example
spec:
  privileged: false
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  # The rest fills in some required fields.
  seLinux:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  runAsUser:
    rule: MustRunAsNonRoot
  fsGroup:
    rule: RunAsAny
  volumes:
  - '*'
```

> 译者注：原文缺少了一些必填字段，导致无法直接使用此策略，译文已添加。

# 总结

如我们所见，Kubernetes Pod 安全策略提供了一种方便的设置方法，可以自动在整个集群中实施强大的安全设置。尽管此主题可能不会在您与同事讨论的令人振奋的事情上排名很高，但是您不应忽略 Pod 安全策略作为确保 Kubernetes 和其中运行的容器安全的一种方式的重要型。

# 补充操作命令记录

```bash
# 测试环境
$ k version      
Client Version: version.Info{Major:"1", Minor:"16", GitVersion:"v1.16.3-k3s.2", GitCommit:"e7e6a3c4e9a7d80b87793612730d10a863a25980", GitTreeState:"clean", BuildDate:"2019-11-18T18:31:23Z", GoVersion:"go1.13.4", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"16", GitVersion:"v1.16.3-k3s.2", GitCommit:"e7e6a3c4e9a7d80b87793612730d10a863a25980", GitTreeState:"clean", BuildDate:"2019-11-18T18:31:23Z", GoVersion:"go1.13.4", Compiler:"gc", Platform:"linux/amd64"}

# 禁用特权容器
$ k create -f prevent-privileged-containers.yaml
podsecuritypolicy.policy/prevent-privileged-containers created

$ k get psp prevent-privileged-containers       
NAME                            PRIV    CAPS   SELINUX    RUNASUSER   FSGROUP    SUPGROUP   READONLYROOTFS   VOLUMES
prevent-privileged-containers   false          RunAsAny   RunAsAny    RunAsAny   RunAsAny   false            *

# 只读文件系统
$ k create -f read-only-fs.yaml
podsecuritypolicy.policy/read-only-fs created

# root @ feilong-Lenovo in ~ [22:09:27] 
$ k get psp read-only-fs                  
NAME           PRIV    CAPS   SELINUX    RUNASUSER   FSGROUP    SUPGROUP   READONLYROOTFS   VOLUMES
read-only-fs   false          RunAsAny   RunAsAny    RunAsAny   RunAsAny   true             *

# 防止特权升级
$ k create -f no-privilege-escalation.yaml 
podsecuritypolicy.policy/no-privilege-escalation created

$ k get psp no-privilege-escalation       
NAME                      PRIV    CAPS   SELINUX    RUNASUSER   FSGROUP    SUPGROUP   READONLYROOTFS   VOLUMES
no-privilege-escalation   false          RunAsAny   RunAsAny    RunAsAny   RunAsAny   false            *

# 防止以 root 身份运行
$ k create -f must-run-as-nonroot.yaml    
podsecuritypolicy.policy/must-run-as-nonroot created

$ k get psp must-run-as-nonroot                                    
NAME                  PRIV    CAPS   SELINUX    RUNASUSER          FSGROUP    SUPGROUP   READONLYROOTFS   VOLUMES
must-run-as-nonroot   false          RunAsAny   MustRunAsNonRoot   RunAsAny   RunAsAny   false            *

# 所有策略汇总
$ k create -f example.yaml            
podsecuritypolicy.policy/example created

$ k get psp example            
NAME      PRIV    CAPS   SELINUX    RUNASUSER          FSGROUP    SUPGROUP   READONLYROOTFS   VOLUMES
example   false          RunAsAny   MustRunAsNonRoot   RunAsAny   RunAsAny   true             *

# 查看所有策略
$ k get psp                             
NAME                            PRIV    CAPS   SELINUX    RUNASUSER          FSGROUP    SUPGROUP   READONLYROOTFS   VOLUMES
prevent-privileged-containers   false          RunAsAny   RunAsAny           RunAsAny   RunAsAny   false            *
read-only-fs                    false          RunAsAny   RunAsAny           RunAsAny   RunAsAny   true             *
no-privilege-escalation         false          RunAsAny   RunAsAny           RunAsAny   RunAsAny   false            *
must-run-as-nonroot             false          RunAsAny   MustRunAsNonRoot   RunAsAny   RunAsAny   false            *
example                         false          RunAsAny   MustRunAsNonRoot   RunAsAny   RunAsAny   true             *

```




# 备注

* 原文：[https://resources.whitesourcesoftware.com/blog-whitesource/kubernetes-pod-security-policy](https://resources.whitesourcesoftware.com/blog-whitesource/kubernetes-pod-security-policy)
* 参考：[https://kubernetes.io/zh/docs/concepts/policy/pod-security-policy/](https://kubernetes.io/zh/docs/concepts/policy/pod-security-policy/)

