---
layout:     post
title:      "如何写一个kubectl plugin"
subtitle:   " \"How To Write a Kubectl Plugin\""
date:       2019-06-30 20:00:00
author:     "会飞的蜗牛"
header-img: "img/kubectl-plugin.jpg"
tags:
    - Kubernetes
    - Kubectl-Plugin
    
---

## 环境
1. ubuntu16.04
2. kubectl 1.14
3. sudo su


## 什么是Kubectl Plugin
我们都知道`kubectl`是`kubenetes`的命令行客户端，通过`kubectl`可以创建删除修改k8s资源，而plugin是以一种插件的形式扩展`kubectl`原生功能。比如`kubectl`并不支持`kubectl ns [option]`命令，但是我可以安装一个`kubectl-ns`插件扩展它，将`kubectl -ns`放到`PATH`路径下从而可以通过`kubectl ns test`进入`test`命名空间。`kubectl plugin`就是一个以`kubectl-plugin`开头的可执行文件。可以执行你想要的子命令并获取相应的结果。`kubectl plugin`从1.8.0alpha版本release，而在1.12版本做了较大的改动，主要的变化在加载方式等。

## 如何实现一个简易的Kubectl Plugin
实现一个plugin很简单，任何计算机语言皆可，我们通过shell脚本来实现一个简易的plugin。然后`mv kubectl-test /usr/local/bin/`。`export PATH=PATH:/usr/local/bin`。那么现在一个plugin就实现了。

```
$ cat kubectl-test
#!/bin/bash

if [[ $1 == "config" ]];then
    echo "yes, it's config"
    exit 0
fi

if [[ $1 == "version" ]];then
    echo "version is 1.0.0"
    exit 0
fi

echo "i'm just a plugin"

$ chmod +x ./kubectl-test
$ mv kubectl-test /usrl/local/bin/
$ kubectl test config
yes, it's config
$ kubectl test help
i'm just a plugin 

```

### 发现插件
kubectl提供了`kubectl plugin list`来通过PATH来查找命令。通过遍历`$PATH`路径来匹配插件命令，按最长命令长度比配，并以PATH的先后顺序依次排列，先到先到，如果不同的文件夹下有相同的插件名字，则后被查到的命令没有机会被执行。同时，kubectl 本身支持的命令有最高的优先级，不会遵循plugin插件的查找机制。比如`kubectl create foo`永远是执行原生的`kubectl create `命令，而不会是你自定义的`kubectl-create-foo`。

### 插件变量
1.12以后， `kubectl pluginname` 只能提供标准标量，其他变量不在支持，比如其他的私有化变量(比如 KUBECTL_PLUGINS_CURRENT_NAMESPACE)不会再提供。如果该option不存在，就返回默认值。由上文可以看出，命令`kubectl test config`的 `$0`永远是插件自己的名字，`$1`开始才是options。

### 命名及查找机制

1. PATH 优先匹配原则
2. 短横线及下划线
3. 以最精确匹配为首要目标
4. 查找失败自动转换为参数

#### PATH优先匹配
```
root@dev:/vagrant_data# echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games
root@dev:/vagrant_data# cat /usr/local/bin/kubectl-test
#!/bin/bash

if [[ $1 == "config" ]];then
    echo "yes, it's config"
    exit 0
fi

echo "i'm just a plugin in /usr/local/bin/"

root@dev:/vagrant_data# cat /usr/local/games/kubectl-test
#!/bin/bash

if [[ $1 == "config" ]];then
    echo "yes, it's config"
    exit 0
fi

echo "i'm a plugin in /usr/local/games"

root@dev:/vagrant_data# kubectl test help
i'm just a plugin in /usr/local/bin/

```

#### 短横线及下划线

