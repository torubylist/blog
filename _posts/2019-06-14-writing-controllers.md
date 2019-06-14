---
layout:     post
title:      "关于如何写Controllers的几点指导意见"
subtitle:   " \"Writing Controllers\""
date:       2019-06-14 20:00:00
author:     "会飞的蜗牛"
header-img: "img/writing-controllers.jpg"
tags:
    - Controllers
    - Kubernetes
    
---

Kubernetes控制器是一个主动调协过程。也就是说，它一边会监听事物的期望状态，同时，它也会监听事物当前的真实状态。然后，如果它发现真实状态和期望状态不一致，就会发送指令以尝试使事物的当前状态往期望状态靠近，最终跟期望状态一致。

简化版的实现-循环：

```
for {
  desired := getDesiredState()
  current := getCurrentState()
  makeChanges(desired, current)
}

```
监听等方法只是用来进行逻辑优化。

## 指导方针
如果你需要写一个controller，那么这里有几点指导意见，你可以看看，我相信这更有助于你写出没有bug，性能更优的控制器。

1. 一次只操作一个条目。这意思是说，如果你使用`workqueue.Interface`， 那么你会先将观察到某个资源的变化入队，然后再将它们取出来，这样可以多个“工作线程”将对他们进行处理，这样的话，在任意时刻，不会有多个gofunc同时处理一个条目。这个条目指的是workqueue里面的条目。
	
	很多控制器必须触发多个资源（当“Y”改变时，检查“X”的状态），但几乎所有控制器都可以根据依赖关系将这些资源折叠成“检查X”的队列。例如，当一个“Pod”被删除时，ReplicaSet控制器需要作出相应的反应，通过检查相关的Replicasets，并将相关的replicaset入队。

2. 不同资源之间存在随机排序，当controllers入队多个不同类型的资源时，并没有机制能够确保这些资源的先后顺序。
	
	不同的监听之间是相互独立的，即使存在着不同的事件，例如“创建了ResourceA/X，创建了B/Y”，但是你的控制器看到的可能是“创建了ResourceB/Y“，然后“创建了A/X”。
	
3. 水平触发，而不是边缘触发。就像拥有一个一直没有运行的shell脚本一样，你的控制器可能会在再次运行之前关闭一段不确定的时间。
	
	一个API对象展现出来的结果是`true`，但是你不能因此推断说，你看到它从false变成了true，而仅仅只是你看到它的当前状态是true。API监听常常遇到这种问题，所以你不能依赖于看到这个变化过程，除非你的控制器在对象的status字段标记了它最后做出决定的信息。

4. 使用SharedInformers。 SharedInformers提供钩子以接收特定资源的添加，更新和删除的通知。还提供了其他的功能，用于访问共享缓存并确定缓存的启动时间。

	使用<https://github.com/kubernetes/kubernetes/blob/master/staging/src/k8s.io/client-go/informers/factory.go>中的工厂方法，以确保你使用方式和其他人一样共享相同的实例。

	这节省了我们与API服务器的连接开销，服务器端的重复序列化与控制器端的重复反序列化成本以及控制器端的重复新建缓存成本。

	您可能之前也看到过其他机制，如反射器和deltafifos驱动控制器。那些是我们后来用于构建SharedInformer的旧机制。您应该避免在新控制器中直接使用它们。

5. 永远不要改变原始对象！缓存是在控制器之间共享的，这意味着如果你改变了某个对象的“副本”（实际上是对象的引用或浅拷贝），你将打乱其他控制器的工作（而不仅仅是你自己的控制器）。

	我们最常见的容易引起失败的操作是对象的浅拷贝，例如去改变一个map，如Annotations。 所以请使用api.Scheme.Copy进行深层复制。


6. 等待二级缓存的同步。许多控制器具有主要和次要资源。主要资源是你要更新状态的资源。辅助资源是您将要管理（创建/删除）或用于查找的资源。

	在启动主同步功能之前，使用framework.WaitForCacheSync函数等待二级缓存。 这将确保像ReplicaSet的Pod计数之类的东西不能即使处理过时信息而导致删除操作。

7. 因为我们的系统中还有其他控制器。所以仅仅因为你没有改变一个对象并不意味着其他人没有。
  
   不要忘记当前状态可能随时改变-所以仅仅观察期望状态是不够的。如果你使用了在期望状态中查找不到的对象，则表示应删除当前状态的内容，请确保你观察实际状态的代码中没有bug（例如，在缓存填充之前执行操作）。

8. 将错误渗透到上层对象以实现一致的重新入队。 我们有一个workqueue.RateLimitingInterface，允许合理的重试，进行简单的重新排队。

	当需要重新入队时，主控制器func应该返回错误。如果不需要重新入队，则应使用utilruntime.HandleError并返回nil。 这使得复查者可以非常轻松地检查错误处理案例，并确保控制器不会意外丢失它应该重试的内容。
	
