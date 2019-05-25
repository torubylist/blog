---
layout:     post
title:      "kubectl run背后的征途"
subtitle:   " \"what-happens-when-k8s-edition...\""
date:       2019-05-25 20:00:00
author:     "会飞的蜗牛"
header-img: "img/what-happens-when-k8s-edition.jpg"
tags:
    - Kubernetes

---

> 翻译主要是为了更好的掌握文中的内容，如有不妥之处或者想我交流，欢迎随时给我发邮件，谢谢。yongpingzhao1#gmail.com

假如我想部署一个nginx服务到kubernetes集群上，我很可能会在终端上输入：

```
kubectl run nginx --image=nginx --replicas=3
```
然后点击enter。几秒钟之后，我应该可以看到三个pods分别运行在各自的工作节点上。这看起来很魔幻，简直让人不敢让人相信。但是那这背后到底发生了什么？

Kubernetes最令人惊叹的事情之一，就是它通过对用户友好的API处理跨基础架构的部署工作。简单的抽象隐藏了复杂性。但是为了充分理解它为我们提供的价值，理解它的内部结构也是有用的。 本指南将引导您完成从客户端到kubelet的请求的整个生命周期，在必要时链接到源代码以说明正在发生的事情。

这是一份持续活跃的文档。如果您发现可以改进或重写的区域，欢迎贡献你的patch！


# kubectl
## 客户端验证和生成器
好了，让我们开始吧。我们刚刚在终端里敲击了enter。接下来会发生什么？

