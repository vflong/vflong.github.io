---
layout: post
title:  "【译文】如何将 PFX 转换为单独的 .key/.crt 文件"
date:   2020-3-12 10:01:34 +0800
categories: sre https
---

    译者注：收到一个 pfx 文件，需要使用此文件创建替换当前 kong 所用的 https 证书。

本文将向您展示将 .PFX 证书文件转换为单独的证书和密钥文件所需的命令。当您需要在诸如 Cisco 路由器/负载均衡器等设备上导入证书时，本文可以派上用场。您可能需要以纯文本格式（未加密）导入证书和密钥文件。首选工具（但可能还有其他工具）是适用于 Windows 的 OpenSSL，可以在[此处](https://www.slproweb.com/products/Win32OpenSSL.html)下载。

# 1. 安装 OpenSSL —— 从 Bin 文件夹启动它。

# 2. 启动 CMD 并 cd 到包含 .pfx 文件的文件夹。

# 3. 提取私钥

首先输入第一个命令以提取私钥：

```bash
openssl pkcs12 -in [yourfile.pfx] -nocerts -out [keyfile-encrypted.key]
```

该命令的作用是从 .pfx 文件中提取私钥。输入后，您需要输入 .pfx 文件的 `import password`（若有）。这是创建 .pfx 文件时用来保护密钥对的密码。如果您不记得它了，可以将 .pfx 文件丢掉，因为您将无法在任何地方再次导入它！输入导入密码后，OpenSSL 要求您输入另一个密码两次！此新密码将保护您的 .key 文件。

# 4. 提取证书

```bash
openssl pkcs12 -in [yourfile.pfx] -clcerts -nokeys -out [certificate.crt]
```

只需按下 Enter 键，您的证书就会出现。

# 5. 提取未加密私钥

现在，正如我在本文的简介中提到的那样，有时您需要具有未加密的 .key 文件才能在某些设备上导入。我可能无需提及您应该小心。如果您将未加密的密钥对存储在不安全的位置上，那么任何人都可以使用它并假冒您公司或个人的网站。因此，在私钥方面要格外小心！使用完后，只需将未加密的密钥文件扔掉即可，只保存加密的密钥文件。

命令如下：

```bash
openssl rsa -in [keyfile-encrypted.key] -out [keyfile-decrypted.key]
```

同样，您需要输入一个导入密码。这次，您需要输入在步骤 1 中创建的新密码。之后，您就完成了。您已解密私钥。在您从中运行 OpenSSL 的文件夹中，您将找到证书（.crt）和两个私钥（加密和未加密）。

# 6. 获取 PEM 格式私钥

在某些情况下，您可能需要将私钥转换为 PEM 格式。您可以使用以下命令实现：

```bash
openssl rsa -in [keyfile-encrypted.key] -outform PEM -out [keyfile-encrypted-pem.key]
```

# 备注

* 原文：[https://www.itsupportmiami.com/how-to-convert-a-pfx-to-a-seperate-key-crt-file/](https://www.itsupportmiami.com/how-to-convert-a-pfx-to-a-seperate-key-crt-file/)