---
layout:     post
title:      "深入理解kubernetes controller"
subtitle:   "Controller in kubernetes"
date:       2019-04-20 21:00:00
author:     "倔强蜗牛"
header-img: "img/k8s.jpeg"
tags:
    - Kubernetes
    - Controller
    - Informer/Workqueue
---

kubernetes集群上运行着许多的线程，这些线程的主要任务就是调整集群的实际状态，使之与集群的期望状态达成一致。例如，ReplicasSets控制着集群的正在运行的pod数，而Node Controller则观察着集群的节点的状态，并且对节点状态的改变作出正确的相应。总的来说，集群的每个controller都控制着kubernetes集群里的一种特殊的资源。作为集群的用户，正确理解集群里面每个controller的角色和任务是很有必要的。然而你有没有真正的想过，这些集群的controller到底是怎么工作的，又或者，你有没有想过亲自实现一个controller呢？

这是系列文章的第一篇，着重关注kubernetes controller的内部机制，在第二篇文章中，我将带领你们一起实现一个简单的通知系统。

我在本文中使用的所有代码块都是Kubernetes控制器的实现中公开的，这些控制器都是用Golang编写的，并且基于[client-go](https://github.com/kubernetes/client-go)库。

### Controller Pattern
我们可以在官方文档中见到关于controller的详细解释：
>In applications of robotics and automation, a control loop is a non-terminating loop that regulates the state of the system. In Kubernetes, a controller is a control loop that watches the shared state of the cluster through the API server and makes changes attempting to move the current state towards the desired state. Examples of controllers that ship with Kubernetes today are the replication controller, endpoints controller, namespace controller, and serviceaccounts controller.

 [Kube-controller-manager](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/)

这段引言的意思是说，在机器人与自动化的应用里，控制循环是一个非终止循环，用于调节系统状态。而在kubernetes里面，controller是一个通过apiserver监听集群共享状态，并进行更改以尝试将当前状态移至期望状态的控制循环。如今，与Kubernetes一起提供的控制器示例包括ReplicaController，EndpointController，Namespace Controller和ServiceAccounts Controller等。

为了降低复杂性，所有控制器都打包并运送到名为kube-controller-manager的单个守护程序中。 控制器最简单的实现是循环：
```Golang
for {
  desired := getDesiredState()
  current := getCurrentState()
  makeChanges(desired, current)
}
```

### Controller Components
控制器有两个主要组件：Informer/SharedInformer和Workqueue。 Informer/SharedInformer监视Kubernetes对象当前状态的更改，并将事件发送到Workqueue，然后由工作线程弹出事件以进行处理。

#### Informer
Kubernetes控制器的重要作用是观察对象的期望状态和实际状态的，然后发送指令以使实际状态更像所需状态。为了获取对象的信息，控制器向Kubernetes API服务器发送请求。

但是，反复从API服务器获取信息可能会变得昂贵。 因此，为了在代码中多次获取和列出对象，Kubernetes开发人员最终使用已经由client-go库提供的缓存。 此外，控制器并不真的想要连续发送请求。 它只关心创建，修改或删除对象时的事件。 client-go库提供Listwatcher接口，该接口执行初始列表并在特定资源上启动监视：
```golang
lw := cache.NewListWatchFromClient(
      client,
      &v1.Pod{},
      api.NamespaceAll,
      fieldSelector)
```
所有这些事件都是在Informer中消费。Informer的一般结构如下所述：
```golang
store, controller := cache.NewInformer {
	&cache.ListWatch{},
	&v1.Pod{},
	resyncPeriod,
	cache.ResourceEventHandlerFuncs{},
```
尽管Informer在当前的Kubernetes中并没有被大量使用(而是使用了SharedInformer，我将在后面解释)，但是当你想要编写自定义控制器时，它仍然是一个必不可少的概念。 以下是用于构造Informer的三种模式：

##### ListWatcher
Listwatcher是特定命名空间中特定资源的列表函数和监听函数的组合。 这有助于控制器只关注它想要查看的特定资源。 字段选择器是一种过滤器，用于缩小搜索资源的结果，比如控制器想要检索与特定字段匹配的资源。Listwatcher的结构如下所述：
```golang
cache.ListWatch {
	listFunc := func(options metav1.ListOptions) (runtime.Object, error) {
		return client.Get().
			Namespace(namespace).
			Resource(resource).
			VersionedParams(&options, metav1.ParameterCodec).
			FieldsSelectorParam(fieldSelector).
			Do().
			Get()
	}
	watchFunc := func(options metav1.ListOptions) (watch.Interface, error) {
		options.Watch = true
		return client.Get().
			Namespace(namespace).
			Resource(resource).
			VersionedParams(&options, metav1.ParameterCodec).
			FieldsSelectorParam(fieldSelector).
			Watch()
	}
}

```
##### ResourceEventHandler
资源事件处理程序是控制器处理特定资源更改通知的地方：
```golang
type ResourceEventHandlerFuncs struct {
	AddFunc    func(obj interface{})
	UpdateFunc func(oldObj, newObj interface{})
	DeleteFunc func(obj interface{})
}
```
* AddFunc 创建新资源时会调用它。
* UpdateFunc 修改现有资源时会调用。oldObj是资源的最后已知状态。 重新同步发生时也会调用UpdateFunc，即使没有任何更改也会调用它，会对比资源的resourceVersion。
* DeleteFunc 删除现有资源时会调用。
 它获取资源的最终状态（如果已知）。 否则，它将获得DeletedFinalStateUnknown类型的对象。 如果监听关闭并且错过了删除事件并且控制器在后续重新列出资源之前没有注意到删除事件，则会发生这种情况。

##### ResyncPeriod
ResyncPeriod定义了控制器遍历缓存中剩余项目的频率，并再次触发UpdateFunc。这提供了一种配置，以周期性地验证当前状态并使其成为所需状态。

在控制器可能错过更新或先前操作失败的情况下，它非常有用。 但是，如果你构建一个自定义控制器，如果设置的周期时间太短，则必须小心CPU负载可能会过高，因为这会增加系统的cpu负载。

#### SharedInformer
informer创建仅由其自身使用的一组资源的本地缓存。 但是，在Kubernetes中，有一组控制器在运行和关注多种资源。 这意味着将存在重叠 - 一个资源正由多个控制器进行处理。

在这种情况下，SharedInformer有助于在多个控制器之间创建单个共享缓存。这意味着不会复制缓存的资源，通过这样做，可以降低系统的内存开销。此外，每个SharedInformer仅在上游服务器上创建一个监听，而不管有多少下游消费者正在从通知者中读取事件。 这也减少了上游服务器的负载。 这对于具有多个多内部控制器的kube-controller-manager来说很常见。

SharedInformer已经提供了钩子来接收添加，更新和删除特定资源的通知。它还提供了便捷功能，用于访问共享缓存并确定何时启动缓存。 这节省了我们与API服务器的连接，服务器端的重复序列化成本，控制器端的重复反序列化成本以及控制器端的重复高速缓存成本。

```golang
lw := cache.NewListWatchFromClient(…)
sharedInformer := cache.NewSharedInformer(lw, &api.Pod{}, resyncPeriod)
```

#### Workqueue
SharedInformer无法跟踪每个控制器的位置（因为它是共享的），因此控制器必须提供自己的排队和重试机制（重试机制是可选的，如果需要的话）。因此，大多数资源事件处理程序只是将项目放在每个消费者的工作队列中。

每当资源发生更改时，资源事件处理程序都会将一个键放入Workqueue。 键使用格式```<resource_namespace> / <resource_name>```，除非```<resource_namespace>```为空，这样的话就只有```<resource_name>```。 通过这样做，事件按键折叠，因此每个消费者可以使用工作程序弹出键，使得这项工作按顺序完成，这将保证没有两个工作程序会同时处理同一个键值。

Workqueue在client-go/util/workqueue的client-go库中提供。 支持的队列有几种，包括延迟队列，定时队列和速率限制队列。

以下是创建速率限制队列的示例：
```golang
queue :=
workqueue.NewRateLimitingQueue(workqueue.DefaultControllerRateLimiter())
```
Workqueue提供了管理键值的便利功能。 下图描述了Workqueue中键值的生命周期：
![Workqueue Life Cycle][1]

在处理事件时失败的情况下，控制器调用AddRateLimited（）函数将其键返回到工作队列，以便稍后使用预定义的重试次数进行处理。 否则，如果进程成功，则可以通过调用Forget（）函数从工作队列中删除键值。但是，该功能仅阻止工作队列跟踪事件的历史记录，为了从工作队列中完全删除事件，控制器必须触发Done（）函数。

因此，工作队列可以处理来自缓存的通知，但问题是，控制器应该何时启动工作线程来处理工作队列？答案是控制器应该等到缓存完全同步以实现最新状态，原因有两个：

1. 在缓存完成同步之前，列出所有资源将是不准确的。

2. 对单个资源的多个快速更新将由缓存/队列折叠到最新版本中。 因此，它必须等到缓存变为空闲才能实际处理项目，以避免在中间状态上浪费工作。

伪代码如下：
```golang
controller.informer = cache.NewSharedInformer(...)
controller.queue = workqueue.NewRateLimitingQueue(workqueue.DefaultControllerRateLimiter())

controller.informer.Run(stopCh)

if !cache.WaitForCacheSync(stopCh, controller.HasSynched)
{
	log.Errorf("Timed out waiting for caches to sync"))
}

// Now start processing
controller.runWorker()
```
### 综述
到目前为止，我刚刚概述了Kubernetes控制器：它是什么，它用于什么情况，它是由哪些组件构成的，以及它是如何工作的。 最令人兴奋的则是是Kubernetes让用户集成他们自己的控制器。

  [1]: https://engineering.bitnami.com/images/kubernetes-controller/key-lifecicle-workqueue.png
  [2]: https://engineering.bitnami.com/articles/a-deep-dive-into-kubernetes-controllers.html