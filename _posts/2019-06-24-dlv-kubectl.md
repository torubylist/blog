---
layout:     post
title:      "dlv and kubectl"
subtitle:   " \"dlv and kubectl\""
date:       2019-06-24 09:00:00
author:     "会飞的蜗牛"
header-img: "img/csi-spec.jpg"
tags:
    - dlv
    - Kubernetes
    - kubectl
    
---


# dlv版本问题
在升级GO版本到1.12.4后发现Goland的Debug报错，如下：could not launch process: decoding dwarf section info at offset 0x0: too short。

原因：

应该是Goland的dlv不是新版本导致不能debug。

解决：

1、更新dlv，`go get -u github.com/derekparker/delve/cmd/dlv`

2、修改Goland的配置，Help->Edit Custom Properties中增加新版dlv的路径配置：`dlv.path=/path/go/bin/dlv`

# kubectl 

```
kubectl -v 4 get nodes --kubeconfig=/Users/zhaoyongping/workspace/vagrant/data/config
I0624 08:59:54.677740   13979 cached_discovery.go:121] skipped caching discovery info due to Get https://192.168.33.10:6443/api?timeout=32s: Forbidden
```
最初以为是kubectl的问题。久思不得法，然同事指点，发现是代理问题。

```
curl https://192.168.33.10:6443/api
curl: (56) Received HTTP code 403 from proxy after CONNECT
```

然后把代理关闭,顺利连通远端kubernetes 服务。

```
http_proxy="" https_proxy="" kubectl -v 4 get nodes --kubeconfig=/Users/zhaoyongping/workspace/vagrant/data/config
I0624 09:03:38.410231   14033 get.go:570] no kind is registered for the type v1beta1.Table in scheme "k8s.io/kubernetes/pkg/api/legacyscheme/scheme.go:29"
NAME   STATUS   ROLES    AGE   VERSION
dev    Ready    master   11h   v1.14.3
```