```
root@dev:/vagrant_data# cat /usr/local/bin/kubectl-test-aaa
#!/bin/bash

echo "test kubectl-test-aaa"
root@dev:/vagrant_data# kubectl test aaa
test kubectl-test-aaa

```
如果我们的aaa是一个参数，那么只能在kubectl-test-aaa不存在的情况下才会去查找`kubectl-test aaa`，例如
```
oot@dev:/vagrant_data# kubectl test aaa
test kubectl-test-aaa
oot@dev:/vagrant_data# rm /usr/local/bin/kubectl-test-aaa
root@dev:/vagrant_data# kubectl test aaa
i'm just a plugin in /usr/local/bin/  # 运行的是kubectl-test
```

如果我们想要运行`kubectl test-aaa`,则需要将插件命名为`kubectl-test_aaa`,例如：

```
root@dev:/vagrant_data# cat /usr/local/bin/kubectl-test_aaa
#!/bin/bash

echo "test kubectl-test_aaa"
root@dev:/vagrant_data# kubectl test-aaa
test kubectl-test_aaa
```

#### 精确查找,查找失败自动转换。
上文已经提到就是总是优先查找到匹配的命令，然后再依次以短划线为分隔符回退。例如`kubectl test aaa bbb`总是先找`kubectl-test-aaa-bbb`，如果找不到，才会查找`kubectl-test-aaa`, bbb作为参数。如果再找不到，则会`kubectl-test`, `aaa bbb` 作为参数$1, $2。

## Golang辅助库及sample文件
kubernetes 提供了一个 [command line runtime package](https://github.com/kubernetes/cli-runtime)，使用 Go 编写插件，配合这个库可以更加方便的解析和调整 kubectl 的配置信息

官方为了演示如何使用这个 [cli-runtime](https://github.com/kubernetes/cli-runtime) 库编写了一个 namespace 切换的插件，仓库地址[Github](https://github.com/kubernetes/sample-cli-plugin) ，基本编译使用如下(直接 go get 后编译文件默认为目录名 cmd)。如果你需要写一个plugin可以参考这个实现。

```
➜  ~ go get k8s.io/sample-cli-plugin/cmd
➜  ~ sudo mv gopath/bin/cmd /usr/local/bin/kubectl-ns
➜  ~ kubectl ns
default
➜  ~ kubectl ns --help
View or set the current namespace

Usage:
  ns [new-namespace] [flags]

Examples:

        # view the current namespace in your KUBECONFIG
        kubectl ns

        # view all of the namespaces in use by contexts in your KUBECONFIG
        kubectl ns --list

        # switch your current-context to one that contains the desired namespace
        kubectl ns foo


Flags:
      --as string                      Username to impersonate for the operation
      --as-group stringArray           Group to impersonate for the operation, this flag can be repeated to specify multiple groups.
      --cache-dir string               Default HTTP cache directory (default "/Users/mritd/.kube/http-cache")
      --certificate-authority string   Path to a cert file for the certificate authority
      --client-certificate string      Path to a client certificate file for TLS
      --client-key string              Path to a client key file for TLS
      --cluster string                 The name of the kubeconfig cluster to use
      --context string                 The name of the kubeconfig context to use
  -h, --help                           help for ns
      --insecure-skip-tls-verify       If true, the server's certificate will not be checked for validity. This will make your HTTPS connections insecure
      --kubeconfig string              Path to the kubeconfig file to use for CLI requests.
      --list                           if true, print the list of all namespaces in the current KUBECONFIG
  -n, --namespace string               If present, the namespace scope for this CLI request
      --request-timeout string         The length of time to wait before giving up on a single server request. Non-zero values should contain a corresponding time unit (e.g. 1s, 2m, 3h). A value of zero means don't timeout requests. (default "0")
  -s, --server string                  The address and port of the Kubernetes API server
      --token string                   Bearer token for authentication to the API server
      --user string                    The name of the kubeconfig user to use

```


# 参考
<https://kubernetes.io/docs/tasks/extend-kubectl/kubectl-plugins/>
<https://mritd.me/2018/11/30/kubectl-plugin-new-solution-on-kubernetes-1.12/>
<https://github.com/kubernetes/sample-cli-plugin>