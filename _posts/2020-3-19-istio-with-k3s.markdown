---
layout: post
title:  "在 k3s 中使用 istio"
date:   2020-3-19 15:32:40 +0800
categories: sre k8s
---

    快速配置一个本地 Istio 环境。

# 背景信息

* 操作系统
```bash
$ uname -a
Linux ubuntu 5.3.0-42-generic #34~18.04.1-Ubuntu SMP Fri Feb 28 13:42:26 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux
```

* 安装 oh-my-zsh，请参考 [Linux 使用笔记]({% post_url 2019-11-04-linux-use %})

# 安装 k3d
```bash
$ curl -s https://raw.githubusercontent.com/rancher/k3d/master/install.sh | bash

# 安装 k3s
$ k3d create --publish 8080:80 --server-arg --no-deploy --server-arg traefik
$ export KUBECONFIG="$(k3d get-kubeconfig --name='k3s-default')"

```

# 安装 kubectl
```bash
$ snap install kubectl --classic
```

# 安装 istio
```bash
$ wget https://github.com/istio/istio/releases/download/1.5.0/istio-1.5.0-linux.tar.gz
$ tar xf istio-1.5.0-linux.tar.gz
$ cd istio-1.5.0
$ cp bin/istioctl /usr/local/bin

# 验证
$ istioctl verify-install

Checking the cluster to make sure it is ready for Istio installation...

#1. Kubernetes-api
-----------------------
Can initialize the Kubernetes client.
Can query the Kubernetes API Server.

#2. Kubernetes-version
-----------------------
Istio is compatible with Kubernetes: v1.17.2+k3s1.

#3. Istio-existence
-----------------------
Istio will be installed in the istio-system namespace.

#4. Kubernetes-setup
-----------------------
Can create necessary Kubernetes configurations: Namespace,ClusterRole,ClusterRoleBinding,CustomResourceDefinition,Role,ServiceAccount,Service,Deployments,ConfigMap. 

#5. SideCar-Injector
-----------------------
This Kubernetes cluster supports automatic sidecar injection. To enable automatic sidecar injection see https://istio.io/docs/setup/kubernetes/additional-setup/sidecar-injection/#deploying-an-app

-----------------------
Install Pre-Check passed! The cluster is ready for Istio installation.

# 安装
$ kubectl apply -f install/kubernetes/istio-demo.yaml

# 查看
$ kgpa   
NAMESPACE      NAME                                      READY   STATUS      RESTARTS   AGE
kube-system    metrics-server-6d684c7b5-gxznr            1/1     Running     0          33m
kube-system    coredns-d798c9dd-lcrwx                    1/1     Running     0          33m
kube-system    local-path-provisioner-58fb86bdfd-2n7p9   1/1     Running     0          33m
istio-system   istio-tracing-797d4c8d48-bd24s            1/1     Running     0          19m
istio-system   kiali-74fdc898b9-kq8gb                    1/1     Running     0          19m
istio-system   svclb-istio-ingressgateway-4cccn          9/9     Running     0          19m
istio-system   istio-citadel-7b95d6cdf7-l5wdc            1/1     Running     0          19m
istio-system   istio-grafana-post-install-1.5.0-hql84    0/1     Completed   0          19m
istio-system   grafana-7797c87688-d5rzf                  1/1     Running     0          19m
istio-system   prometheus-c8fdbd64f-rk7zc                1/1     Running     0          19m
istio-system   istio-sidecar-injector-7db6f668b4-vs4fn   1/1     Running     0          19m
istio-system   istio-galley-78448d877b-4827k             1/1     Running     0          19m
istio-system   istio-telemetry-b588f778d-922p9           2/2     Running     0          19m
istio-system   istio-policy-64dcf9d8f-hh8jf              2/2     Running     0          19m
istio-system   istio-pilot-65b77f4fb7-xpgt8              2/2     Running     1          19m
istio-system   istio-egressgateway-554448866c-lsqzl      1/1     Running     0          19m
istio-system   istio-ingressgateway-7fc66f49dd-hc4sg     1/1     Running     0          19m

```

