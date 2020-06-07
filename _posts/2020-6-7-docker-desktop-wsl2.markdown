---
layout: post
title:  "在 Windows WSL 2 中使用 Docker Desktop"
date:   2020-6-7 13:12:40 +0800
categories: sre k8s
---

# 版本选择
 
WSL 2 已支持在 Windows 家庭版中运行 Docker Desktop，在此选择使用 WSL 2 运行 Docker Desktop。
 
详细信息可参考 https://docs.docker.com/docker-for-windows/install-windows-home/ 和 https://docs.docker.com/docker-for-windows/install-windows-home/。
 
# 安装过程
 
## 要求
 
* Windows 10，版本 2004 或更高，可以使用 `winver` 命令查看。
 
* 开启 Windows 的 WSL 2 功能，操作请参考 https://docs.microsoft.com/zh-cn/windows/wsl/install-win10。
 
* 硬件要求
 
  * 具有二级地址转换（SLAT）的 64 位处理器
  * 4GB 以上系统内存
  * BIOS 中开启硬件虚拟化
 
* 下载并安装 [Linux 内核更新包](https://docs.microsoft.com/windows/wsl/wsl2-kernel)。
 
## 安装
 
访问 https://hub.docker.com/editions/community/docker-ce-desktop-windows/ 下载安装包。
 
安装时请勾选 `Enable WSL 2 Windows Features`。
 
![docker-desktop-wsl2-1](/assets/img/docker-desktop-wsl2-1.png)

启动界面如下：

![docker-desktop-wsl2-2](/assets/img/docker-desktop-wsl2-2.png)

可以点击启动界面中的 `Start` 按钮学习基础的使用方法。


# 使用

* 在 PowerShell 中查看 wsl 状态

```shell
❯ wsl -l -v
  NAME STATE VERSION
* Ubuntu Running 2
  docker-desktop-data Running 2
  docker-desktop Running 2
```

* 在 wsl 的 Ubuntu 发行版中查看 docker 信息

```shell
❯ docker version
Client: Docker Engine - Community
 Version: 19.03.8
 API version: 1.40
 Go version: go1.12.17
 Git commit: afacb8b7f0
 Built: Wed Mar 11 01:25:46 2020
 OS/Arch: linux/amd64
 Experimental: false

Server: Docker Engine - Community
 Engine:
  Version: 19.03.8
  API version: 1.40 (minimum version 1.12)
  Go version: go1.12.17
  Git commit: afacb8b
  Built: Wed Mar 11 01:29:16 2020
  OS/Arch: linux/amd64
  Experimental: false
 containerd:
  Version: v1.2.13
  GitCommit: 7ad184331fa3e55e52b890ea95e65ba581ae3429
 runc:
  Version: 1.0.0-rc10
  GitCommit: dc9208a3303feef5b3839f4323d9beb36df0a9dd
 docker-init:
  Version: 0.18.0
  GitCommit: fec3683
```