---
layout: post
title:  "【译文】kubectl port-forward 的工作原理"
date:   2020-3-15 17:55:28 +0800
categories: sre k8s
---

今天下午，我正在使用 [Kubernetes: Up and Running](http://amzn.to/2EAT0n3) 一书中的 [kuard demo 程序](https://github.com/kubernetes-up-and-running/kuard) 深入研究 kube-dns 服务。Kuard 有一个“服务器端 DNS 查询”菜单，您可以用于在 Pod 中解析域名并在浏览器中显示结果。为了创建一个用于测试的 kuard 实例，我运行了可信赖的老朋友 kubectl：

```bash
$ kubectl run --image=gcr.io/kuar-demo/kuard-amd64:1 kuard
```

以上命令生成了一个 Pod：

```bash
$ kubectl get pods -o wide
NAME                     READY     STATUS    RESTARTS   AGE       IP         NODE
kuard-59f4bf4795-m9bzb   1/1       Running   0          30s       10.1.4.8   kubworker4.prefetch.net
```

为了获得对 kuard 的访问权限，我在 Pod 中创建了一个从 `localhost:8080` 到端口 8080 的 port-forward 端口转发：

```bash
$ kubectl port-forward kuard 8080:8080
Forwarding from 127.0.0.1:8080 -> 8080
```

当我尝试在 chrome 浏览器中访问 [localhost:8080](localhost:8080) 时，我收到了如下错误：

```bash
Handling connection for 8080
E0203 14:12:54.651409   20574 portforward.go:331] an error occurred forwarding 8080 -> 8080: error forwarding port 8080 to pod febaeb6b4747d87036534845214f391db41cda998a592e541d4b6be7ff615ef4, uid : unable to do port forwarding: socat not found.
```

这些错误很容易理解，node 节点没有安装 socat 二进制文件。我没有回想起 socat 工具的使用前提条件，所以我决定开始研究 kubelet 源码，以了解 port-forward 的工作原理。在查看 kubelet 源代码之后，我找到了文件 [docker_streaming.go](https://github.com/kubernetes/kubernetes/blob/master/pkg/kubelet/dockershim/docker_streaming.go)。改文件包含了一个函数 `portForward(...)`，其中包含端口转发逻辑。port-forward 的代码实际上非常简单，并使用了 socat 和 nsenter 完成其工作。首先，该函数检查 socat 和 nsenter 是否存在：

```go
        containerPid := container.State.Pid
        socatPath, lookupErr := exec.LookPath("socat")
        if lookupErr != nil {
                return fmt.Errorf("unable to do port forwarding: socat not found.")
        }

        args := []string{"-t", fmt.Sprintf("%d", containerPid), "-n", socatPath, "-", fmt.Sprintf("TCP4:localhost:%d", port)}

        nsenterPath, lookupErr := exec.LookPath("nsenter")
        if lookupErr != nil {
                return fmt.Errorf("unable to do port forwarding: nsenter not found.")
        }
```

如果两项检查均通过，它将使用 `exec() nsenter` 将目标进程 id（[pause 容器](https://www.ianlewis.org/en/almighty-pause-container) 的 PID）传递给“-t”，并将命令传递给“-n”：

```go
        commandString := fmt.Sprintf("%s %s", nsenterPath, strings.Join(args, " "))
        glog.V(4).Infof("executing port forwarding command: %s", commandString)

        command := exec.Command(nsenterPath, args...)
        command.Stdout = stream
```

这可以使用 [Brendan Gregg's execsnoop 工具](https://github.com/iovisor/bcc/blob/master/tools/execsnoop.py) 进行验证：

```bash
$ execsnoop -n nsenter
PCOMM            PID    PPID   RET ARGS
nsenter          25898  976      0 /usr/bin/nsenter -t 4947 -n /usr/bin/socat - TCP4:localhost:8080

```

我喜欢阅读代码，并且对 port-forward 现在的工作原理有了更好的了解。如果您使用此功能访问您的节点中的 Pod，则需要确保节点中已安装 nsenter 和 socat。

---

# 原文

* [https://prefetch.net/blog/2018/02/03/how-the-kubectl-port-forward-command-works/](https://prefetch.net/blog/2018/02/03/how-the-kubectl-port-forward-command-works/)