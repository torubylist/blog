---
layout:     post
title:      "[译]kubernetes admission webhook指南"
subtitle:   " \"A Guide to Kubernetes Admission Controllers\""
date:       2019-06-23 20:00:00
author:     "会飞的蜗牛"
header-img: "img/admission-webhook.jpg"
tags:
    - Admission Webhook
    - Kubernetes
    
---

Kubernetes甫一出现，就席卷了整个基础架构的世界，极大的提高了当今生产环境中后端集群的管理效率。由于其灵活性，可扩展性和易用性，从而成为容器编排领域的事实标准。同时，kubernetes还提供了一系列保护生产环境工作负载的安全支持，其中就包括我们今天要讲的“准入控制器”插件。必须启用准入控制器才能使用Kubernetes的一些更高级的安全功能，例如在整个命名空间中强制实施安全配置的pod安全策略。 下文将帮助你利用准入控制器在Kubernetes中充分利用这些安全功能。


# 什么是kubernetes安全准入控制器
简单的说，kubernetes准入控制器就是一组插件，用来管理集群被使用的方式。可以认为他们是一个给集群看门的，用来拦截已经经过认证的请求，可以修改请求的对象内容或者拒绝请求。准入控制器一般分为两阶段。先修改阶段，然后验证阶段，也可以二者只取其一。例如LimitRanger准入控制器。既能在修改阶段增大pod的资源请求和限制，也能在第二阶段验证请求的资源是否超过是否超过每个namespace的上限，超过上限的请求予以拒绝。



