---
layout:     post
title:      "不翻墙安装kubernetes教程"
subtitle:   " \"Install kubernetes by Kubeadm\""
date:       2019-06-11 20:00:00
author:     "会飞的蜗牛"
header-img: "img/install-kubernetes-by-kubeadm.jpg"
tags:
    - Kubernetes
    
---

# 环境
- Ubuntu16.04
- Virtrual box
- All in one
- 无需翻墙
- 1.14.0

我用的机器是virtualbox，2核4G内存，ubuntu16.04桌面版。`sudo su`切换到root权限

## 更新apt源
```
$ cat /etc/apt/sources.list
deb http://mirrors.163.com/ubuntu/ xenial main
deb-src http://mirrors.163.com/ubuntu/ xenial main

deb http://mirrors.163.com/ubuntu/ xenial-updates main
deb-src http://mirrors.163.com/ubuntu/ xenial-updates main

deb http://mirrors.163.com/ubuntu/ xenial universe
deb-src http://mirrors.163.com/ubuntu/ xenial universe
deb http://mirrors.163.com/ubuntu/ xenial-updates universe
deb-src http://mirrors.163.com/ubuntu/ xenial-updates universe

deb http://mirrors.163.com/ubuntu/ xenial-security main
deb-src http://mirrors.163.com/ubuntu/ xenial-security main
deb http://mirrors.163.com/ubuntu/ xenial-security universe
deb-src http://mirrors.163.com/ubuntu/ xenial-security universe
```

## 安装docker
```
apt-get update 
apt install docker.io -y

```
安装完后是18.09版本。

## 修改kubernetes aliyun源
```
apt-get update && apt-get install -y apt-transport-https
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add - 
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
```

## 安装kubeadm/kubelet/kubectl
```
apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
```
如果是多节点，非master节点安装kubeadm/kubectl是可选选项，装了更好。

## 关闭swap
```
swapoff -a
```
把/etc/fstab包含swap那行记录删掉。

## 配置docker mirror
创建/etc/docker/daemon.json.里面内容如下：

```
{
"registry-mirrors": ["https://registry.docker-cn.com"]
}
```
重新加载docker配置
```
systemctl daemon-reload && systemctl reload docker
```

## 提前拉取k8s镜像
```
$ kubeadm config images list --kubernetes-version v1.14.0
root@dev:/home/vagrant# kubeadm config images list --kubernetes-version v1.14.0
k8s.gcr.io/kube-apiserver:v1.14.0
k8s.gcr.io/kube-controller-manager:v1.14.0
k8s.gcr.io/kube-scheduler:v1.14.0
k8s.gcr.io/kube-proxy:v1.14.0
k8s.gcr.io/pause:3.1
k8s.gcr.io/etcd:3.3.10
k8s.gcr.io/coredns:1.3.1
```

拉取镜像脚本

```
root@dev:/home/vagrant# cat pull_image.sh
#!/bin/bash
images=(
    kube-apiserver:v1.14.0
    kube-controller-manager:v1.14.0
    kube-scheduler:v1.14.0
    kube-proxy:v1.14.0
    pause:3.1
    etcd:3.3.10
    coredns:1.3.1
)

for i in ${images[@]} ; do
    docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/$i
    docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/$i k8s.gcr.io/$i
done
```

## 安装master节点
```
kubeadm init --kubernetes-version=v1.14.0 --apiserver-advertise-address=<your ip> --pod-network-cidr=192.168.0.0/16
```
本文使用weave Network。可以简化为如下，如果是flannel或者calico网络，则必须设置pod-network-cidr字段。

```
kubeadm init --kubernetes-version=v1.14.0
```


## 部署weave网络
```
sysctl net.bridge.bridge-nf-call-iptables=1 -w
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

## 添加worker节点
本文只有一个节点，故把master节点的taint去掉即可，允许调度pod。

```
kubectl taint nodes --all node-role.kubernetes.io/master-
```

如果你有多个节点的话，不需要去掉master的taint。其他节点参照上面的准备阶段在各个节点上做好准备工作以后，只要再Join一下就行了。Join命令在kubeadm init的输出信息里有。

```
kubeadm join --token={token} {master ip}

```

## 总结

以上，一个kubernetes集群就安装好了。

```
root@dev:/home/vagrant# kubectl get nodes
NAME   STATUS   ROLES    AGE   VERSION
dev    Ready    master   33h   v1.14.3
```



