---
layout: post
title:  "【译文】创建 Kubernetes manifest 的初学者指南"
date:   2020-3-15 23:24:48 +0800
categories: sre k8s
---

作为 Kubernetes 长期用户，我最常听到的问题是“我该如何创建 manifest（描述如何在集群中创建和管理资源的文件）？”当我问提出问题的人他们当前如何创建资源时，我经常听到他们将在网上找到的一堆随机 manifest 拼凑在一起，或者根据网站的建议使用 `$(kubectl -f http://site/manifest)`。

当我开始使用 Kubernetes 时，学习如何从头开始生成 manifest 令我感到困惑。我找不到完整的指南来说明如何从头开始创建资源，并且精通此过程所需的信息分散在各个站点。为了帮助刚进入 Kubernetes 领域的人们，我想我应该记录一下我用来处理“如何从头开始创建 manifest”的过程这个问题。

因此，让我们从基础开始。Kubernetes manifest 描述了您要创建的资源（例如，Deployment、Service、Pod 等），以及希望这些资源在集群中运作的方式。接下来，我将描述如何更多地了解每种资源类型。当您在 manifest 中定义资源时，它将包含以下 4 个字段：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  ...
spec:
  ...
```

* apiVersion：该字段指定要用于创建资源的 API 组和要使用的 API 版本。Kubernetes API 被聚合到 API 组中，这允许 API server 按目的对 API 进行分组。如果我们分析 apiVersion 行，则“apps”将是 API 组，而 v1 将是要使用的 apps API 的版本。要列出可用的 API 组及其版本，您可以使用“api-versions”选项运行 kubectl：

```bash
$ kubectl api-versions |more
admissionregistration.k8s.io/v1beta1
apiextensions.k8s.io/v1beta1
apiregistration.k8s.io/v1
apiregistration.k8s.io/v1beta1
...
```

