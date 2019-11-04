---
layout: post
title:  "Ubuntu 使用笔记"
date:   2019-11-4 22:42:07 +0800
categories: sre
---

    Ubuntu 安装、配置等笔记。


## 安装

> 在 VMware 中安装。

## 初始化

* 修改镜像源为 "mirrors.aliyun.com"，具体内容请参考[阿里巴巴开源镜像站](https://opsx.alibaba.com/mirror?lang=zh-CN)。

* 更新系统并安装软件。

```bash
$ sudo -i
# apt update -y && apt upgrade -y
# apt install vim tmux htop git -y
```

## 安装 Docker
```bash
# 参考 https://docs.docker.com/install/linux/docker-ce/ubuntu/
$ curl -fsSL https://get.docker.com -o get-docker.sh
$ sudo sh get-docker.sh

# 配置阿里云镜像加速器，https://cr.console.aliyun.com/cn-hangzhou/instances/mirrors
# 登录阿里云控制台之后获取私人加速地址，替换 your-code
# 加速服务免费，https://help.aliyun.com/document_detail/64340.html
# sudo mkdir -p /etc/docker
# sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://your-code.mirror.aliyuncs.com"]
}
EOF
# sudo systemctl daemon-reload
# sudo systemctl restart docker

# 测试
# docker run hello-world

# 非 root 用户使用 Docker
# sudo usermod -aG docker feilong
```

## stunnel4 配置

```bash
$ sudo apt-get install stunnel4 -y
$ sudo vim /etc/stunnel/stunnel.conf
$ sudo vim /etc/stunnel/stunnel.pem
$ sudo sed -i 's/ENABLED=0/ENABLED=1/g' /etc/default/stunnel4
$ sudo systemctl restart stunnel4.service 
$ sudo systemctl status stunnel4.service 
$ export https_proxy='127.0.0.1:3080'; curl -vI "https://www.google.com/robots.txt"
```

## 配置 git 环境

```bash
# 生成密钥，并将公钥添加至 GitHub SSH Keys
$ ssh-keygen 

$ git clone git@github.com:vflong/istio.io.git
```