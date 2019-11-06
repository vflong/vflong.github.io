---
layout: post
title:  "Ubuntu 使用笔记"
date:   2019-11-04 22:42:07 +0800
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

## 配置 SSH Server

```bash
$ sudo apt install openssh-server
$ systemctl status sshd
```

## 安装 oh-my-zsh

```bash
# https://ohmyz.sh/
$ sudo apt install zsh
$ wget https://raw.github.com/robbyrussell/oh-my-zsh/master/tools/install.sh
$ sh -x install.sh

# 安装 zsh-autosuggestions
# https://github.com/zsh-users/zsh-autosuggestions
$ git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
$ cat .zshrc | grep autosuggestions                            
plugins=(zsh-autosuggestions)
$ source .zshrc
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

# 非 root 用户使用 Docker，需要注销重新登录
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

## 配置 istio-official-translation 所需环境

> [istio-official-translation](https://github.com/servicemesher/istio-official-translation)

```bash
# 生成密钥，并将公钥添加至 GitHub SSH Keys
$ ssh-keygen 

# clone 源码
$ git clone git@github.com:vflong/istio.io.git
$ cd istio.io/
$ git config user.name 'feilong'
$ git config user.email 'weifeilong2013@gmail.com'
$ git remote add upstream git@github.com:istio/istio.io.git
$ git checkout -b zh-trans-1267
# 对比翻译
$ vim ./content/*/docs/reference/glossary/operator.md -O
$ git diff content/zh/docs/reference/glossary/operator.md
diff --git a/content/zh/docs/reference/glossary/operator.md b/content/zh/docs/reference/glossary/operator.md
index 0d277a53..58ccdff5 100644
--- a/content/zh/docs/reference/glossary/operator.md
+++ b/content/zh/docs/reference/glossary/operator.md
@@ -1,4 +1,4 @@
 ---
 title: Operator
 ---
-Operators are a method of packaging, deploying and managing a Kubernetes application. For more information, see [Operator pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/).
+Operator 是打包，部署和管理 Kubernetes 应用程序的一种方法。有关更多信息，请参见 [Operator pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)。

$ docker pull gcr.io/istio-testing/build-tools:2019-10-24T14-05-17
$ sudo apt install make
$ make serve
# 翻译段落的地址：http://localhost:1313/zh/docs/reference/glossary/#operator

# 检查
$ make INTERNAL_ONLY=True lint
Building with the build container: gcr.io/istio-testing/build-tools:2019-10-24T14-05-17.

                   | EN  | ZH   
+------------------+-----+-----+
  Pages            | 546 | 545  
  Paginator pages  |   0 |   0  
  Non-page files   | 164 | 164  
  Static files     |  54 |  54  
  Processed images |   0 |   0  
  Aliases          |   1 |   0  
  Sitemaps         |   2 |   1  
  Cleaned          |   0 |   0  

Total in 241551 ms
>> 430 files are free from spelling errors
>> 429 files are free from spelling errors
Running ["ImageCheck", "ScriptCheck", "HtmlCheck", "LinkCheck", "OpenGraphCheck"] on ["./public"] on *.html... 

# 提交并推送
$ git add .
$ git commit -m 'zh-translation:/docs/reference/glossary/operator.md'
$ git push --set-upstream origin zh-trans-1267

# 清理已 merge 的分支
$ git checkout master
$ git fetch upstream master
$ git merge upstream/master
$ git branch -d zh-trans-1267
$ git push origin :zh-trans-1267

# 继续解决下一个 issue
$ git checkout -b zh-trans-1094
$ vim ./content/*/docs/reference/glossary/adapters.md -o
$ git add .
$ git commit -m 'zh-translation:/docs/reference/glossary/adapters.md'
$ git push --set-upstream origin zh-trans-1094
```