`kubectl`将要做的第一件事是执行一些客户端验证。这可确保那些始终会失败的请求（例如，创建不受支持的资源或使用[格式错误的镜像名称](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/kubectl/cmd/run/run.go#L264)）将快速失败，并且这个请求不会发送到kube-apiserver，通过这种方式来减少不必要的负载从而改善系统性能。

验证后，`kubectl`开始汇总发送给kube-apiserver的HTTP请求。在Kubernetes系统中所有的访问或更改状态的尝试都将通过API服务器进行，​​后者又与etcd进行通信。kubectl客户端也不例外，`kubectl`使用[生成器](https://kubernetes.io/docs/user-guide/kubectl-conventions/#generators)来构造HTTP请求，生成器是一个负责序列化的抽象概念。

不太明显的是，你实际上可以使用 `kubectl run` 指定多个资源类型，而不仅仅是Deployments。为了实现这一目的，如果未使用`--generator`标志显式指定生成器名称，`kubectl`将[自动](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/kubectl/cmd/run/run.go#L319-L339)推断资源类型。

例如，具有`--restart-policy=Always`的资源被视为Deployments，而具有--`restart-policy=Never`的资源被视为Pods。`kubectl`还将确定是否需要触发其他操作，例如记录命令（用于部署或审计），或者是否只是试运行（由`--dry-run`标志指示）。

在意识到我们要创建 Deployment 之后，它将使用 DeploymentAppsV1 生成器从我们提供的参数生成[运行时](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/kubectl/generate/versioned/run.go#L237)对象。 “运行时对象”是资源的通用术语。

## API组和版本协商
在我们继续旅途之前，值得指出的是Kubernetes使用的是一个隶属于某个“API组”的版本化API。API 组旨在对类似资源进行分类，以便更容易推理出正确的处理器。它还为单个单片 API 提供了更好的替代方案。 Deployment 的 API 组名为 apps ，其最新版本为 v1 。这就是你需要在 Deployment 清单顶部输入 apiVersion: apps/v1 的原因。

无论怎样...在`kubectl`生成运行时对象之后，它开始为这个对象找到适当的[API组和API版本](https://github.com/kubernetes/kubernetes/blob/master/pkg/kubectl/cmd/run.go#L580-L597)，然后组装成一个符合该资源各种REST语义的[版本化客户端](https://github.com/kubernetes/kubernetes/blob/master/pkg/kubectl/cmd/run.go#L598)。这个发现阶段称为版本协商，`kubectl`会扫描远程 API 上的 /apis 路径以检索所有可能的 API 组。由于 kube-apiserver 在此路径中公开了 /apis 的架构文档（采用OpenAPI格式），因此客户端可以轻松找到正确的结果。

为了提高查找性能，`kubectl`还将[OpenAPI架构模式](https://github.com/kubernetes/kubernetes/blob/7650665059e65b4b22375d1e28da5306536a12fb/pkg/kubectl/cmd/util/factory_client_access.go#L117)缓存到了 `～/.kube/cache/discovery`目录。如果要查看此 API 发现的具体操作，可以尝试删除该目录并将`-v`标志设为最大值。您将看到试图查找所有这些API版本的HTTP请求。请求会非常多！

最后一步是实际发送HTTP请求。一旦它这样做并获得成功的响应，`kubectl`将根据所需的输出格式打印出成功消息。

## 客户端身份验证
我们在上一步中没有提到的一点是客户端身份验证（这是在发送HTTP请求之前处理的），所以现在让我们来看看。

为了成功发送请求，kubectl需要能够进行身份验证。用户凭据几乎总是存储在磁盘上的kubeconfig文件中，但该文件可以存储在不同的位置。为了找到它，kubectl执行以下操作：

- 如果提供了`--kubeconfig`标志，那就可以使用这个标志。
- 如果设置了`$KUBECONFIG`环境变量，那也可以使用。
- 不然就查找预计可能的用户根目录`~/.kube`，例如`$HOME/.kube/config`，使用第一个找到的文件。

解析完文件后，它会确定当前要使用的上下文，要指向的当前集群以及与当前用户关联的所有身份验证信息。如果用户在命令行中提供了特定标志的值（例如--username），则优先使用这些值，并将覆盖kubeconfig中指定的值。 一旦有了这些信息，`kubectl`就会填充客户端的配置，以便它能够适当地修饰HTTP请求：

- x509证书使用[tls.TLSConfig](https://github.com/kubernetes/client-go/blob/82aa063804cf055e16e8911250f888bc216e8b61/rest/transport.go#L80-L89).（也包括CA）
- bearer tokens在"Authorization" HTTP请求头中[发送](https://github.com/kubernetes/client-go/blob/c6f8cf2c47d21d55fa0df928291b2580544886c8/transport/round_trippers.go#L314)。
- 用户名和密码可以通过HTTP基本验证[发送](https://github.com/kubernetes/client-go/blob/c6f8cf2c47d21d55fa0df928291b2580544886c8/transport/round_trippers.go#L223)。
- OpenID身份验证可以由用户事先处理，产生一个像bearer token一样被发送的token。


# kube-apiserver

## Authentication（认证）
所以我们的请求已经发送，万岁！那接下来是什么？走进我们的视野的是kube-apiserver。正如我们前文所述，kube-apiserver是客户端和系统组件用来持久化和检索集群状态的主要接口。要执行其功能，首先它需要能够验证请求者的身份。此过程称为身份验证。

那apiserver如何验证请求者的身份？当服务器首次启动时，它会查看用户提供的所有[CLI](https://kubernetes.io/docs/admin/kube-apiserver/)标志，并组装合适的验证器列表。我们举个例子：如果传入了`--client-ca-file`，它会附加 x509 身份验证器. 如果设置了`--token-auth-file`，它会将令牌验证器附加到列表中。每次收到请求时，都会运行[身份验证器链](https://github.com/kubernetes/apiserver/blob/51bebaffa01be9dc28195140da276c2f39a10cd4/pkg/authentication/request/union/union.go#L54)，直到成功为止：

- [x509处理程序](https://github.com/kubernetes/apiserver/blob/51bebaffa01be9dc28195140da276c2f39a10cd4/pkg/authentication/request/x509/x509.go#L60)将验证HTTP请求是否使用由 CA 根证书签名的TLS密钥进行编码
- [bearer token处理器](https://github.com/kubernetes/apiserver/blob/51bebaffa01be9dc28195140da276c2f39a10cd4/pkg/authentication/request/bearertoken/bearertoken.go#L38)将验证 `--token-auth-file`提供的令牌（在HTTP Authorization标头中指定）是否存在于指定的磁盘上的文件中
- [basic auth](https://github.com/kubernetes/apiserver/blob/51bebaffa01be9dc28195140da276c2f39a10cd4/plugin/pkg/authenticator/request/basicauth/basicauth.go#L37)处理器将类似地确保HTTP请求的基本身份验证凭据与其自己的本地状态匹配。

如果所有的[身份验证器](https://github.com/kubernetes/apiserver/blob/20bfbdf738a0643fe77ffd527b88034dcde1b8e3/pkg/authentication/request/union/union.go#L71)都失败了，那么请求就失败了，就会返回聚合后的错误。如果身份验证成功，则会从请求中删除Authorization标头，并将[用户信息](https://github.com/kubernetes/apiserver/blob/e30df5e70ef9127ea69d607207c894251025e55b/pkg/endpoints/filters/authentication.go#L71-L75)添加到它的上下文中。这为将来的步骤（例如授权和准入控制器）提供了访问用户身份的能力。

## Authorization（授权）
好的，请求已经发送，kube-apiserver已成功验证我们是谁。终于松口气了！但是，我们的工作还没有完成。我们的身份没有问题，但我们是否有权执行此操作？毕竟，身份和权限许可并不是一回事。为了让我们的”征途“能够继续，kube-apiserver需要给我们授权。

kube-apiserver处理授权的方式与身份验证非常相似：基于标志输入，它将汇集一系列授权者，这些授权者将针对每个传入的请求进行判断。如果所有授权者都拒绝该请求，则该请求将被Forbidden并且[不能再继续进行下去](https://github.com/kubernetes/apiserver/blob/e30df5e70ef9127ea69d607207c894251025e55b/pkg/endpoints/filters/authorization.go#L60)。如果某个授权人批准，则请求继续。

kubernetes v1.8自带的授权者的一些案例是：

- [Webhook](https://github.com/kubernetes/apiserver/blob/d299c880c4e33854f8c45bdd7ab599fb54cbe575/plugin/pkg/authorizer/webhook/webhook.go#L143)，与集群外HTTP(S)服务交互;
- [ABAC](https://github.com/kubernetes/kubernetes/blob/77b83e446b4e655a71c315ad3f3890dc2a220ccf/pkg/auth/authorizer/abac/abac.go#L223)，它执行静态文件中定义的策略;
- [RBAC](https://github.com/kubernetes/kubernetes/blob/8db5ca1fbb280035b126faf0cd7f0420cec5b2b6/plugin/pkg/auth/authorizer/rbac/rbac.go#L43)，它强制执行RBAC角色，由管理员添加为k8s资源
- [Node](https://github.com/kubernetes/kubernetes/blob/dd9981d038012c120525c9e6df98b3beb3ef19e1/plugin/pkg/auth/authorizer/node/node_authorizer.go#L67)，确保节点客户端（即kubelet）只能访问自身托管的资源。

可以仔细查看每个授权方法，看看它们是如何工作的！


## Admission control（准入控制）
好了，到目前为止，我们已经过认证并获得了kube-apiserver的授权。那还剩下什么呢？从kube-apiserver的角度来看，它相信我们是谁并允许我们的请求继续，但是对于Kubernetes系统的其他部分对请求的内容应不应该被许可的有强烈的意见。这时[准入控制器](https://kubernetes.io/docs/admin/admission-controllers/#what-are-they)就进入了版图的中心。

授权的重点是回答用户是否具有权限，但admission control会拦截该请求，以确保其符合群集更广泛的期望和规则。它们是对象持久化到etcd之前的最后一个控制堡垒，因此它们封装了剩余的系统检查以确保操作不会产生意外或负面的结果。不同于认证和授权分别只关心请求的用户和操作，准入控制还处理请求的内容，并且仅对创建、更新、删除或连接（如代理）等有效，而对读操作无效。

注意：准入控制器的工作方式跟认证者和授权者的工作方式非常类似，但有一个区别：与认证者和授权者链不同，如果单个准入控制器失败，则整个链断开，请求将失败。

准入控制器设计的真正优势在于它致力于提升可扩展性。每个控制器都作为插件存储在[plugin/pkg/admission](https://github.com/kubernetes/kubernetes/tree/master/plugin/pkg/admission)目录中，并且与某一个小接口相匹配。然后每个控制器都将编译进kubernetes二进制文件中。

准入控制器通常分为资源管理，安全，默认控制器和参照一致性。以下是一些只关注资源管理的准入控制器示例：

- InitialResources 根据过去的使用情况为容器的资源设置默认资源限制;
- LimitRanger 设置容器资源请求和限制的默认值，或强制某些资源的上限（不超过2GB的内存，默认为512MB）;
- ResourceQuota 计算并拒绝命名空间中的多个对象（pod，rc，service loadbalancer）或总的资源（cpu，内存，磁盘）消耗。


# etcd
到目前为止，Kubernetes已经完全审查了传入的请求，并允许它继续前进。在下一步中，kube-apiserver将反序列化HTTP请求，从请求中构造运行时对象（有点像kubectl生成器的逆过程），并将它们持久化到数据存储区。让我们稍微分析一下。

kube-apiserver如何知道在接受我们的请求时该怎么做？ 好吧，在任何请求被服务之前，会发生一系列非常复杂的步骤。让我们从第一次运行二进制文件开始说起：

1. 运行kube-apiserver二进制文件时，它会创建一个[服务链](https://github.com/kubernetes/kubernetes/blob/master/cmd/kube-apiserver/app/server.go#L119)，允许使用聚合apiserver。这基本上就是支持多个apiservers的一种方式（我们无需担心这个）。
2. 同时，会创建一个[通用的apiserver](https://github.com/kubernetes/kubernetes/blob/master/cmd/kube-apiserver/app/server.go#L149)提供默认服务。
3. 然后，生成了OpenAPI规范的[配置](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/server/config.go#L149)。
4. 然后kube-apiserver遍历模式中指定的所有API组，并将每一个 API 组配置一个通用存储抽象的[storage provider](https://github.com/kubernetes/kubernetes/blob/c7a1a061c3dc5acabcc0c35b3b96a6935dccf546/pkg/master/master.go#L410)，并保存到 etcd 中。这就是kube-apiserver在访问或改变资源状态时，就会调用这些api组，跟etcd交互。
5. 对于每个API组，它还会有不同的迭代版本，并存储了每个HTTP路由的[REST映射](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/groupversion.go#L92)。 这允许kube-apiserver一旦找到匹配的路由就映射请求或委托给正确的逻辑链路。
6. 对于我们的某个特定案例，就需要注册一个[POST处理器](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/installer.go#L710)，该处理器将它委托给[创建资源处理器](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/handlers/create.go#L37)。

现在，kube-apiserver 已经知道了所有的路由及其对应的 REST 路径，在请求时就知道调用哪些处理器和存储提供器。多么机智的设计，现在假设客户端的 HTTP 请求已经被 kube-apiserver 收到了: 

1. 如果处理器链路可以将请求与一个设定的模式（例如我们注册的路由）匹配，它将为这个请求调用路由注册的[专用处理程序](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/server/handler.go#L143)。否则它会回退到基于[路径的处理程序](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/server/mux/pathrecorder.go#L248)（这是调用/apis时会发生的情况）。如果没有为该路径注册处理器，则会调用[未找到处理程序](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/server/mux/pathrecorder.go#L254)，从而生成404。
2. 幸运的是，我们有一个名为[createHandler](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/handlers/create.go#L37)的注册路由！它有什么作用？首先，它会解码HTTP请求并执行基本验证，例如确保它们提供的JSON与我们的版本化API资源一致。
3. 然后[进入](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/handlers/create.go#L93-L104)审计和最后的准入认证阶段。
4. 通过使用[storage provider](https://github.com/kubernetes/apiserver/blob/19667a1afc13cc13930c40a20f2c12bbdcaaa246/pkg/registry/generic/registry/store.go#L327)代理的方式将资源将保存到[etcd](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/handlers/create.go#L111)。通常，etcd键是<namespace>/<name>的形式，但这是可以自由配置的。
5. 它将捕获所有的创建时错误，最后，storage provider会执行get调用以确保对象被创建了。然后，如果需要额外finalization，它会调用post-create处理器和装饰器。
6. 这样，HTTP响应就[构造](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/handlers/create.go#L131-L142)好了，并发回给apiserver。

步骤真的非常多！我们按图索骥，终于柳暗花明。这让我们意识到apiserver做了多么了不起的工作。总结一下：现在我们的Deployment资源存放在etcd中。但是这还只是万里长征的第一步...

# 初始化器
将对象持久化到数据存储区后，要直到一系列[初始化程序](https://kubernetes.io/docs/admin/extensible-admission-controllers/#initializers)之后才会被apiserver看到或调度。 初始化程序是与资源类型相关联的控制器，并在资源可被外部使用之前对资源执行一些逻辑。 如果资源类型注册了零初始化器，则跳过此初始化步骤，并使资源立即可见。

正如许多[优秀的博客](https://ahmet.im/blog/initializers/)文章所指出的，这是一个强大的功能，因为它允许我们执行通用的自举操作。一些可能的例子是：

 - 将代理边车容器注入到暴露端口80的Pod中，或者使用特定注释。
 - 将具有测试证书的卷注入特定命名空间中的所有Pod。
 - 如果密钥短于20个字符（例如密码），请阻止其创建。

initializerConfiguration对象允许你声明应为特定资源类型运行哪些初始值设定项。想象一下，我们希望每次创建Pod时都运行自定义初始化程序，我们会这样做：

```
apiVersion: admissionregistration.k8s.io/v1alpha1
kind: InitializerConfiguration
metadata:
  name: custom-pod-initializer
initializers:
  - name: podimage.example.com
    rules:
      - apiGroups:
          - ""
        apiVersions:
          - v1
        resources:
          - pods
```
 
创建此配置后，它会将custom-pod-initializer附加到每个Pod的metadata.initializers.pending字段。初始化程序控制器已经部署，并将定期扫描新的Pod。当initializer程序在Pod的pending字段中检测到了它的名字时，它就会执行初始化逻辑。完成这个过程后，就从pod的pending字段列表中删除它的名字。只有名字在pending列表中第一位的初始化程序才可以操作资源。当所有初始化程序完成且pending字段为空时，就认为该对象已完成初始化。

火眼金睛的你可能会发现一个潜在的问题。如果kube-apiserver无法看到这些资源，那么userland控制器如何处理资源？为了解决这个问题，kube-apiserver公开了一个？includeUninitialized查询参数，该参数返回所有对象，甚至是未初始化的对象。


# Control loops（控制循环）
## Deployments控制器
在此阶段，我们的Deployments记录存在于etcd中，并且所有的初始化逻辑都已完成。接下来的阶段，我们将设置Kubernetes所依赖的资源拓扑。可以想到的是，Deployment实际上只是ReplicaSet的集合，而ReplicaSet是Pod的集合。那么Kubernetes如何从一个HTTP请求创建这个层次结构呢？这正是Kubernetes的内置控制器接管的地盘。

Kubernetes在整个系统中充分利用“控制器”。控制器是一个异步脚本，用于将Kubernetes系统的当前状态与期望状态进行调协。每个控制器都有自己管控的一亩三分地，由kube-controller-manager发起并行运行。让我们来看下接管的第一个Deployments控制器。

将Deployment记录存储到etcd并初始化后，通过kube-apiserver就可以看到它。当这个新的资源可用时，Deployment控制器会去检测到它，它的工作是监听Deployment记录的更改。在我们的例子中，控制器通过一个通知器注册创建事件的[特定回调](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/deployment/deployment_controller.go#L122)（有关这是什么的更多信息，请参见下文）。

当我们的Deployment首次可用时，将执行此处理程序，并将该对象添加到[内部工作队列](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/deployment/deployment_controller.go#L170)。当它处理我们的对象时，控制器将检查我们的[Deployment](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/deployment/deployment_controller.go#L572)，并[意识](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/deployment/deployment_controller.go#L633)到没有与之关联的ReplicaSet或Pod记录。它通过使用标签选择器查询kube-apiserver来检查。有趣的是，这个同步过程是状态不可知的：它以与现有记录相同的方式协调新记录。

在意识到这些资源都不存在之后，它将开始[扩张过程](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/deployment/sync.go#L385)来开启状态解析。通过推出（例如创建）ReplicaSet资源，为其分配标签选择器，并为其提供修订号为1来执行此操作。ReplicaSet的PodSpec从Deployment清单及其相关元数据复制而来。有时，Deployment记录也需要在此之后更新（例如，如果设置了process deadline）。

完成了上述步骤，会[更新Deployment的状态](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/deployment/sync.go#L70)，然后重新进入相​​同的调协循环，等待Deployment达成最终的期望状态。由于Deployment控制器只关心创建ReplicaSet，因此需要由下一个控制器ReplicaSet控制器继续进行此协调阶段。


## ReplicaSets controller（副本控制器）
在上一步中，Deployments控制器创建了Deployment的第一个ReplicaSet，但我们仍然没有Pod。而这正是ReplicaSet控制器发挥作用的地方！它的工作是监视ReplicaSet及其相关资源（Pod）的生命周期。与大多数其他控制器一样，它通过触发某些事件的处理程序来实现。

我们感兴趣的事件是资源创建。创建ReplicaSet时（由Deployment控制器提供），RS控制器会[检查](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/replicaset/replica_set.go#L583)新ReplicaSet的状态，并意识到实际状态与期望状态之间存在偏差。

然后，它试图通过[实时计算](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/replicaset/replica_set.go#L460)属于Replicaset的pod的数量来协调这种状态。它开始以谨慎的方式创建它们，确保ReplicaSet的瞬时计数（它从其父Deployment继承）始终匹配。

Pods的创建操作也是[批处理](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/replicaset/replica_set.go#L487)的，从SlowStartInitialBatchSize开始，以一种“slow start”操作方式，在每次成功之后数量加倍。这旨在当存许多pod启动失败时（例如资源配额限制），可以减少不必要的HTTP请求从而淹没kube-apiserver的风险。如果我们要失败，我们也可以优雅地失败，对其他系统组件的影响减少到最小！

Kubernetes通过`Owner References`（子资源中的一个字段引用其父项的ID）强制执行对象层次结构。一旦控制器管理的资源被删除（级联删除），这不仅可以确保子资源被垃圾收集，还可以为父资源提供一种有效的方式来避免他们竞争同一个子级资源（想象两对父母都认为的他们拥有同一个孩子的情景）。

`Owner Reference`设计另一个微妙优点是它是有状态的：如果任何控制器要重启，那么控制器宕机时间不会造成更大的影响，因为资源拓扑独立于控制器。这种对隔离的关注也渗透到控制器本身的设计中：控制器不应该在它们没有明确拥有的资源上进行操作。相反，控制器应该选择对资源的所有权，互不干涉，互不共享。

不管怎样，还是让我们回到`Owner Reference`！ 有时系统中存在“Ophan”资源，通常在以下情况下发生：

- 父资源被删除但子资源未被删除
- 垃圾收集策略禁止删除子项

发生这种情况时，控制器将确保孤儿新父母接收。 多个父母可以竞争收养`ophan`资源，但只有一个会成功（其他人将收到一个验证错误）。

## Informers(通知器)
您可能已经注意到，某些控制器（如RBAC授权程序或Deployment控制器）需要检索集群状态才能运行。让我们回到RBAC授权器的示例，当请求进入时，验证器将保存用户状态的初始表征供以后使用。然后，RBAC授权器将使用它来检索与etcd中的用户关联的所有角色和角色绑定。控制器如何访问和修改这些资源？事实证明这是一个常见的用例，并在Kubernetes中通过Informer来解决这个问题。

Informer是一种模式，允许控制器订阅存储事件并轻松列出他们感兴趣的资源。Informer除了提供了一个很好的工作抽象，它还需要处理很多细节，如缓存（缓存很重要，因为它减少了不必要的kube-apiserver连接，并减少了服务器端和控制端的重复序列化成本）。通过使用这种设计，它还允许控制器以线程安全的方式进行交互，而不必担心会影响其他控制器。

有关通知器如何与控制器工作的更多信息，请查看此[博客文章](http://borismattijssen.github.io/articles/kubernetes-informers-controllers-reflectors-stores)。

## Scheduler（调度器）
在所有相关的控制器运行之后，我们有了一个Deployment，一个ReplicaSet和三个Pod存储在etcd中，并且对kube-apiserver可见。但是，我们的pod的状态是Pending，因为它们尚未安排到某个具体的Node上。解决此问题的最终控制器是调度程序。

调度器作为控制平面的独立组件运行，并以与其他控制器相同的方式运行：它侦听事件并尝试协调状态。在这种情况下，它会[过滤](https://github.com/kubernetes/kubernetes/blob/master/plugin/pkg/scheduler/factory/factory.go#L190)PodSpec中具有空NodeName字段的pod，并给pod查找合适的Node。

为了找到合适的节点，必须使用具体的调度算法。默认调度算法的工作方式如下：

1. 调度程序启动时，会注册一组[默认的predicates（预选策略）](https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/plugin/pkg/scheduler/algorithmprovider/defaults/defaults.go#L65-L81)。[预选](https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/plugin/pkg/scheduler/core/generic_scheduler.go#L117)根据节点资源计算托管pod与节点的适配性，这些predicates是非常有效的。例如，如果PodSpec显式请求CPU或RAM资源，并且节点由于容量不足而无法满足这些请求，则会排除这些Node（资源容量为总容量减去目前正在运行容器资源请求的总和）。

2. 过滤了不合适的节点之后，就会对剩余的节点运行一系列[priority函数](https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/plugin/pkg/scheduler/core/generic_scheduler.go#L354-L360)，以便对它们的适用性进行优先级排序。例如，为了在整个系统中平衡工作负载，它将选择资源请求最少的节点（因为这表明运行的工作负载较少）。当运行这些函数时，它会为每个节点进行排名。然后选择排名最高的节点进行调度。

一旦算法找到了一个节点，调度程序就会创建一个[Binding对象](https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/plugin/pkg/scheduler/scheduler.go#L336-L342)，其Name和UID与Pod匹配，其ObjectReference字段包含所选节点的名称。然后通过POST请求将其[发送到](https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/plugin/pkg/scheduler/factory/factory.go#L1095)apiserver。

当kube-apiserver收到此Binding对象时，注册表会反序列化该对象并更新Pod对象上的以下字段：它在ObjectReference里将NodeName设置为选中的[Node名字](https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/pkg/registry/core/pod/storage/storage.go#L170)，它添加[所有的相关注释](https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/pkg/registry/core/pod/storage/storage.go#L174-L176)，并将其PodScheduled状态条件[设置](https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/pkg/registry/core/pod/storage/storage.go#L177-L180)为True。

一旦调度程序将Pod调度到节点，该节点上的kubelet就可以接管并开始部署。精彩！

附注：自定义调度程序：有趣的是predicate和priority函数都是可扩展的，可以使用--policy-config-file标志来自定义。这引入了一定程度的灵活性，管理员还可以在独立部署中运行自定义调度程序（具有自定义处理逻辑的控制器）。如果PodSpec包含schedulerName，Kubernetes会将该pod的调度移交给在该schdedulerName下注册的调度程序。

# Kubelet
## Pod同步
好吧，所有的 Controller 都完成了工作！总结一下：

- HTTP请求通过了身份验证，授权和准入控制阶段
- Deployment，ReplicaSet和三个Pod资源被持久化到etcd
- 运行了一系列初始化程序
- 最后，每个Pod被调度到了相应合适的节点

然而，到目前为止，所有的状态变化仅仅只是针对保存在 etcd 中的资源记录。接下来的步骤涉及到在工作节点之间的Pod分配，这才是像Kubernetes这样的分布式系统的关键点！Kubernetes通过一个叫kubelet的组件来完成这个任务。让我们开始吧！

在Kubernetes集群中，kubelet是一个运行在每个节点上的代理组件，负责管理Pods的生命周期。这意味着它处理“Pod”（实际上只是一个Kubernetes概念）与其构建块，容器之间的所有转换逻辑。它还处理有关挂载卷、容器日志记录、垃圾收集以及许多其他重要事项的相关逻辑。

一个有用的方式是把kubelet看成是一个控制器！它每隔20秒（这是可配置的）从kube-apiserver查询Pods，过滤出那些NodeName与运行kubelet的[节点名称相匹配](https://github.com/kubernetes/kubernetes/blob/3b66adb8bc6929e1205bcb2bc32f380c39be8381/pkg/kubelet/config/apiserver.go#L34)的Pods。一旦 kubelet 获取该列表，它就会通过将该列表与其内部缓存进行比较来检测是否有新添加pods，如果存在差异就开始进行同步。我们来看看同步过程是什么样的：

1. 如果正在创建pod（我们的是！），它会在Prometheus中[注册用于跟踪](https://github.com/kubernetes/kubernetes/blob/fc8bfe2d8929e11a898c4557f9323c482b5e8842/pkg/kubelet/kubelet.go#L1519)pod延迟的一些启动指标。
2. 然后[生成](https://github.com/kubernetes/kubernetes/blob/dd9981d038012c120525c9e6df98b3beb3ef19e1/pkg/kubelet/kubelet_pods.go#L1287)一个 PodStatus 对象，该对象表示Pod当前阶段的状态。Pod的状态是 Pod 在其生命周期中的最精简的概要。包括 Pending，Running，Succeeded，Failed 和Unknown。生成这种状态的过程非常复杂，所以让我们深入了解发生的过程：
	
	- 首先，顺序执行一系列PodSyncHandler。每个处理程序检查Pod是否仍应驻留在节点上。如果他们中的任何一个处理器决定Pod不再属于那个节点，Pod的阶段将[变为PodFailed](https://github.com/kubernetes/kubernetes/blob/dd9981d038012c120525c9e6df98b3beb3ef19e1/pkg/kubelet/kubelet_pods.go#L1293-L1297)，并最终从该Node中逐出。这些示例包括在超过activeDeadlineSeconds（在Jobs期间使用）之后驱逐Pod。
	- 接下来，Pod 的状态（Phase）由其 init 和真实容器的状态决定。由于我们的容器尚未启动，因此容器被归类为[waiting](https://github.com/kubernetes/kubernetes/blob/fc8bfe2d8929e11a898c4557f9323c482b5e8842/pkg/kubelet/kubelet_pods.go#L1244)。任何具有 waiting 容器的 Pod 都处于[Pending](https://github.com/kubernetes/kubernetes/blob/fc8bfe2d8929e11a898c4557f9323c482b5e8842/pkg/kubelet/kubelet_pods.go#L1258-L1261)阶段。
 	- 最后，Pod 状态由其容器的状况决定。由于我们的容器尚未由容器运行时创建，因此它将[PodReady条件](https://github.com/kubernetes/kubernetes/blob/fc8bfe2d8929e11a898c4557f9323c482b5e8842/pkg/kubelet/status/generate.go#L70-L81)设置为 False。
3. 生成 PodStatus 后，它将被发送到 Pod 的状态管理器，该管理器的任务是通过 apiserver 异步更新 etcd 记录。
4. 接下来，运行一系列准入处理程序以确保 pod 具有正确的安全权限。这些包括强制执行[A​​ppArmor配置文件和NO_NEW_PRIVS](https://github.com/kubernetes/kubernetes/blob/fc8bfe2d8929e11a898c4557f9323c482b5e8842/pkg/kubelet/kubelet.go#L883-L884)。在此阶段被拒绝的 Pod 将无限期地处于 Pending 状态。
5. 如果已指定 --cgroups-per-qos 运行时标志，则 kubelet 将为 pod 创建 cgroup 并应用资源参数。这是为了实现更好的服务质量（QoS）处理。
6. 为 pod [创建数据目录](https://github.com/kubernetes/kubernetes/blob/dd9981d038012c120525c9e6df98b3beb3ef19e1/pkg/kubelet/kubelet_pods.go#L772)。这些包括 pod 目录（通常是 /var/run/kubelet/pods/<podID> ），其卷目录（<podDir>/volumes）及其插件目录（<podDir>/plugins）。
7. 卷管理器将[附加并等待](https://github.com/kubernetes/kubernetes/blob/2723e06a251a4ec3ef241397217e73fa782b0b98/pkg/kubelet/volumemanager/volume_manager.go#L330) Spec.Volumes 中定义的任何相关卷。根据要安装的卷的类型，某些 pod 需要等待更长时间（例如，云或 NFS 卷）。
8. 从 Apiserver 中[检索](https://github.com/kubernetes/kubernetes/blob/dd9981d038012c120525c9e6df98b3beb3ef19e1/pkg/kubelet/kubelet_pods.go#L788) Spec.ImagePullSecrets 中定义的所有 Secrets，以便接下来可以将它们注入容器中。
9. 最后容器运行时运行容器（在下面更详细地描述）

## CRI and pause containers（容器运行时接口和Pause容器）
我们现在完成了大部分启动设置，并且容器已准备好启动。执行此启动任务的程序称为容器运行时 Container Runtime（例如docker或rkt）。

为了让 kubelet 更具可扩展性，自 v1.5.0 以来的 kubelet 一直使用一个名为 CRI（容器运行时接口）的接口来与具体的容器运行时进行交互。简而言之，CRI 提供了 kubelet 和特定运行时实现之间的接口抽象。通过 [protocol buffer](https://github.com/google/protobuf)（像一个更快的 JSON ）和一个 [gRPC API](https://grpc.io/)（一种非常适合执行 Kubernetes 操作的 API ）进行通信。这是一个非常酷的想法，因为通过在 kubelet 和运行时之间使用已定义的契约，容器编排方式的具体实现细节变得无关紧要。重要的是协议约定，这允许以最小的开销添加新的运行时，因为 Kubernetes 核心代码完全无需更改！

不好意思，有点离题了，让我们回到我们的容器部署...当一个 pod 首次启动时，kubelet调用 [RunPodSandbox](https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/pkg/kubelet/kuberuntime/kuberuntime_sandbox.go#L51)远程程序调用（RPC）。“沙箱”是描述一组容器的 CRI 术语，在 Kubernetes 的说法中，你可能已经猜到了，就是一个容器。这个术语是故意模糊的，因此它不会失去其他可能实际上并不使用容器的作为运行时的意义（想象一个基于虚拟机管理程序的运行时，沙箱可能是一个VM）。

在我们的例子中，我们使用的是 Docker 。在此运行时中，创建沙箱涉及创建 “pause” 容器。 pause 容器像 Pod 中的所有其他容器的父容器一样，因为它承载了工作负载容器最终将使用的许多pod级资源。这些“资源”是 Linux 命名空间（IPC，网络，PID）。如果您不熟悉容器在 Linux 中的工作方式，那么我们来快速回顾一下。 Linux 内核具有命名空间的概念，允许主机操作系统分割出一组专用资源（例如 CPU 或内存）并将其提供给进程，就像是进程的专属资源一样。 Cgroup在这里也很重要，因为它们是 Linux 管理资源分配的方式。 Docker 使用这两种内核功能来托管一个保证资源和强制隔离的进程。更多相关信息，请查看 b0rk 的精彩帖子：[什么是容器](https://jvns.ca/blog/2016/10/10/what-even-is-a-container/)

“pause”容器提供了一种托管所有这些命名空间的方法，并允许子容器共享它们。通过成为同一网络命名空间的一部分，提供的一个好处是同一个 pod 中的容器可以使用 localhost 相互通信。 

“pause”容器的第二个功能与 PID 命名空间的工作方式有关。在 PID 命名空间中，进程形成一个树状结构，一旦某个子进程由于父进程的错误而变成了“孤儿进程”，顶部的“init”进程负责进行收养并最终回收资源。有关其工作原理的更多信息，请查看这篇精彩的博客文章。[The Almighty Pause Container](https://www.ianlewis.org/en/almighty-pause-container)。

一旦创建好了 pause 容器，下面就会开始检查磁盘状态然后开始启动业务容器。


## CNI and pod networking（容器网络接口和pod网络）
我们的Pod现在有了它的骨架：一个“pause”容器，它托管所有命名空间以允许pod间通信。但网络如何运作以及如何建立网络呢？

当kubelet为pod设置网络时，它将任务委托给 “CNI” 插件。CNI 代表 Container Network Interface，其运行方式与 Container Runtime Interface 类似。简而言之，CNI 是一种抽象，允许不同的网络提供商对容器使用不同的网络实现。通过 stdin 将 JSON 数据（配置文件位于 /etc/cni/net.d 中）传输到相关的 CNI 二进制文件（位于/ opt/cni/bin 中），在 kubelet 里注册插件并与它们进行交互。 这是 JSON 配置的示例：

```
{
    "cniVersion": "0.3.1",
    "name": "bridge",
    "type": "bridge",
    "bridge": "cnio0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "ranges": [
          [{"subnet": "${POD_CIDR}"}]
        ],
        "routes": [{"dst": "0.0.0.0/0"}]
    }
}
```
它还通过 CNI_ARGS 环境变量为 pod 指定其他元数据，例如其名称和命名空间。

接下来会发生什么取决于 CNI 插件，但让我们来看看 CNI 插件：

1. 该插件将首先在根网络命名空间中设置本地 Linux 网桥，以便为该主机上的所有容器提供服务。
2. 然后，它会将一个网络 interface（一个 veth 对的一端）插入"pause"容器的网络命名空间，并将另一端连接到网桥。理解一个 veth 对的最好方法就是把它想像一跟大管：一边连接到容器，另一边是根网络命名空间，允许数据包在中间传输。
3. 然后它为 “pause” 容器的网络接口分配 IP 并设置路由。这样 Pod 就拥有了自己的 IP 地址。 IP 分配工作委托给在 JSON 配置中指定的 IPAM 程序。
  - IPAM 插件类似于主网络插件：它们通过二进制文件调用并具有标准化接口。每个必须确定容器接口的 IP/Subnets以及网关和路由，并将此信息返回给主插件。最常见的 IPAM 插件称为 host-local，并从预定义的一组地址范围中分配 IP 地址。它将状态存储在主机文件系统上，从而确保单个主机上 IP 地址的唯一性。
4. 对于 DNS，kubelet 将为 CNI 插件指定内部 DNS 服务器 IP 地址，这将确保正确设置容器的 resolv.conf 文件。

一旦完成上述过程，插件将把 JSON 数据返回到 kubelet，表明操作的结果。

## Inter-host networking（跨主机网络）
到目前为止，我们已经描述了容器如何连接到主机，但主机之间如何通信？不同机器上的两个 Pod 显然需要通信。

这通常使用称为覆盖网络的概念来实现，这是一种跨多个主机动态同步路由的方法。一个流行的覆盖网络提供者是 flannel。安装后，其核心职责是在群集中的多个节点之间提供第3层 IPv4 网络通信。 flannel 不控制容器如何与本地主机的通信（这是 CNI 的工作），而是在主机之间传输流量。为此，它为主机选择一个子网并将其注册到 etcd 中。然后，它保留集群路由的本地表示，并将传出的数据包封装在 UDP 数据报中，确保它到达正确的主机。有关更多信息，请查看[CoreOS的文档](https://github.com/coreos/flannel)。


## Container startup（容器启动）
所有的网络工作都已完成。那还剩下什么？接下来我们就开始启动实际工作负载容器。

沙盒完成初始化并处于活跃状态后， kubelet 可以开始为其创建容器。它首先启动 PodSpec 中定义的所有 [init容器](https://github.com/kubernetes/kubernetes/blob/5adfb24f8f25a0d57eb9a7b158db46f9f46f0d80/pkg/kubelet/kuberuntime/kuberuntime_manager.go#L690)，然后再启动主容器。过程如下：

1. [拉取容器镜像](https://github.com/kubernetes/kubernetes/blob/5f9f4a1c5939436fa320e9bc5973a55d6446e59f/pkg/kubelet/kuberuntime/kuberuntime_container.go#L90)。PodSpec 中定义的所有 secrets 都可以用于私有镜像仓库。
2. 通过 CRI [创建容器](https://github.com/kubernetes/kubernetes/blob/5f9f4a1c5939436fa320e9bc5973a55d6446e59f/pkg/kubelet/kuberuntime/kuberuntime_container.go#L115)。它通过从父 PodSpec 填充 ContainerConfig 数据结构（其中定义了命令，镜像，标签，挂载，设备，环境变量等），然后通过 protobufs将其发送到 CRI 插件来实现。对于 Docker，它将 payload 反序列化并填充自己的配置信息，再将其发送到 Daemon API。在此过程中，它会将一些元数据标签（例如容器类型，日志路径，沙箱ID）添加到容器中。
3. 然后通过 CPU 管理器来约束容器，这是1.8中的一个新加的alpha功能，它通过使用 UpdateContainerResources CRI 方法将容器分配给本地节点上的 CPU 资源池。
4. 然后正式[启动](https://github.com/kubernetes/kubernetes/blob/5f9f4a1c5939436fa320e9bc5973a55d6446e59f/pkg/kubelet/kuberuntime/kuberuntime_container.go#L135)容器。
5. 如果注册了任何 post-start 容器生命周期钩子，这个时候就会[运行](https://github.com/kubernetes/kubernetes/blob/5f9f4a1c5939436fa320e9bc5973a55d6446e59f/pkg/kubelet/kuberuntime/kuberuntime_container.go#L156-L170)这些钩子。钩子可以是 Exec 类型（在容器内执行特定命令）或 HTTP（对容器端点执行HTTP请求）。如果 PostStart 钩子运行时间太长，挂起或失败，容器将永远不会达到 running 状态。

## 总结

好吧，终于完成了，好长。


毕竟，我们在一个或多个工作节点上运行了3个容器。所有的网络，卷和secrets都由kubelet填充，并通过CRI插件制作成容器。

## 原文
<https://github.com/jamiehannaford/what-happens-when-k8s>