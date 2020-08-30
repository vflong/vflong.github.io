---
layout: post
title:  "修改 Docker 数据根目录的 3 种方式"
date:   2020-8-30 11:33:25 +0800
categories: sre k8s
---


# 需求

Docker 的镜像以及运行时产生的数据会占据大量的磁盘空间，因此有必要将此目录转移至大容量磁盘。

环境信息

```shell
# 系统版本
cat /etc/redhat-release 
CentOS Linux release 7.8.2003 (Core)

# docker 版本
docker -v
Docker version 19.03.12, build 48a66213fe
```

# 实现

## 方式一：自定义配置（推荐）

> There are a number of ways to configure the daemon flags and environment variables for your Docker daemon. The recommended way is to use the platform-independent `daemon.json` file, which is located in `/etc/docker/` on Linux by default.
>
> 推荐使用 `/etc/docker/daemon.json` 自定义配置。

操作过程

```shell
# 关闭 docker
systemctl stop docker
systemctl status docker

# 移动文件
mv /var/lib/docker /data

# 新增文件 /etc/docker/daemon.json，默认不存在
cat /etc/docker/daemon.json 
{
    "data-root": "/data/docker"
}

# 重启 docker，start 无效
systemctl restart docker

# 查看镜像是否仍存在
docker images
```

## 方式二：软链接

>Docker supports softlinks for the Docker data directory (`/var/lib/docker`) and for `/var/lib/docker/tmp`.
>
>Docker 的数据目录支持软链接。

操作过程

```shell
# 查看 docker 数据目录大小
du -sh /var/lib/docker
8.9G /var/lib/docker

# 关闭 docker
systemctl stop docker
systemctl status docker

# 移动文件
mv /var/lib/docker /data

# 创建软链接
ln -s /data/docker /var/lib/

# 启动 docker
systemctl start docker

# 查看镜像是否仍存在
docker images

# 查看 docker 数据目录大小
du -sh /var/lib/docker
0 /var/lib/docker

du -sh /var/lib/docker/
12G /var/lib/docker/

du -sh /data/docker/
12G /data/docker/
```

## 方式三：修改启动参数（不推荐）

> --data-root string                      Root directory of persistent Docker state (default "/var/lib/docker")
> --data-root string                      Docker 状态持久化的根目录（默认“/var/lib/docker”）

操作过程

```shell
# 查看 docker 数据目录大小
du -sh /var/lib/docker
4.8G /var/lib/docker

# 关闭 docker
systemctl stop docker
systemctl status docker

# 移动文件
mv /var/lib/docker /data

# 增加参数 --data-root /data/docker
vim /etc/systemd/system/docker.service
ExecStart=/usr/bin/dockerd \
          --data-root /data/docker \
          $DOCKER_OPTS \
          $DOCKER_STORAGE_OPTIONS \
          $DOCKER_NETWORK_OPTIONS \
          $DOCKER_DNS_OPTIONS \
          $INSECURE_REGISTRY

# reload
systemctl daemon-reload

# 启动 docker
systemctl start docker

# 查看镜像是否仍存在
docker images
```

# 参考

* https://docs.docker.com/engine/reference/commandline/dockerd/
* https://docs.docker.com/config/daemon/systemd/


