---
layout: post
title:  "一台阿里云主机的配置记录"
date:   2019-12-18 18:43:19 +0800
categories: sre
---

    朋友有一台闲置的阿里云 ECS 实例，拿来做一些有意义的事。


## 基础设置

### SSH 配置

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
# 配置 ssh 密钥
$ cat .ssh/config                              
Host github.com
    Hostname github.com
        IdentityFile ~/.ssh/github_rsa 

Host C-3PO
    Hostname C-3PO
        IdentityFile ~/.ssh/yangweijie

# 登录
$ ssh root@C-3PO
Enter passphrase for key '/home/feilong/.ssh/yangweijie': 

# 创建普通用户并配置 sudo 权限
[root@iZwz93nze2swrww2w3egwfZ ~]# useradd weifeilong
[root@iZwz93nze2swrww2w3egwfZ ~]# passwd weifeilong
Changing password for user weifeilong.
New password: 
Retype new password: 
passwd: all authentication tokens updated successfully.
[root@iZwz93nze2swrww2w3egwfZ ~]# id weifeilong
uid=1000(weifeilong) gid=1000(weifeilong) groups=1000(weifeilong)
[root@iZwz93nze2swrww2w3egwfZ ~]# usermod -a -G wheel weifeilong
[root@iZwz93nze2swrww2w3egwfZ ~]# id weifeilong
uid=1000(weifeilong) gid=1000(weifeilong) groups=1000(weifeilong),10(wheel)

# 为普通用户添加密钥
# 阿里云 ECS 实例，在更换操作系统时，选择密钥认证后，已默认关闭密码登录，即 PasswordAuthentication no
# 需要使用 root 账户为上面刚刚创建的普通用户添加密钥
# 在本地主机生成密钥，复制 .ssh/weifeilong_ywj.pub
$ ssh-keygen -P "xxx" -N "" -f .ssh/weifeilong_ywj

# 连接主机手动添加公钥
[root@iZwz93nze2swrww2w3egwfZ ~]# su - weifeilong
[weifeilong@iZwz93nze2swrww2w3egwfZ ~]$ mkdir .ssh
# 粘贴 .ssh/weifeilong_ywj.pub
[weifeilong@iZwz93nze2swrww2w3egwfZ ~]$ vim .ssh/authorized_keys
[weifeilong@iZwz93nze2swrww2w3egwfZ ~]$ chmod 700 .ssh
[weifeilong@iZwz93nze2swrww2w3egwfZ ~]$ chmod 600 .ssh/authorized_keys 

# 连接测试
$ ssh -i .ssh/weifeilong_ywj weifeilong@C-3PO              
Enter passphrase for key '.ssh/weifeilong_ywj': 

# 更改 ssh 默认的 22 号端口，并禁止 root 用户直接登录
[weifeilong@iZwz93nze2swrww2w3egwfZ ~]$ sudo vim /etc/ssh/sshd_config
[weifeilong@iZwz93nze2swrww2w3egwfZ ~]$ sudo tail -n 6 /etc/ssh/sshd_config 
UseDNS no
AddressFamily inet
SyslogFacility AUTHPRIV
PermitRootLogin no
PasswordAuthentication no
Port 1222
[weifeilong@iZwz93nze2swrww2w3egwfZ ~]$ sudo systemctl restart sshd

# 在本地主机连接测试
# 测试 root 账户
$ ssh -i .ssh/yangweijie -p 1222 root@C-3PO       
Enter passphrase for key '.ssh/yangweijie': 
root@c-3po: Permission denied (publickey,gssapi-keyex,gssapi-with-mic)

# 测试普通账户
$ ssh -i .ssh/weifeilong_ywj weifeilong@C-3PO  
ssh: connect to host c-3po port 22: Connection refused
# feilong @ feilong-PC in ~ [10:15:44] C:255
$ ssh -i .ssh/weifeilong_ywj -p 1222 weifeilong@C-3PO
```

### 系统配置

```bash
# 更新系统
[weifeilong@iZwz93nze2swrww2w3egwfZ ~]$ sudo yum update -y

# 修改主机名 hostname
[weifeilong@iZwz93nze2swrww2w3egwfZ ~]$ cat /etc/hostname
iZwz93nze2swrww2w3egwfZ
[weifeilong@iZwz93nze2swrww2w3egwfZ ~]$ hostnamectl set-hostname C-3PO
==== AUTHENTICATING FOR org.freedesktop.hostname1.set-static-hostname ===
Authentication is required to set the statically configured local host name, as well as the pretty host name.
Authenticating as: weifeilong
Password: 
==== AUTHENTICATION COMPLETE ===
[weifeilong@iZwz93nze2swrww2w3egwfZ ~]$ hostname
c-3po
[weifeilong@iZwz93nze2swrww2w3egwfZ ~]$ sudo reboot
```

### 配置 SHELL 

```bash
[weifeilong@c-3po ~]$ sudo yum install zsh git
[weifeilong@c-3po ~]$ wget https://raw.github.com/robbyrussell/oh-my-zsh/master/tools/install.sh
[weifeilong@c-3po ~]$ sh -x install.sh
➜  ~ $ git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
➜  ~ vim .zshrc
➜  ~ grep -A 5 "plugins=" .zshrc
# Example format: plugins=(rails git textmate ruby lighthouse)
# Add wisely, as too many plugins slow down shell startup.
plugins=(
  git
  zsh-autosuggestions
  kubectl
  docker
)
➜  ~ source .zshrc
```