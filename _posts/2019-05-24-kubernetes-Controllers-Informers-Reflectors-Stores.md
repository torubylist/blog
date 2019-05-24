---
layout:     post
title:      "【译】Kubernetes: Controllers, Informers, Reflectors and Stores"
subtitle:   " \"Kubernetes: Controllers, Informers, Reflectors and Stores\""
date:       2019-05-24 20:00:00
author:     "会飞的蜗牛"
header-img: "img/kubernetes-controller-informer-reflector-store.jpg"
tags:
    - Kubernetes
    - Controller
    - Informer
    - Reflector
    - Store
---

> 翻译主要是为了更好的掌握文中的内容，如有不妥之处，请给我发邮件，谢谢。yongpingzhao1#gmail.com

众所周知，Kubernetes给我们提供了强大的数据结构，用以获取API服务器资源的本地表征。

为了我的学士论文，我正在用Kubernetes进行开发工作。我的合作伙伴和我正在研发一个在Kubernetes上执行工作流的[系统](https://github.com/nov1n/kubernetes-workflow)。我们都非常喜欢Kubernetes的思维架构-将整个系统视为一个控制系统。也就是说，这个系统不断尝试将其当前状态移动到期望状态。确保系统达到期望状态的工作单元称为[ Controller-控制器](https://kubernetes.io/docs/user-guide/replication-controller/#alternatives-to-replication-controller)。由于我们想要实现自己的工作流控制器，决定查看JobController。就这样，我们偶然发现了控制器的秘密所在：

```
1 jm.jobStore.Store, jm.jobController = framework.NewInformer(
2  &cache.ListWatch{
3    ListFunc: func(options api.ListOptions) (runtime.Object, error) {
4       // Direct call to the API server, using the job client
5      return jm.kubeClient.Batch().Jobs(api.NamespaceAll).List(options)
6    },
7    WatchFunc: func(options api.ListOptions) (watch.Interface, error) {
8      // Direct call to the API server, using the job client
9     return jm.kubeClient.Batch().Jobs(api.NamespaceAll).Watch(options)
10    },
11  },
12   〜
13  framework.ResourceEventHandlerFuncs{
14     AddFunc: jm.enqueueController,
15     UpdateFunc: 〜
16     DeleteFunc: jm.enqueueController,
17   },
18 )
```

似乎返回了一个具有 list 和 watch 功能 JobsStore ，但我们还不知道它到底是如何起作用的。在这个案例中，第5行和第9行的 list 和 watch 功能实现是使用 job 客户端直接调用 api-server。看起来是不是非常酷？你给它提供一个 list和 watch api-server 的接口，Informer 就自动将上游数据同步到下游存储，甚至为你提供一些方便使用的事件钩子。

但有时候我们也会观察到一些我们没想到的行为。例如：UpdateFunc 每30秒调用一次，而上游实际上没有更新。这时我决定深入了解 Informers，Controller，Reflector 和 Store 背后的运作方式。我首先将试图解释 Controller 的工作原理，然后我将解释控制器如何在内部使用Reflector 和 DeltaFIFO 存储，最后会说明 Informers 是怎样的一个包装器，用于同步上下游的存储。

请注意，Kubernetes 正如日中天，发展迅猛。代码库在本文写作和您阅读本文的这段时间内可能发生了变化。我是在这个[blob](https://github.com/kubernetes/kubernetes/tree/82cb4c17581752ae9f00bd746a63e529149c04b4/pkg)上引用代码。

# Controllers（控制器）
当我们想要知道 informers 是如何工作的时，我们就需要知道控制器是如何工作的，那最好的例子就在[controller_test.go](https://github.com/kubernetes/kubernetes/blob/82cb4c17581752ae9f00bd746a63e529149c04b4/pkg/controller/framework/controller_test.go)中。

```
1 // 数据源模拟apiserver终端对象
2 // 是Reflector的数据源
3 source := framework.NewFakeControllerSource()
4 
5 //以我们熟悉的方式保存下游状态
6 downstream := cache.NewStore(framework.DeletionHandlingMetaNamespaceKeyFunc)

7 //保留传入的更改。注意我们如何以KeyLister的形式传递下游，这样重新同步操作就会产生正确的update/delete增量集合。 
8 // 这将是Reflector的存储。
...
12 fifo := cache.NewDeltaFIFO(cache.MetaNamespaceKeyFunc, nil, downstream)
13
14 // 通过线程安全的方式输出预期结果
15 deletionCounter := make(chan string, 1000)
16
17 // 给controller配置一个数据源，FIFO队列，以及一个处理函数。
18 cfg := &framework.Config{
19   Queue:            fifo,
20   ListerWatcher:    source,
21   ObjectType:       &api.Pod{},
22   FullResyncPeriod: time.Millisecond * 100,
23   RetryOnError:     false,
24 	
25 	// 实现一个简单的控制器，删除进来的一切源数据。
26	
27   Process: func(obj interface{}) error {
28  	// Obj是从队列中Pop出来的。
29     newest := obj.(cache.Deltas).Newest()
30
31     if newest.Type != cache.Deleted {
32       //更新下游存储
33       err := downstream.Add(newest.Object)
34       if err != nil {
35         return err
36       }
37 	   
38 	   // 删除源里的对象	
39       source.Delete(newest.Object.(runtime.Object))
40     } else {
41       //更新下游存储
42       // Update our downstream store.
43       err := downstream.Delete(newest.Object)
44       if err != nil {
45         return err
46       }
47       
48		 //	fifo的KeyOf函数是最容易的，因为它处理
49		 // DeletedFinalStateUnknown 制造者
50      // fifo's KeyOf is easiest, because it handles
51      // DeletedFinalStateUnknown markers.
52      key, err := fifo.KeyOf(newest.Object)
53      if err != nil {
54        return err
55      }
56
57      // 报告这次删除事件.
58      deletionCounter <- key
59     }
60    return nil
61   },
62 }
63 
64 // 创建和运行controller，知道我们关闭stop通道.
65 stop := make(chan struct{})
66 defer close(stop)
67 go framework.New(cfg).Run(stop)
68 
69 // 给数据源添加数据.
70 testIDs := []string{"a-hello", "b-controller", "c-framework"}
71 for _, name := range testIDs {
72 	//注意，这些pods是无效的，假的数据源不会调用其他诸如验证之类的函数。
73   source.Add(&api.Pod{ObjectMeta: api.ObjectMeta{Name: name}})
74 }
75 
76 //等待controller处理我们刚添加的事件
77 outputSet := sets.String{}
78 for i := 0; i < len(testIDs); i++ {
79   outputSet.Insert(<-deletionCounter)
80 }
81 
82 for _, key := range outputSet.List() {
83   fmt.Println(key)
84 }
```


输出：

	a-hello
	b-controller
	c-framework

让我们看看这是如何工作的！第3行声明了一个数据来源。此源通常是 API 服务器的客户端。现在它是一个假的来源，这样我们就可以更好的控制它的行为。第6行声明了下游存储，我们将使用它来获得源的本地表示。第12行声明了一个 DeltaFIFO 队列，用于跟踪源和下游之间的差异。要配置控制器，我们需要为它提供 FIFO 队列，数据源和处理器。

Process 循环包含将系统的当前状态达成期望状态的逻辑。进程函数接收一个 obj，它是来自 FIFO 队列的 Deltas 数组。在我们的示例中，我们检查 delta 是否为 Deleted 以外的任何类型。如果不是 Deleted类型，属于 delta 的 Object 将添加到下游存储，将 Object 从源中删除。如果 delta是 Deleted 类型，则从下游删除 Object，并在 deletionCounter 通道上发送消息。

第一次运行这个例子的时候，对于 newest.Type=cache.Deleted 是否应该在31行，我有点困惑。为什么不直接检查是否添加了类型？我决定在第30行放置以下print语句：

`fmt.Printf（“[％v]％v \ n”，newest.Object。（* api.Pod）.Name，newest.Type）`。

这是我得到的输出：

	[a-hello] Sync
	[b-controller] Sync
	[c-framework] Sync
	[a-hello] Deleted
	[b-controller] Deleted
	[c-framework] Deleted

所以从上面的输出可以看出，当添加了某个内容时，我们实际上也做了同步事件，这是为什么呢？让我们来一起来解密吧！

## Controller Run
示例67行创建并运行了一个controller，一起来看下run背后的机制：

```
1 // Run开始处理items，知道stopCH返回消息
2 // 调用Run超过一次就会发生错误
3 // Run函数是阻塞的; 通过go关键字调用.
4 func (c *Controller) Run(stopCh <-chan struct{}) {
5   defer utilruntime.HandleCrash()
6  // 创建一个新的Reflector
7   r := cache.NewReflector(
7     c.config.ListerWatcher,
9     c.config.ObjectType,
10     c.config.Queue,
11     c.config.FullResyncPeriod,
12   )
13   〜
14   // 调用RunUntil, 这是非阻塞的
15   r.RunUntil(stopCh)
16   // 调用processLoop，完成后，等待1秒，又调用一次。
17   // 这是一个无限循环，直到我们从stopCh接收到信号。
18   wait.Until(c.processLoop, time.Second, stopCh)
19 }
```
首先创建一个新的 reflector。 给它配置了 ListerWatcher（源）和队列（DeltaFIFO）。 然后，在 reflector 上调用 RunUntil ，这是一个非阻塞调用。最后使用[wait.Until](https://github.com/kubernetes/kubernetes/blob/82cb4c17581752ae9f00bd746a63e529149c04b4/pkg/util/wait/wait.go#L46)调用processLoop。 processLoop 排空 FIFO 队列并使用队列中弹出的 Deltas 调用 Process 函数：

```
1 func (c *Controller) processLoop() {
2   for {
3     obj := c.config.Queue.Pop()
4     err := c.config.Process(obj)
5     〜
6   }
7 }
```
我们现在知道如何调用示例中的 Process 函数，但我们仍然不知道 FIFO Queue 中的内容如何。 要了解其工作原理，我们必须深入研究 Reflector 。

# Reflectors
根据[reflector.go](https://github.com/kubernetes/kubernetes/blob/82cb4c17581752ae9f00bd746a63e529149c04b4/pkg/client/cache/reflector.go)中的注释，“Reflector 监听指定的资源并使所有更改反映在给定的存储中”。 在我们的示例中，资源是第3行的源，而存储是第12行的 DeltaFIFO 。

创建反射器时，调用其 RunUntil 方法。 让我们来看看它做了什么：

```
1 func (r *Reflector) RunUntil(stopCh <-chan struct{}) {
2   〜
3   go wait.Until(func() {
4     if err := r.ListAndWatch(stopCh); err != nil {
5       utilruntime.HandleError(err)
6     }
7   }, r.period, stopCh)
8 }
```
所以它一直调用 ListAndWatch ，直到 stopCh 收到消息。现在让我们深入了解[ListAndWatch](https://github.com/kubernetes/kubernetes/blob/82cb4c17581752ae9f00bd746a63e529149c04b4/pkg/client/cache/reflector.go#L281)，这个让魔法真正发生的地方：

```
1 // 最初，ListAndWatch列出所有的项，同时获取资源版本 
2 // 然后使用资源版本做监听的参数。
3 // 如果ListAndWatch不能初始化watch就会返回error
4 func (r *Reflector) ListAndWatch(stopCh <-chan struct{}) error {
5   〜
6   resyncCh, cleanup := r.resyncChan()
7   defer cleanup()
8 
9   // 显式的将资源版本设置为“0”，这对List()来说是可以的。
10  // 为了从cache中获取数据，跟etcd内容有关的可能会被推迟
11  // etcd contents. Reflector框架最终会定期通过Watch()获取数据
12   options := api.ListOptions{ResourceVersion: "0"}
13   // 初次调用List.
14   list, err := r.listerWatcher.List(options)
15   〜
16   listMetaInterface, err := meta.ListAccessor(list)
17   〜
18   // 从list中获取resourceVersion.
19   resourceVersion = listMetaInterface.GetResourceVersion()
20   items, err := meta.ExtractList(list)
21   〜
22   if err := r.syncWith(items, resourceVersion); err != nil {
23     return fmt.Errorf("%s: Unable to sync list result: %v", r.name, err)
24   }
25   // 设置resourceVersion这样Watch可以使用。
26   r.setLastSyncResourceVersion(resourceVersion)
27	  // 不终止循环，除非未知的error错误，
28   // 函数返回一个引起循环终止的错误，
29   // 删除了错误处理
30   // 为了更好的阅读感. 当ListAndWatch返回时,
31   // 会被RunUntil再次调用
32   for {
33     options := api.ListOptions{
34        ResourceVersion: resourceVersion,
35        〜
36     }
37     w, err := r.listerWatcher.Watch(options)
38     〜
39     if err := r.watchHandler(w, &resourceVersion, resyncCh, stopCh); err != nil {
40       〜
41       if err != errorResyncRequested {
42         return nil
43       }
44     }
45     if r.canForceResyncNow() {
46       〜
47       if err := r.store.Resync(); err != nil {
48         return err
49       }
50       cleanup()
51       resyncCh, cleanup = r.resyncChan()
52     }
53   }
54 }
```
方法上面的注释说：“ ListAndWatch 首先列出所有项目，并在调用时获取资源版本，然后使用资源版本监听”。 过程如下图所示：

![](http://borismattijssen.github.io/images/Reflector%20Time%20Diagram.png)

在 t=0 时刻，我们的 ListerWatcher- 在我们的例子中是上游源。 List函数帮助我们获得列表中所有项目的最新资源版本。在 t>0 时，我们继续观察 ListerWatcher ，以获得比获得的 resourceVersion 更新的更改。如果你还不知道资源版本是什么，请查看[API约定](https://github.com/kubernetes/kubernetes/blob/master/docs/devel/api-conventions.md#concurrency-control-and-consistency)。

那么它如何转化为 ListAndWatch 的代码？ 在第14行（t=0），List 函数被调用。列表中的items将传递给 syncWith 方法。SyncWith 使用列表中的项调用 store.Replace：

```
1 func (r *Reflector) syncWith(items []runtime.Object, resourceVersion string) error {
2   found := make([]interface{}, 0, len(items))
3   for _, item := range items {
4     found = append(found, item)
5   }
6   return r.store.Replace(found, resourceVersion)
7 }
```

后文再解释store.Replace是如何工作的。

在初始列表之后，我们开启了一个无限for循环（t> 0）。 在这个循环中，我们使用[watch.Interface](https://github.com/kubernetes/kubernetes/blob/82cb4c17581752ae9f00bd746a63e529149c04b4/pkg/watch/watch.go#L26)调用watchHandler方法。[watchHandler](https://github.com/kubernetes/kubernetes/blob/82cb4c17581752ae9f00bd746a63e529149c04b4/pkg/client/cache/reflector.go#L369)的内容是以下一段阻塞代码：

```
1 // watchHandler监听并时刻保持更新*resourceVersion.
2 func (r *Reflector) watchHandler(w watch.Interface, resourceVersion *string, resyncCh <-chan time.Time, stopCh <-chan struct{}) error {
3   〜
4  defer w.Stop()
4   〜
5   for {
6     select {
7      case <-stopCh:
8       return errorStopRequested
9     // 当设置了纳秒resyncPeriod.
10     case <-resyncCh:
11      return errorResyncRequested
12     case event, ok := <-w.ResultChan():
13       〜
14       // 抓住watch的事件.
15       switch event.Type {
16       case watch.Added:
17         r.store.Add(event.Object)
18       case watch.Modified:
19         r.store.Update(event.Object)
20       case watch.Deleted:
21        〜
22         r.store.Delete(event.Object)
23       〜
24       }
25     }
26   }
27   〜
28   return nil
29 }

```
在第12-22行，捕获来自上游的监听事件并调用相应的存储方法。而在第10-11行，当在resyncCh上收到消息时，watchHandler返回。那么消息如何在resyncCh中结束？

当我们回顾一下[ListAndWatch](https://borismattijssen.github.io/articles/kubernetes-informers-controllers-reflectors-stores#list-and-watch)的代码时，我们在6-7行中找到答案。第6行调用[resyncChan](https://github.com/kubernetes/kubernetes/blob/82cb4c17581752ae9f00bd746a63e529149c04b4/pkg/client/cache/reflector.go#L233)方法，该方法返回一个在 r.resyncPeriod 纳秒后接收消息的定时器通道。此通道传递给w atchHandler，以便在 r.resyncPeriod ns之后返回。当 watchHandler返回时，ListAndWatch 方法最终可以调用 [canForceResyncNow](https://github.com/kubernetes/kubernetes/blob/82cb4c17581752ae9f00bd746a63e529149c04b4/pkg/client/cache/reflector.go#L269)方法。如果我们能够进行下一次定期的Resync，则CanForceResyncNow返回true。在那种情况下，调用store.Resync。所以最终情况如下：


![](http://borismattijssen.github.io/images/Reflector%20Time%20Diagram2.png)

所以现在我们知道Reflector何时调用DeltaFIFO存储。下面让我们弄清楚存储的运作方式！

# DeltaFIFO Store
根据 [delta_fifo.go](https://github.com/kubernetes/kubernetes/blob/82cb4c17581752ae9f00bd746a63e529149c04b4/pkg/client/cache/delta_fifo.go) 中的注释，“DeltaFIFO 是生产者-消费者队列，其中 Reflector 旨在成为生产者，而调用 Pop() 方法的就是消费者”。 在我们的例子中，我们的控制器的 processLoop 方法是消费者。

从它的定义我们知道 DeltaFIFO 拥有一个队列，一个字符串数组和一个 items 映射，其键对应于队列中的字符串：


```
1 type DeltaFIFO struct {
2   〜
3   items map[string]Deltas
4   queue []string
5   〜
6 }
```
items 将键映射到 Deltas 数组。 Delta 会告诉您发生了哪些更改（Add，Update，Delete，Sync）以及更改后对象的状态。

让我们看一下从Reflector调用的Add，Update，Delete，Replace和Resync方法。

## Add, Update, and Delete
Add，Update和Delete方法都使用相应的[DeltaTypes](https://github.com/kubernetes/kubernetes/blob/82cb4c17581752ae9f00bd746a63e529149c04b4/pkg/client/cache/delta_fifo.go#L513)调用queueActionLocked方法。 在[queueActionLocked](https://github.com/kubernetes/kubernetes/blob/82cb4c17581752ae9f00bd746a63e529149c04b4/pkg/client/cache/delta_fifo.go#L273)中，给定的obj被插入到队列中。

## Replace
[Replace](https://github.com/kubernetes/kubernetes/blob/82cb4c17581752ae9f00bd746a63e529149c04b4/pkg/client/cache/delta_fifo.go#L393)获取items列表。使用 Sync DeltaType 将每个item入队。 然后，它使用knownObject（在我们的例子中是对下游存储的引用）来查看item是否删除。如果是已删除，已删除的事件将进入队列。请注意，这就是我们在控制器示例中获取Sync事件的原因。在Reflector开始于t=0之前插入三个pods（可能并非总是如此，因为启动Reflector并在存储中插入pods是并行任务）。因此，在初始列表中找到了三个pod，然后调用store.Replace方法。由于store.Replace仅触发新项目的同步事件，因此我们未找到任何已添加事件。

## Resync
[Resync](https://github.com/kubernetes/kubernetes/blob/82cb4c17581752ae9f00bd746a63e529149c04b4/pkg/client/cache/delta_fifo.go#L459) 发送Sync事件给knownObjects中所有的items-就是下游存储里的内容

# Recap on Controller, Reflector and Store
信息太多了，我们再来回顾一下。

## The controller:

- 引用了一个FIFO队列
- 引用ListerWatcher（在我们的例子中是上游源）
- 负责消耗FIFO队列
- 有一个进程循环，负责使系统达成期望的状态
- 创建一个Reflector

## The reflector:

- 引用相同的FIFO队列（内部称为存储）
- 引用了同一个ListerWatcher
- 列出并监听ListerWatcher
- 负责产生FIFO队列的输入
- 负责在每个resyncPeriod ns上调用FIFO队列上的Resync方法。

## The FIFO queue:

- 引用下游存储;
- Deltas队列，用于Reflector列出和监听的对象。

# Informer
我们现在知道 Controller，Reflector和FIFO Queue 如何协同工作以与上游数据源保持同步。 因此，让我们看一下 Informer 如何使用这些组件将上游源同步到下游源。根据[controller.go](https://github.com/kubernetes/kubernetes/blob/82cb4c17581752ae9f00bd746a63e529149c04b4/pkg/controller/framework/controller.go)的注释，“NewInformer返回一个cache.Store和一个控制器，用于填充本地存储，同时还提供事件通知”。 这就是[NewInformer](https://github.com/kubernetes/kubernetes/blob/82cb4c17581752ae9f00bd746a63e529149c04b4/pkg/controller/framework/controller.go#L198)函数的功能：

```
1 func NewInformer(
2   lw cache.ListerWatcher,
3   objType runtime.Object,
4   resyncPeriod time.Duration,
5   h ResourceEventHandler,
6 ) (cache.Store, *Controller) {
7   // 保存客户端状态
8   // 下游存储
9   clientState := cache.NewStore(DeletionHandlingMetaNamespaceKeyFunc)
10
11   // 保存输入的变化. 注意我们给clientState传入了一个KeyLister
12   // resync操作将返回update/delete deltas的结果
13   // 
14   fifo := cache.NewDeltaFIFO(cache.MetaNamespaceKeyFunc, nil, clientState)
15 
16   cfg := &Config{
17     Queue:            fifo,
18     ListerWatcher:    lw,
19     ObjectType:       objType,
20     FullResyncPeriod: resyncPeriod,
21     RetryOnError:     false,
22
23     Process: func(obj interface{}) error {
24      // from oldest to newest
25       for _, d := range obj.(cache.Deltas) {
26         switch d.Type {
27         case cache.Sync, cache.Added, cache.Updated:
28          if old, exists, err := clientState.Get(d.Object); err == nil && exists {
29             if err := clientState.Update(d.Object); err != nil {
30               return err
31             }
32             h.OnUpdate(old, d.Object)
33          } else {
34             if err := clientState.Add(d.Object); err != nil {
35               return err
36             }
37             h.OnAdd(d.Object)
38           }
39         case cache.Deleted:
40           if err := clientState.Delete(d.Object); err != nil {
41             return err
42           }
43           h.OnDelete(d.Object)
44         }
45       }
46      return nil
47     },
48   }
49   return clientState, New(cfg)
50 }
```
这基本上就是一个controller的模版代码了，用于将事件从FIFO队列同步到下游存储。它需要一个ListerWatcher和一个ResourceEventHandler，在JobController源代码中如下：

```
1 jm.jobStore.Store, jm.jobController = framework.NewInformer(
2   &cache.ListWatch{
3   ListFunc: func(options api.ListOptions) (runtime.Object, error) {
4       // 直接调用APIserver, 使用job客户端
5      return jm.kubeClient.Batch().Jobs(api.NamespaceAll).List(options)
6    },
7     WatchFunc: func(options api.ListOptions) (watch.Interface, error) {
8       // 直接调用APIserver, 使用job客户端
9       return jm.kubeClient.Batch().Jobs(api.NamespaceAll).Watch(options)
10     },
11   },
12   〜
13   framework.ResourceEventHandlerFuncs{
14     AddFunc: jm.enqueueController,
15     UpdateFunc: 〜
16     DeleteFunc: jm.enqueueController,
17   },
18 )
```

Process 函数处理所有 Delta 事件。它在下游存储上调用相应的 Add，Update和Delete 方法-在此代码中就是clientState。请注意，ResourceEventHandlerFuncs 没有 SyncFunc 。因此，在收到 Sync 事件时会调用AddFunc或UpdateFunc-即使对象未更新但仅重新同步。

我希望这篇文章能够让你清楚地了解 Kubernetes 中使用的 Controller，Informer，Reflector和Store概念，这些概念可以让我们的生活更加美好。 欢迎随时评论或批评:)


最后，感谢[@nov1n](https://github.com/nov1n) 帮我做的这些研究.



## 原文
<http://borismattijssen.github.io/articles/kubernetes-informers-controllers-reflectors-stores>