# 部署 bookinfo

```bash
# 开启注入
kubectl label namespace default istio-injection=enabled

# 部署 bookinfo
$ kaf samples/bookinfo/platform/kube/bookinfo.yaml                            
service/details created
serviceaccount/bookinfo-details created
deployment.apps/details-v1 created
service/ratings created
serviceaccount/bookinfo-ratings created
deployment.apps/ratings-v1 created
service/reviews created
serviceaccount/bookinfo-reviews created
deployment.apps/reviews-v1 created
deployment.apps/reviews-v2 created
deployment.apps/reviews-v3 created
service/productpage created
serviceaccount/bookinfo-productpage created
deployment.apps/productpage-v1 created

$ kga 
NAME                                  READY   STATUS    RESTARTS   AGE
pod/details-v1-78d78fbddf-b822k       2/2     Running   0          5m50s
pod/productpage-v1-85b9bf9cd7-b6qdc   2/2     Running   0          5m49s
pod/reviews-v1-564b97f875-jf98g       2/2     Running   0          5m49s
pod/reviews-v3-67b4988599-vvvx9       2/2     Running   0          5m49s
pod/reviews-v2-568c7c9d8f-hnk4x       2/2     Running   0          5m49s
pod/ratings-v1-6c9dbf6b45-nb49n       2/2     Running   0          5m49s

NAME                  TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/kubernetes    ClusterIP   10.43.0.1       <none>        443/TCP    79m
service/details       ClusterIP   10.43.207.255   <none>        9080/TCP   5m50s
service/ratings       ClusterIP   10.43.228.24    <none>        9080/TCP   5m50s
service/reviews       ClusterIP   10.43.207.232   <none>        9080/TCP   5m50s
service/productpage   ClusterIP   10.43.109.144   <none>        9080/TCP   5m49s

NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/details-v1       1/1     1            1           5m50s
deployment.apps/productpage-v1   1/1     1            1           5m49s
deployment.apps/reviews-v1       1/1     1            1           5m49s
deployment.apps/reviews-v3       1/1     1            1           5m49s
deployment.apps/reviews-v2       1/1     1            1           5m49s
deployment.apps/ratings-v1       1/1     1            1           5m50s

NAME                                        DESIRED   CURRENT   READY   AGE
replicaset.apps/details-v1-78d78fbddf       1         1         1       5m50s
replicaset.apps/productpage-v1-85b9bf9cd7   1         1         1       5m49s
replicaset.apps/reviews-v1-564b97f875       1         1         1       5m49s
replicaset.apps/reviews-v3-67b4988599       1         1         1       5m49s
replicaset.apps/reviews-v2-568c7c9d8f       1         1         1       5m49s
replicaset.apps/ratings-v1-6c9dbf6b45       1         1         1       5m50s

$ kaf samples/bookinfo/networking/bookinfo-gateway.yaml
```

# 验证

* 获取 Ingress 地址

```bash
$ kgs -n istio-system istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
172.19.0.2
```

访问 [http://localhost:8080/productpage](http://localhost:8080/productpage) 或者 [http://172.19.0.2/productpage](http://172.19.0.2/productpage)

# 访问 kiali
$ istioctl dashboard kiali                                                                         
http://localhost:20001/kiali


---

# 参考

* [https://dev.to/bufferings/tried-k8s-istio-in-my-local-machine-with-k3d-52gg](https://dev.to/bufferings/tried-k8s-istio-in-my-local-machine-with-k3d-52gg)
* [https://github.com/rancher/k3d/issues/104](https://github.com/rancher/k3d/issues/104)
* [https://github.com/rancher/k3d](https://github.com/rancher/k3d)
* [https://istio.io/docs/examples/bookinfo/](https://istio.io/docs/examples/bookinfo/)