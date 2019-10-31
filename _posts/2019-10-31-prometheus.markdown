---
layout: post
title:  "Prometheus 笔记"
date:   2019-10-31 22:53:11 +0800
categories: sre
---

    Prometheus 安装、调试等笔记。

## 本地学习

### 本地环境信息

* 操作系统：Windows 10
* 终端工具：Windows Terminal (Preview) Version: 0.6.2951.0

### 操作过程

#### 监控本机的配置
配置文件：prometheus.yml

```yml
global:
  scrape_interval:     15s # By default, scrape targets every 15 seconds.

  # Attach these labels to any time series or alerts when communicating with
  # external systems (federation, remote storage, Alertmanager).
  external_labels:
    monitor: 'codelab-monitor'

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s

    static_configs:
      - targets: ['localhost:9090']
```

#### 启动 Prometheus

```bat
PS C:\prometheus> .\prometheus.exe --config.file=prometheus.yml
:: http://localhost:9090/graph
```

#### 启动样本目标
```bat
:: 安装 random 命令
PS C:\Users\feilong\go\src\github.com> go env
set GOARCH=amd64
set GOBIN=
set GOCACHE=C:\Users\feilong\AppData\Local\go-build
set GOEXE=.exe
set GOFLAGS=
set GOHOSTARCH=amd64
set GOHOSTOS=windows
set GOOS=windows
set GOPATH=C:\Users\feilong\go
set GOPROXY=
set GORACE=
set GOROOT=C:\Go
set GOTMPDIR=
set GOTOOLDIR=C:\Go\pkg\tool\windows_amd64
set GCCGO=gccgo
set CC=gcc
set CXX=g++
set CGO_ENABLED=1
set GOMOD=
set CGO_CFLAGS=-g -O2
set CGO_CPPFLAGS=
set CGO_CXXFLAGS=-g -O2
set CGO_FFLAGS=-g -O2
set CGO_LDFLAGS=-g -O2
set PKG_CONFIG=pkg-config
set GOGCCFLAGS=-m64 -mthreads -fno-caret-diagnostics -Qunused-arguments -fmessage-length=0 -fdebug-prefix-map=C:\Users\feilong\AppData\Local\Temp\go-build367024130=/tmp/go-build -gno-record-gcc-switches

PS C:\Users\feilong\go\src\github.com> git clone https://github.com/prometheus/client_golang.git

PS C:\Users\feilong\go\src\github.com> cd client_golang/examples/random
PS C:\Users\feilong\go\src\github.com\client_golang\examples\random> go get -d
PS C:\Users\feilong\go\src\github.com\client_golang\examples\random> go build

:: 在三个终端分别执行
PS C:\Users\feilong\go\src\github.com\client_golang\examples\random> .\random.exe -listen-address=:8080
PS C:\Users\feilong\go\src\github.com\client_golang\examples\random> .\random.exe -listen-address=:8081
PS C:\Users\feilong\go\src\github.com\client_golang\examples\random> .\random.exe -listen-address=:8082
```

#### 配置 Prometheus 监控样本目标
```yml
scrape_configs:
  - job_name:       'example-random'

    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s

    static_configs:
      - targets: ['localhost:8080', 'localhost:8081']
        labels:
          group: 'production'

      - targets: ['localhost:8082']
        labels:
          group: 'canary'
```