9. 通知器和监听器需要进行”同步“。他们会定期将群集中的每个匹配对象传递给Update方法。这适用于你可能需要对对象执行其他操作的情况，但有时你知道这并没有必要。

	如果你确定在没有新更改时不需要重新入队，则可以比较新旧对象的资源版本，如果版本一致则跳过。这样做时一定要非常小心。如果你在程序发生故障时跳过重新入队，则可能会使程序失败，因为不会重新入队，然后再也不会重试该项目。

10. 如果控制器正在协调的主资源支持ObservedGeneration，请确保当ObservedGeneration与metadata.Generation两个字段的值不匹配时，需要将ObservedGeneration设置为metadata.Generation。

	这样，客户端就知道控制器已处理了资源。所以确保你的控制器是负责该资源的主控制器，否则如果你需要通过自己的控制器进行通信观察，则需要在资源的状态中创建不同类型的ObservedGeneration。
	
11. 考虑使用owner reference来创建相关的其他资源（例如，ReplicaSet会导致创建Pod）。 因此，一旦你删除了主控制器管理的资源，就可以确保对子资源进行垃圾收集。有关owner reference的更多信息，请在此处阅读[更多](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/controller-ref.md)。

    如果你有收养策略，一定要特别谨慎。当父资源或子资源被标记为删除时，不能收养子资源。如果你正在使用缓存，则可能需要使用API从apiserver直接读取信息来绕过缓存，以防止其中一个子项更新了owner reference而你并不知道的情况，这样的话会发生冲突。 因此，你需要确保控制器不与垃圾收集器发生竞争。
    
    详细信息请看<https://github.com/kubernetes/kubernetes/pull/42938>

## 代码轮廓
最终，你的代码轮廓应该差不多如下所示：

```
type Controller struct{
	// podLister pods次级缓存，用于主资源查找时使用。
	podLister cache.StoreToPodLister

	// queue 工作任务入口，可以去重，允许发生错误的时候限速重新入队。

	queue workqueue.RateLimitingInterface
}

func (c *Controller) Run(threadiness int, stopCh chan struct{}){
	// 当发生panics的时候，不要导致进程crash。
	defer utilruntime.HandleCrash()
	// 确保队列关闭，处理完全部任务后。
	defer c.queue.ShutDown()

	glog.Infof("Starting <NAME> controller")

	// 开始同步工作之前，填充次级cache。
	if !framework.WaitForCacheSync(stopCh, c.podStoreSynced) {
		return
	}

	// 开启一个或者工作线程，有些controller有多种类型的任务。 

	for i := 0; i < threadiness; i++ {
		// runWorker会一直循环，直到意外的事情发生。Until会每隔一秒钟触发一次worker
		
		go wait.Until(c.runWorker, time.Second, stopCh)
	}

	// 退出进程
	<-stopCh
	glog.Infof("Shutting down <NAME> controller")
}

func (c *Controller) runWorker() {
	// 热循环，直到被告知退出。processNextWorkItem会一直等待直到有任务可做，
	// 所以无需担心二次等待。
	for c.processNextWorkItem() {
	}
}

// processNextWorkItem 处理出队的键(namespace/name)，如果结束了就返回false。

func (c *Controller) processNextWorkItem() bool {
	// 从队列里面拉取下一个任务，这个任务就是一个可以从本地cache查找的key。
	
	key, quit := c.queue.Get()
	if quit {
		return false
	}
	// 每次退出的时候必须告知队列你已经完成的工作。
	
	defer c.queue.Done(key)
	
	// 根据key值，处理你自己的任务逻辑
	
	err := c.syncHandler(key.(string))
	if err == nil {
		// 如果没有返回错误，则需要告知队列停止跟踪这个key。这个forget
		// 函数也会重置每个limit rate queue条目的失败次数。
		
		c.queue.Forget(key)
		return true
	}
	
	// 任务失败了一定要报告错误。 此方法允许可插入的错误处理，可用于集群监控等事情
	
	utilruntime.HandleError(fmt.Errorf("%v failed with : %v", key, err))
	
    // 因为我们任务失败了，所以我们应该重新入队该项目以便再次运行任务。
    // 这里会增加一个回退机制
    // 以避免在某个特定项目上一直循环不退出（因为它们可能仍然无法立即工作）
    // 宏观来说，这是控制器自我保护策略（任务失败了，控制器需要冷却下来，否则它
    // 可能会一直循环，饿死其他有用的任务）。
		
	c.queue.AddRateLimited(key)

	return true
}

```

## 参考
<https://github.com/kubernetes/community/blob/8decfe4/contributors/devel/controllers.md>












