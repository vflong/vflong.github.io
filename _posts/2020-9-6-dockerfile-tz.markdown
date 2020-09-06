---
layout: post
title:  "Dockerfile 修改时区"
date:   2020-9-6 15:43:48 +0800
categories: jekyll
---

## 介绍

创建 Docker 镜像时经常遇到时区问题，在此记录了常用的底层镜像修改时区的 `Dockerfile`。

## Alpine

```dockerfile
FROM alpine:3

LABEL maintainer="weifeilong2013@gmail.com"

ARG TZ="Asia/Shanghai"

RUN apk update \
        && apk add --no-cache tzdata \
        && cp /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone \
        && rm -rf /var/cache/apk/
```

## CentOS

```dockerfile
FROM centos:7

LABEL maintainer="weifeilong2013@gmail.com"

ENV TZ=Asia/Shanghai
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone
```