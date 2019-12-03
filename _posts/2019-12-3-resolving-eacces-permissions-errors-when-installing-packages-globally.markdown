---
layout: post
title:  "解决 npm 全局安装软件包时 EACCES 权限错误"
date:   2019-12-2 17:29:39 +0800
categories: sre
---

    在容器内编译前端代码时发现了一个问题。

### 解决

```bash
# 在 npm install 命令前增加以下命令
mkdir ~/.npm-global
npm config set prefix '~/.npm-global'
export PATH=~/.npm-global/bin:$PATH

# 容器内编译，可以省略 source 命令
source ~/.profile
```

### 参考

* [resolving-eacces-permissions-errors-when-installing-packages-globally](https://docs.npmjs.com/resolving-eacces-permissions-errors-when-installing-packages-globally)
