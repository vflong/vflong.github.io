---
layout: post
title:  "一台阿里云主机的配置记录"
date:   2019-12-18 18:43:19 +0800
categories: sre
---

    朋友有一台闲置的阿里云 ECS 实例，拿来做一些有意义的事。


## 安全加固

> 由于长期未使用，且默认使用了 root 账户，22 号端口，密码登录的方式，早已中毒，因此直接更换操作系统，删除数据，从头再来。

```bash
# 生成密钥，更换操作系统过程中添加下面生成的公钥
$ ssh-keygen -P "xxx" -N "" -f .ssh/yangweijie    
Generating public/private rsa key pair.
Your identification has been saved in .ssh/yangweijie.
Your public key has been saved in .ssh/yangweijie.pub.

# 配置 host，设置此主机的别名为 C-3PO
$ cat /etc/hosts
aliyunecsip C-3PO

# 更换操作系统完成后使用密钥登录
# 配置 ssh 密钥，其中 myhost 为阿里云 ECS 的公网 IP
$ cat .ssh/config                              
Host github.com
    Hostname github.com
        IdentityFile ~/.ssh/github_rsa 

Host C-3PO
    Hostname C-3PO
        IdentityFile ~/.ssh/yangweijie

# 登录
$ ssh root@yangweijie
Enter passphrase for key '/home/feilong/.ssh/yangweijie': 

# 创建普通用户并配置 sudo 权限

# 更改 ssh 默认的 22 号端口，禁止使用密码登录，并禁止 root 用户直接登录
```