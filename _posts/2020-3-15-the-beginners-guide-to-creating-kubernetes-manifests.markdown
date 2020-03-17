---
layout: post
title:  "【译文】创建 Kubernetes manifest 的初学者指南"
date:   2020-3-15 23:24:48 +0800
categories: sre k8s
---

作为 Kubernetes 长期用户，我最常听到的问题是“我该如何创建 manifest（描述如何在集群中创建和管理资源的文件）？”当我问提出问题的人他们当前如何创建资源时，我经常听到他们将在网上随机找到的一堆 manifest 拼凑在一起，或者根据网站的建议使用 `$(kubectl -f http://site/manifest)`。

当我开始使用 Kubernetes 时，学习如何从头开始生成 manifest 令我感到困惑。我找不到关于如何从头开始创建资源的完整指南，并且精通此过程所需的信息分散在各个站点。为了帮助刚进入 Kubernetes 领域的人们，我想我应该记录一下我用来处理“如何从头开始创建 manifest”的过程这个问题。

因此，让我们从基础开始。Kubernetes manifest 描述了您要创建的资源（例如，Deployment、Service、Pod 等），以及希望这些资源在集群中运作的方式。接下来，我将描述如何更多地了解每种资源类型。当您在 manifest 中定义资源时，它将包含以下 4 个字段：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  ...
spec:
  ...
```

# apiVersion

* apiVersion：该字段指定要用于创建资源的 API 组和要使用的 API 版本。Kubernetes API 被聚合到 API 组中，这允许 API server 按目的对 API 进行分组。如果我们分析 apiVersion 行，则“apps”将是 API 组，而 v1 将是要使用的 apps API 的版本。要列出可用的 API 组及其版本，您可以使用“api-versions”选项运行 kubectl：

```bash
$ kubectl api-versions | more
admissionregistration.k8s.io/v1beta1
apiextensions.k8s.io/v1beta1
apiregistration.k8s.io/v1
apiregistration.k8s.io/v1beta1
...
```

# kind

* kind：指定您要创建的资源的类型。Deployment、ReplicaSet、Cronjob、StatefulSet 等都是可以创建的资源示例。您可以使用 `kubectl api-resources` 命令查看可用的资源类型以及与它们关联的 API 组：

```bash
$ kubectl api-resources | more
NAME                              SHORTNAMES   APIGROUP                       NAMESPACED   KIND
daemonsets                        ds           apps                           true         DaemonSet
deployments                       deploy       apps                           true         Deployment
replicasets                       rs           apps                           true         ReplicaSet
statefulsets                      sts          apps                           true         StatefulSet
...
```

使用 `api-versions` 和 `api-resources` 命令，我们可以找到可用资源（KIND 列），与资源类型关联的 API 组（APIGROUP 列）以及 API 组版本（来自 `api-versions`）。此信息可用于填写 `apiVersion: ` 和 `kind: ` 字段。要了解每种资源类型的目的，可以使用 `kubectl explain` 命令：

```bash
$ kubectl explain --api-version=apps/v1 replicaset
KIND:     ReplicaSet
VERSION:  apps/v1

DESCRIPTION:
     ReplicaSet ensures that a specified number of pod replicas are running at
     any given time. 
```

这将为您提供有关作为参数传递的资源以及您可以填充的字段的详细说明。

# metadata

现在，我们已经学习了前两个字段，我们可以继续学习 `metadata: ` 和 `spec: `。`metadata: ` 部分用于唯一标识 Kubernetes 集群中的资源。有了它您可以为资源命名、分配标签、注解、指定命名空间等。要查看可以添加到 `metadata: ` 中的字段，您可以使用下面命令：

```bash
$ kubectl explain deployment.metadata | more
KIND:     Deployment
VERSION:  extensions/v1beta1

RESOURCE: metadata <Object>

DESCRIPTION:
     Standard object metadata.
                                                                                                                                             
     ObjectMeta is metadata that all persisted resources must have, which
     includes all objects users must create.

FIELDS:
   annotations  <map[string]string>
     Annotations is an unstructured key value map stored with a resource that
     may be set by external tools to store and retrieve arbitrary metadata. They
     are not queryable and should be preserved when modifying objects. More
     info: http://kubernetes.io/docs/user-guide/annotations
...
```

# spec

既然我们已经学习了前 3 个字段，那么我们来深入研究一下 manifest 的构成。`spec: ` 部分！本节介绍如何创建和管理资源。您可以定义要使用的容器镜像、ReplicaSet 中副本的数量、selector 的条件、存活或就绪探针定义等。要查看可以添加到 `spec: ` 部分的字段，您可以使用以下命令；

```bash
$ kubectl explain deployment.spec | more
KIND:     Deployment
VERSION:  extensions/v1beta1

RESOURCE: spec <Object>

DESCRIPTION:
     Specification of the desired behavior of the Deployment.

     DeploymentSpec is the specification of the desired behavior of the
     Deployment.

FIELDS:
   minReadySeconds      <integer>
     Minimum number of seconds for which a newly created pod should be ready
     without any of its container crashing, for it to be considered available.
     Defaults to 0 (pod will be considered available as soon as it is ready)
...
```

`kubectl explain` 在显示每个部分的值方面做得非常好，但是手动将它们组合在一起需要花费大量的时间和耐心。为了简化此过程，kubectl 开发人员提供了 `-o yaml` 和 `--dry-run` 选项。这些选项可以与 run 和 create 命令结合使用，以生成作为参数传递的资源的基础 manifest：

```bash
$ kubectl create deployment nginx --image=nginx -o yaml --dry-run
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: nginx
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        resources: {}
status: {}
```

一旦有了基础 manifest，就可以通过在 `spec: ` 和 `metadata: ` 部分中添加其他字段来扩展它。您还可以在 `kubectl explain` 中添加 `--recursive` 选项，以获取各个字段的分层视图。下面的示例显示如何递归显示可用于自定义容器字段的每个选项：

```bash
$ kubectl explain deployment.spec.template.spec.containers --recursive | more
KIND:     Deployment
VERSION:  extensions/v1beta1

RESOURCE: containers <[]Object>

DESCRIPTION:
     List of containers belonging to the pod. Containers cannot currently be
     added or removed. There must be at least one container in a Pod. Cannot be
     updated.

     A single application container that you want to run within a pod.

FIELDS:
   args <[]string>
   command      <[]string>
   env  <[]Object>
      name      <string>
      value     <string>
      valueFrom <Object>
         configMapKeyRef        <Object>
            key <string>
            name        <string>
            optional    <boolean>
...
```

如果您想了解有关特定字段的更多信息，可以通过传递它进行解释以获取更多的信息：

```bash
$ kubectl explain deployment.spec.selector.matchExpressions.operator
KIND:     Deployment
VERSION:  extensions/v1beta1

FIELD:    operator <string>

DESCRIPTION:
     operator represents a key's relationship to a set of values. Valid
     operators are In, NotIn, Exists and DoesNotExist.
```

我希望这种有关如何开始使用 Kubernetes manifest 的简短说明会有所帮助。这篇文章仍然是一个正在进行的工作，我计划在出现问题时将其添加到其中。如果您有任何问题或意见，请在 twitter 上联系我。

# 参考

* [Kubernetes Ojbects](https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/)
* [Kubernetes metadata 约定](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md#metadata)

---

# 备注

* 原文：[https://prefetch.net/blog/2019/10/16/the-beginners-guide-to-creating-kubernetes-manifests/](https://prefetch.net/blog/2019/10/16/the-beginners-guide-to-creating-kubernetes-manifests/)
