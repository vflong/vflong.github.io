---
layout: post
title:  "ingress 使用中遇到的问题（持续更新）"
date:   2020-3-30 20:36:04 +0800
categories: sre k8s
---

    持续记录与 ingress 相关的问题。

### 如何设置文件上传大小限制？

* kong deployment

```yaml
# 此值为 http 级别配置，由于已存在默认配置不能覆盖，修改后会报错
# Kong 中已默认 client_max_body_size 0;
        - name: KONG_NGINX_HTTP_CLIENT_MAX_BODY_SIZE
           value: 60m

# 此值为 server 级别
        - name: KONG_NGINX_PROXY_CLIENT_MAX_BODY_SIZE
           value: 60m
```

```bash
# 查看配置
# grep 'client_max_body_size' /usr/local/kong/nginx-kong.conf
client_max_body_size 0;
    client_max_body_size 60m;
    client_max_body_size 10m;
```

* ingress 规则文件

```yaml
apiVersion: extensions/v1beta1
 kind: Ingress
 metadata:
   annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: 60m
```

---

# 参考

* [https://docs.konghq.com/1.3.x/configuration/](https://docs.konghq.com/1.3.x/configuration/)
* [http://nginx.org/en/docs/http/ngx_http_core_module.html#client_max_body_size](http://nginx.org/en/docs/http/ngx_http_core_module.html#client_max_body_size)
* [https://www.nginx.com/resources/wiki/start/topics/examples/full/](https://www.nginx.com/resources/wiki/start/topics/examples/full/)