![](https://d33wubrfki0l68.cloudfront.net/af21ecd38ec67b3d81c1b762221b4ac777fcf02d/7c60e/images/blog/2019-03-21-a-guide-to-kubernetes-admission-controllers/admission-controller-phases.png)


值得我们注意的是，用户使用的诸多kubernetes操作背后多有admission controller的身影。例如当一个namespace被删除之后进入`Terminating`阶段，这时候`NamesapceLifeCycle`就会拒绝其他pod创建在该namespace里。

在众多的kubernetes准入控制器里，有两个比较特殊，那就是`ValidatingAdmissionWebhooks `和`MutatingAdmissionWebhooks`，它俩使用起来非常灵活，在kubernetes1.13中已经是beta版本了。这两个准入控制器自身并没有任何规则逻辑，与之相替的是一个跑在集群里的REST http服务。这种方式可以将控制逻辑与kubernetes api本身进行解耦。这样，用户可以随意的定制业务逻辑，可以针对所有的kubernete资源的增删改查。

两种准入控制器webhooks之间的差异几乎是不言自明的：`MutatingAdmissionWebhooks`可能会修改对象，而`ValidatingAdmissionWebhooks`则不会。 然而，`MuatingAdmissionWebhook`也可以拒绝请求，从而以验证的方式行事。 `ValidatingAdmissionWebhooks`相对于`MuatatingAdmissionWebhooks`有两个主要优点：首先，出于安全原因，可能会禁用`MutatingAdmissionWebhooks`（或者对`MutatingWebhookConfiguration`对象应用更严格的`RBAC`限制），因为它可能会造成混乱甚至危险操作的副作用。 其次，如上图所示，验证准入控制器（包括`webhooks`）在任何变异控制之后运行。 因此，验证`webhook`看到的任何请求对象都是将持久保存到`etcd`的最终版本。

通过标志传递给Kubernetes API服务器来配置启用的准入控制器集。 请注意，旧的-admission-control标志在1.10中已弃用，并替换为-enable-admission-plugins。

```
--enable-admission-plugins=ValidatingAdmissionWebhook,MutatingAdmissionWebhook
```

Kubernetes默认推荐的准入控制器如下：

```
--enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,Priority,ResourceQuota,PodSecurityPolicy
```
完整的准入控制器可以在[官方的文档](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#what-does-each-admission-controller-do)中找到。本文仅讨论webhook方式的准入控制器。


# 为什么需要准入控制器
**安全性**：准入控制器可以通过在整个命名空间或集群中强制使用合理的安全基准来提高安全性。 内置的PodSecurityPolicy门禁控制器就是最好的例子; 例如，它可以用于禁止容器以root身份运行，或者确保容器的根文件系统始终以只读方式挂载。 可通过基于webhook的自定义准入控制器实现的其他用例包括：

- 允许用户仅从企业已知的特定registries中拉取图像，同时拒绝未知的镜像仓库。
- 拒绝不符合安全标准的部署。例如，使用特权标志的容器会规避许多安全检查。 基于webhook的准入控制器可以减轻此风险，该准入控制器拒绝此类部署（验证）或覆盖特权标志，将其设置为false。

**治理**：准入控制器强制你遵守某些做法，例如良好的标签，注释，资源限制或其他设置。一些常见的场景包括：

- 对不同对象强制执行标签验证，以确保将正确的标签用于各种对象，例如分配给团队或项目的每个对象，或指定每个deployments的应用程序标签。
- 自动向对象添加注释，例如为“dev”部署资源分配到正确的数据中心。

**配置管理**：准入控制器允许你验证群集中运行的对象的配置，并防止任何明显的错误配置进入群集。准入控制器可用于检测和修复没有语义标签的部署镜像，例如：

- 自动添加资源限制或验证资源限制
- 确保给pods添加合理的标签
- 确保生产部署中使用的镜像不使用latest标记或带有-dev后缀的标记。


# 如何实现和部署一个Admission Controller Webhook

为了说明如何利用准入控制器webhook来建立自定义安全策略，让我们考虑一个解决Kubernetes缺点之一的例子：kubernetes的许多默认值都经过优化，易于使用并减少摩擦，有时是以牺牲安全性为代价的。其中一个设置是默认允许容器以root身份运行（并且，如果没有进一步的配置，Dockerfile中也没有USER指令，也会这样做）。尽管容器在一定程度上与底层主机隔离，但以root身份运行容器确实会增加部署的风险级别 - 应该避免作为许多安全性[最佳实践](https://www.stackrox.com/post/2018/12/6-container-security-best-practices-you-should-be-following/)之一。例如，最近暴露的[runC漏洞（CVE-2019-5736）](https://www.stackrox.com/post/2019/02/the-runc-vulnerability-a-deep-dive-on-protecting-yourself/)只有在容器以root身份运行时才能被利用。

您可以使用自定义修改准入控制器webhook来应用更安全的默认值：除非明确请求，否则我们的webhook将确保pod作为非root用户运行（如果未进行明确分配，我们将分配用户ID1234）。请注意，此设置不会阻止你在群集中部署任何工作负载，包括那些合法需要以root身份运行的工作负载。它只要求您在部署配置中明确启用此风险程序操作模式，而对所有其他工作负载默认为非root模式。

完整的代码以及部署说明可以在我们的[GitHub](https://github.com/stackrox/admission-controller-webhook-demo)存储库中找到。在这里，我们将重点介绍webhook如何工作的一些更微妙的方面。

## Mutating Webhook Configuration
该配置主要是用于告诉apiserver，当符合该配置文件里面的请求出现时，需要转发到该外部http服务。

```
apiVersion: admissionregistration.k8s.io/v1beta1
kind: MutatingWebhookConfiguration
metadata:
  name: demo-webhook
webhooks:
  - name: webhook-server.webhook-demo.svc
    clientConfig:
      service:
        name: webhook-server
        namespace: webhook-demo
        path: "/mutate"
      caBundle: ${CA_PEM_B64}
    rules:
      - operations: [ "CREATE" ]
        apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["pods"]
```
此配置定义webhook webhook-server.webhook-demo.svc，并指示在创建pod时,Kubernetes API服务器需要通过向webhook-demo命名空间的webhook-server服务的/mutate URL发出HTTP POST请求。要使此配置生效，必须满足几个先决条件。


## Webhook REST API
`Kubernetes API`服务器向给定服务和URL路径发出`HTTPS POST`请求，并在请求正文中使用`JSON`编码的`AdmissionReview`（设置了`Request`字段）。 响应应该是`JSON`编码的`AdmissionReview`，这次设置了`Response`字段。

我们的演示存储库包含一个处理序列化/反序列化样板代码的函数，并专注于实现在`Kubernetes API`对象上运行的逻辑。在我们的示例中，实现准入控制器逻辑的函数称为`applySecurityDefaults`，在`/mutate` URL下提供此功能的`HTTPS`服务器可以设置如下：

```
mux := http.NewServeMux()
mux.Handle("/mutate", admitFuncHandler(applySecurityDefaults))
server := &http.Server{
  Addr:    ":8443",
  Handler: mux,
}
log.Fatal(server.ListenAndServeTLS(certPath, keyPath))
```

请注意，要使服务器在没有提升权限的情况下运行，我们让HTTP服务器侦听端口8443.Kubernetes不允许在webhook配置中指定端口; 它始终采用HTTPS端口443.但是，由于无论如何都需要服务对象，我们可以轻松地将服务的端口443映射到容器上的端口8443：

```
apiVersion: v1
kind: Service
metadata:
  name: webhook-server
  namespace: webhook-demo
spec:
  selector:
    app: webhook-server  # specified by the deployment/pod
  ports:
    - port: 443
      targetPort: webhook-api  # name of port 8443 of the container
```
## 对象修改逻辑

在变异的准入控制器webhook中，通过JSON补丁执行修改操作。 虽然JSON补丁标准包含许多复杂的逻辑，这远远超出了本讨论的范围，但我们的示例中的Go数据结构及其用法应该为用户提供有关JSON补丁如何工作的初步概述：

```
type patchOperation struct {
  Op    string      `json:"op"`
  Path  string      `json:"path"`
  Value interface{} `json:"value,omitempty"`
}
```
要将``pod` 的字段 `.spec.securityContext.runAsNonRoot` 设置为true，我们构造以下 `patchOperation` 对象：

```
patches = append(patches, patchOperation{
  Op:    "add",
  Path:  "/spec/securityContext/runAsNonRoot",
  Value: true,
})
```
## TLS证书
由于必须通过 `HTTPS` 提供 `webhook` ，因此我们需要适当的服务器证书。 这些证书可以是自签名的（即：由自签名CA签名），但我们需要 `Kubernetes	` 在与 `webhook` 服务器通信时出示相应的CA证书。此外，证书的公用名（CN）必须与 `Kubernetes API` 服务器使用的服务器名称匹配，内部服务的名称是`<service-name>.<namespace>.svc`， 在我们的案例中即 `webhook-server.webhook-demo.svc`。 由于自签名TLS证书的生成在Internet上有详细说明，因此我们只需在示例中引用相应的shell脚本。

之前显示的webhook配置包含占位符`${CA_PEM_B64}`。 在我们创建此配置之前，我们需要将此部分替换为 `Base64` 编码的 `CA` 的`PEM`证书。 `openssl base64 -A` 命令可用于此目的。


## 测试Webhook
在部署webhook服务器并对其进行配置之后，可以通过调用./deploy.sh脚本来完成，现在是时候测试并验证webhook是否确实完成了它的工作。 存储库包含三个示例：

1. 未指定安全上下文的pod（pod-with-defaults）。 我们希望此pod以非root身份运行，用户ID为1234。
2. 一个确定安全上下文的pod，明确允许它以root身份运行（pod-with-override）。
3. 具有冲突配置的pod，指定它必须以非root用户身份运行，但用户ID为0（pod-with-conflict）。 为了展示拒绝对象创建请求，我们增加了我们的准入控制器逻辑，以拒绝这些明显的错误配置。
通过运行kubectl create -f examples/<name>.yaml创建其中一个pod。 在前两个示例中，你可以通过检查日志来验证pod运行的用户ID，例如：

```
$ kubectl create -f examples/pod-with-defaults.yaml
$ kubectl logs pod-with-defaults
I am running as user 1234
```

In the third example, the object creation should be rejected with an appropriate error message:

```
$ kubectl create -f examples/pod-with-conflict.yaml
```

```
Error from server (InternalError): error when creating "examples/pod-with-conflict.yaml": Internal error occurred: admission webhook "webhook-server.webhook-demo.svc" denied the request: runAsNonRoot specified, but runAsUser set to 0 (the root user)
```

# 参考
<https://kubernetes.io/blog/2019/03/21/a-guide-to-kubernetes-admission-controllers/>


  
  