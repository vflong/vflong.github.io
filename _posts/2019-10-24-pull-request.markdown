---
layout: post
title:  "第一次 pull request 记录"
date:   2019-10-24 18:34:04 +0800
categories: github
---

> 使用 helm 安装 incubator/zookeeper 过程中，发现了一个 BUG。
https://github.com/helm/charts/pull/18275

## 过程
1. fork [helm/charts](https://github.com/helm/charts) 仓库
2. 创建分支 `vflong-patch-1` 并修改代码提交
3. 发起 `pull request`

## 注意事项
* 直接在 github web 页面修改代码时需要添加 `Signed-off-by: Joe Smith <joe.smith@example.com>` 格式的认证信息，若未添加，可以重新 commit。
```bash
git clone git@github.com:vflong/charts.git -b vflong-patch-1
git config user.email 'weifeilong2013@gmail.com'
git config user.name 'feilong'
git commit --amend --signoff
git push --force-with-lease origin vflong-patch-1
```


## 参考
* [sign-your-work](https://github.com/helm/charts/blob/master/CONTRIBUTING.md#sign-your-work)
