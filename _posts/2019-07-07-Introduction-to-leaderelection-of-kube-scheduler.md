---
layout:     post
title:      "kube-scheduler选主机制介绍"
subtitle:   " \"Introduction To Leaderelection of Kube-scheduler\""
date:       2019-07-09 20:00:00
author:     "会飞的蜗牛"
header-img: "img/leaderelection-k8s-scheduler.jpg"
tags:
    - ResourceLock
    - Leaderelection
    - Kubernetes
    
---

## Kubernetes高可用模式

Kubernetes的核心组件包括ETCD, APIServer, Scheduler, Controller-Manager。其中ETCD是存放集群所有配置文件的，Watch集群信息的变化，同时保存到磁盘上。一般是3副本部署，基于Raft协议实现其高可用性与数据的一致性。后面会有对Raft协议有详细的介绍，这里就不赘述了。而APIServer也是高可用的，属于Active-Active模式，某种意义上可以说APIServer是ETCD的前端代理，但其也有自己的业务逻辑，比如Authentication/Authorization/Admission-Controller等。而scheduler和controller-manager分别负责资源调度和集群资源控制。所谓任务调度就是pod的最终归属，由哪个Node来负责运行某个pods，这里就涉及到资源的分配，kubernete资源包括可压缩资源（CPU）以及非压缩资源（内存和Disk等）。集群控制管理器管理集群的控制器，这是k8s的精华所在，虽然k8s的scheduler做的也不错，但真正让kubernetes闪耀的是它的控制器模式。kubernete的控制器模式将集群看作是个控制循环，目的就是让集群的最终状态跟期望状态一致。期望状态是用户告知APIServer的，控制器会监听APIServer里某个资源的变化，而实际状态是它通过某个方式拿到的。它关心的就是最终实际状态和期望状态一致，至于中间发生了什么，它并不关心，也无需关心，这就是控制器模式的妙处。那么scheduler和controller一般都是多副本的，他们又是怎样高可用的呢？是不是也不是跟APIServer一样多活呢？答案是这两个组件并不是多活，而是Leader-StandBy。任意时刻只有一个Leader在干活。如果正在运行的leader因某种原因导致当前进程退出，或者锁丢失，则由其它副本去竞争新的leader，获取leader继而执行业务逻辑。那么这个过程具体是怎么实现的呢？

## 选主过程
Scheduler和ControllerManager并没有依赖外部的组件例如ETCD或者Zookeeeper来选主，而是通过抢资源分布式锁来获得主控制权，并开启主循环，没有拿到锁的进程则不会开启主循环。Kubernetes的资源并不仅仅只能用于处理任务，而且可以实现一个分布式锁，因为任何资源都是全局唯一的，通过Namespace/Name来区分。拿到这个锁的就是主，而其他两个是Standy。目前client-go里面锁的实现主要有三种，分别是configmaps，endpoints以及lease。而kubernetes-scheduler以及controllermanager目前支持configmaps和endpoints。
我们来看下scheduler的配置参数：


```
--leader-elect     默认: true
在执行主循环之前，启动 leader 选举客户端并获得leader资格。运行复制组件以实现高可
用性时启用此选项。
--leader-elect-lease-duration duration     默认: 15s
非leader在观察leader续约之后将等待的时间，直到试图获得领导但尚未更新的 leader 位
置。这实际上是 leader 在被另一个候选人替换之前可以停止的最长持续时间。这仅适用于
启用 leader 选举的情况。
--leader-elect-renew-deadline duration     默认: 10s
代理 master 在leader停止之前更新leader的时间间隔。这必须小于或等于租约期限。
这仅适用于启用 leader 选举的情况
--leader-elect-resource-lock endpoints     默认: "endpoints"
在 leader 选举期间用于锁定的资源对象的类型。支持的选项是 endpoints (默认) 
和 configmaps。
--leader-elect-retry-period duration     默认: 2s
客户端在尝试获取和更新领导之间应该等待的持续时间。这仅适用于启用leader选举的情况。

```

`--leader-elect` 默认设置为true，在执行主循环之前，启动 leader 选举客户端并获得leader资格。如果为false，则每个副本都参与实际工作。

`leader-elect-lease-duration` 为资源锁租约观察时间，如果其它竞争者在该时间间隔过后发现leader没更新获取锁时间，则其它副本可以认为leader已经挂掉不参与工作了，将重新选举leader。

`leader-elect-renew-deadline` leader在该时间内没有更新则失去leader身份。

`leader-elect-retry-period` 为其它副本获取锁的时间间隔(竞争leader)和leader更新间隔。

`leader-elect-resource-lock` 是k8s分布式资源锁的资源对象，目前只支持endpoints和configmas。


## 实现分析

```
// Package leaderelection implements leader election of a set of endpoints.
// It uses an annotation in the endpoints object to store the record of the
// election state.
//
// This implementation does not guarantee that only one client is acting as a
// leader (a.k.a. fencing). A client observes timestamps captured locally to
// infer the state of the leader election. Thus the implementation is tolerant
// to arbitrary clock skew, but is not tolerant to arbitrary clock skew rate.
//
// However the level of tolerance to skew rate can be configured by setting
// RenewDeadline and LeaseDuration appropriately. The tolerance expressed as a
// maximum tolerated ratio of time passed on the fastest node to time passed on
// the slowest node can be approximately achieved with a configuration that sets
// the same ratio of LeaseDuration to RenewDeadline. For example if a user wanted
// to tolerate some nodes progressing forward in time twice as fast as other nodes,
// the user could set LeaseDuration to 60 seconds and RenewDeadline to 30 seconds.
//
// While not required, some method of clock synchronization between nodes in the
// cluster is highly recommended. It's important to keep in mind when configuring
// this client that the tolerance to skew rate varies inversely to master
// availability.
//
// Larger clusters often have a more lenient SLA for API latency. This should be
// taken into account when configuring the client. The rate of leader transitions
// should be monitored and RetryPeriod and LeaseDuration should be increased
// until the rate is stable and acceptably low. It's important to keep in mind
// when configuring this client that the tolerance to API latency varies inversely
// to master availability.


```

包 `leaderelection` 实现了一组基于 `endpoints/configmaps` 的领导者选举机制。它使用`endpoints/configmaps` 对象中的注释来存储选举状态的记录。

此实现不保证只有一个客户端充当领导者（a.k.a. fencing）。客户端观察本地捕获的时间戳以推断领导者选举的状态。因此，该实现容许有时钟偏差，但不能容忍任意的时钟偏差率。

但是，可以通过适当设置`RenewDeadline`和 `LeaseDuration` 来配置对偏差率的容忍度。偏差容忍度表示为在最快节点上传递的时间与在最慢节点上传递的时间的比率，可以通过设置`LeaseDuration` 与 `RenewDeadline` 为相同的比率的配置来近似实现。例如，如果用户想要容忍某些节点的时间比其他节点快一倍，则用户可以将 `LeaseDuration` 设置为60秒，将 `RenewDeadline` 设置为30秒。

虽然时钟同步不是必需的，但强烈建议在群集中节点中使用。在配置此客户端时，请务必记住偏差率的公差与主可用性成反比。这个意思是说公差越大，主可用性越低。因为时钟差别大，发生主切换的概率更高，就导致频繁切主，从而导致主不可用的比例大大增加。

对于API延迟，较大的群集通常具有更宽松的SLA。配置客户端时应考虑这一点。应监测领导者转换的速率，并应增加 `RetryPeriod` 和 `LeaseDuration`，让它维持在一个可以接受的较低的切换速率，如果频繁切换，会导致 主循环可用性降低，从而降低了组件的效率。在配置此客户端时，请务必牢记API延迟的公差与主可用性成反比。

kube-scheduler中的代码：

```
// 如果leader-election为true，则创建一个leaderelection对象。

var leaderElectionConfig *leaderelection.LeaderElectionConfig
if config.LeaderElection.LeaderElect {
    leaderElectionConfig, err = makeLeaderElectionConfig(config.LeaderElection, leaderElectionClient, recorder)
    if err != nil {
        return nil, err
    }
}


// 该函数创建一个leaderelection配置文件，并创建resource lock对象。
// 该对象返回的是一个leaderelection包中的resourcelock 接口。

func makeLeaderElectionConfig(config componentconfig.KubeSchedulerLeaderElectionConfiguration, client clientset.Interface, recorder record.EventRecorder) (*leaderelection.LeaderElectionConfig, error) {
    // 获得当前锁的对象标识
    hostname, err := os.Hostname()
    if err != nil {
        return nil, fmt.Errorf("unable to get hostname: %v", err)
    }
    // 调用k8s的leaderelection包去生成锁对象
    // 后面会专门讲解leaderelection包
    rl, err := resourcelock.New(config.ResourceLock,
        config.LockObjectNamespace,
        config.LockObjectName,
        client.CoreV1(),
        resourcelock.ResourceLockConfig{
            Identity:      hostname,
            EventRecorder: recorder,
        })
    if err != nil {
        return nil, fmt.Errorf("couldn't create resource lock: %v", err)
    }

    return &leaderelection.LeaderElectionConfig{
        Lock:          rl,
        // 租约时间间隔
        LeaseDuration: config.LeaseDuration.Duration,
        // leader持有锁时间
        RenewDeadline: config.RenewDeadline.Duration,
        // 其它副本重试(竞争leader)时间间隔
        RetryPeriod:   config.RetryPeriod.Duration,
    }, nil
}
```

```
//新建一个资源锁对象，可接受的类型包括Configmaps，Endpoints以及Leases
func New(lockType string, ns string, name string, coreClient corev1.CoreV1Interface, coordinationClient coordinationv1.CoordinationV1Interface, rlc ResourceLockConfig) (Interface, error) {
	switch lockType {
	case EndpointsResourceLock:
		return &EndpointsLock{
			EndpointsMeta: metav1.ObjectMeta{
				Namespace: ns,
				Name:      name,
			},
			Client:     coreClient,
			LockConfig: rlc,
		}, nil
	case ConfigMapsResourceLock:
		return &ConfigMapLock{
			ConfigMapMeta: metav1.ObjectMeta{
				Namespace: ns,
				Name:      name,
			},
			Client:     coreClient,
			LockConfig: rlc,
		}, nil
	case LeasesResourceLock:
		return &LeaseLock{
			LeaseMeta: metav1.ObjectMeta{
				Namespace: ns,
				Name:      name,
			},
			Client:     coordinationClient,
			LockConfig: rlc,
		}, nil
	default:
		return nil, fmt.Errorf("Invalid lock-type %s", lockType)
	}
}
```
使用刚创建的leaderelectionconfig，开启选举，若选举成功则调用run函数，执行scheduler逻辑。

```
// Prepare a reusable run function.
run := func(stopCh <-chan struct{}) {
    // scheduler的调度逻辑
    sched.Run()
    <-stopCh
}

// If leader election is enabled, run via LeaderElector until done and exit.
if s.LeaderElection != nil {
    s.LeaderElection.Callbacks = leaderelection.LeaderCallbacks{
        // 回调函数，如果选举成功则执行run函数，run函数就是上面定义的
        OnStartedLeading: run,
        // 若因某种原因导致失去leader，执行该函数
        OnStoppedLeading: func() {
            utilruntime.HandleError(fmt.Errorf("lost master"))
        },
    }
    // 根据scheduler的leader配置去返回leader对象，主要是判定配置的leader config是否正确
    leaderElector, err := leaderelection.NewLeaderElector(*s.LeaderElection)
    if err != nil {
        return fmt.Errorf("couldn't create leader elector: %v", err)
    }
    // 开始选举，若获得锁成为leader则执行OnStartedLeading，若获取锁失败则不断重试，直到成为leader
    leaderElector.Run()

    return fmt.Errorf("lost lease")
}

```

## client-go leaderelection包分析
k8s.io/client-go/tools/leaderelection

```
~/go/src/k8s.io/client-go/tools/leaderelection   master ●  tree .
.
├── OWNERS
├── healthzadaptor.go
├── healthzadaptor_test.go
├── leaderelection.go //leader选举主要逻辑
├── leaderelection_test.go
├── metrics.go
└── resourcelock
    ├── configmaplock.go //configmap资源锁的实现
    ├── endpointslock.go //endpoint资源锁的实现
    ├── interface.go
    └── leaselock.go //lease 资源锁的实现
```
首先调用newleaderelection

```
func NewLeaderElector(lec LeaderElectionConfig) (*LeaderElector, error) {
	if lec.LeaseDuration <= lec.RenewDeadline {
		return nil, fmt.Errorf("leaseDuration must be greater than renewDeadline")
	}
	if lec.RenewDeadline <= time.Duration(JitterFactor*float64(lec.RetryPeriod)) {
		return nil, fmt.Errorf("renewDeadline must be greater than retryPeriod*JitterFactor")
	}
	if lec.LeaseDuration < 1 {
		return nil, fmt.Errorf("leaseDuration must be greater than zero")
	}
	if lec.RenewDeadline < 1 {
		return nil, fmt.Errorf("renewDeadline must be greater than zero")
	}
	if lec.RetryPeriod < 1 {
		return nil, fmt.Errorf("retryPeriod must be greater than zero")
	}

	if lec.Lock == nil {
		return nil, fmt.Errorf("Lock must not be nil.")
	}
	le := LeaderElector{
		config:  lec,
		clock:   clock.RealClock{},
		metrics: globalMetricsFactory.newLeaderMetrics(),
	}
	le.metrics.leaderOff(le.config.Name)
	return &le, nil
}
```
然后调用run

```
func (le *LeaderElector) Run(ctx context.Context) {
	defer func() {
		runtime.HandleCrash()
		le.config.Callbacks.OnStoppedLeading()
	}()
	if !le.acquire(ctx) {
		return // ctx signalled done
	}
	// 各个副本竞争锁，选举leader，如果有实例竞争成功，返回并继续执行OnStartedLeading函数，
    // 即我们上面这次的sche.Run()函数，接着继续执行renew函数不断的更新获取锁时间，让其它副本
    // 知道自己还存活着。对于没有成为leader的副本将阻塞在acquire()函数，不断重试成为leader，
   // 如果renew函数异常退出，则发信号给OnStartedLeading，告诉它也应停止工作。
	ctx, cancel := context.WithCancel(ctx)
	defer cancel()
	go le.config.Callbacks.OnStartedLeading(ctx)
	le.renew(ctx)
}
```

再调用aquire。这里就是获取分布式资源锁，如果拿到，则返回true，并继续后面的逻辑，调用回调函数onStartedLeading，这就是scheduler里面的run，里面跑scheduler的主程序，接下来继续renew。如果没有拿到，则会aquire里面继续重试，先看aquire的代码。aquire就是每隔retryPeriod时间不断重试，调用tryAcquireOrRenew 去获取锁或者更新锁的记录，renew函数也会用到该函数。

```
func (le *LeaderElector) acquire(ctx context.Context) bool {
	ctx, cancel := context.WithCancel(ctx)
	defer cancel()
	succeeded := false
	// desc形式为资源锁对象namespace/name
	desc := le.config.Lock.Describe()
	klog.Infof("attempting to acquire leader lease  %v...", desc)
 // 不断重试获取锁，重试间隔为RetryPeriod(该值是我们在scheduler的启动参数中配置的重试间隔)，
 // 如果获取锁成功则退出该JitterUntil函数，否则继续重试，直到成为leader

	wait.JitterUntil(func() {
		succeeded = le.tryAcquireOrRenew()
		le.maybeReportTransition()
		// 获取锁失败，退出继续执行JitterUntil函数
		if !succeeded {
			klog.V(4).Infof("failed to acquire lease %v", desc)
			return
		}
		
		le.config.Lock.RecordEvent("became leader")
		le.metrics.leaderOn(le.config.Name)
		klog.Infof("successfully acquired lease %v", desc)
		cancel()
	}, le.config.RetryPeriod, JitterFactor, true, ctx.Done())
	return succeeded
}
```
renew 就是不断重试，如果超时还没发renew，则表明无法renew，失去leader。如果tryAcquireOrRenew返回false，还是会继续重试。只有超时后才会失去leader。

```
func (le *LeaderElector) renew(ctx context.Context) {
	ctx, cancel := context.WithCancel(ctx)
	defer cancel()

    // 更新锁，继续占有，如果在RenewDeadline间隔内未更新成功，将失去leader
	wait.Until(func() {
		timeoutCtx, timeoutCancel := context.WithTimeout(ctx, le.config.RenewDeadline)
		defer timeoutCancel()
		err := wait.PollImmediateUntil(le.config.RetryPeriod, func() (bool, error) {
			done := make(chan bool, 1)
			go func() {
				defer close(done)
				done <- le.tryAcquireOrRenew()
			}()

			select {
			case <-timeoutCtx.Done():
				return false, fmt.Errorf("failed to tryAcquireOrRenew %s", timeoutCtx.Err())
			case result := <-done:
				return result, nil
			}
		}, timeoutCtx.Done())

		le.maybeReportTransition()
		desc := le.config.Lock.Describe()
		if err == nil {
			klog.V(5).Infof("successfully renewed lease %v", desc)
			return
		}
		le.config.Lock.RecordEvent("stopped leading")
		le.metrics.leaderOff(le.config.Name)
		klog.Infof("failed to renew lease %v: %v", desc, err)
		cancel()
	}, le.config.RetryPeriod, ctx.Done())

	// if we hold the lease, give it up
	if le.config.ReleaseOnCancel {
		le.release()
	}
}
```
tryAcquireOrRenew 非leader会努力尝试去获取leader租期，leader则会去刷新租期，成功就返回true，失败返回false。

```
func (le *LeaderElector) tryAcquireOrRenew() bool {
	now := metav1.Now()
	leaderElectionRecord := rl.LeaderElectionRecord{
		HolderIdentity:       le.config.Lock.Identity(),
		LeaseDurationSeconds: int(le.config.LeaseDuration / time.Second),
		RenewTime:            now,
		AcquireTime:          now,
	}

	// 1. 获取或者创建一条锁记录
	oldLeaderElectionRecord, err := le.config.Lock.Get()
	if err != nil {
		if !errors.IsNotFound(err) {
			klog.Errorf("error retrieving resource lock %v: %v", le.config.Lock.Describe(), err)
			return false
		}
		if err = le.config.Lock.Create(leaderElectionRecord); err != nil {
			klog.Errorf("error initially creating leader election record: %v", err)
			return false
		}
		le.observedRecord = leaderElectionRecord
		le.observedTime = le.clock.Now()
		return true
	}

	// 2. 获得锁记录， 更新身份和时间
	if !reflect.DeepEqual(le.observedRecord, *oldLeaderElectionRecord) {
		le.observedRecord = *oldLeaderElectionRecord
		le.observedTime = le.clock.Now()
	}
	if len(oldLeaderElectionRecord.HolderIdentity) > 0 &&
		le.observedTime.Add(le.config.LeaseDuration).After(now.Time) &&
		!le.IsLeader() {
		klog.V(4).Infof("lock is held by %v and has not yet expired", oldLeaderElectionRecord.HolderIdentity)
		return false
	}

	// 3. 更新锁记录的交换次数和获取时间。
	if le.IsLeader() {
		leaderElectionRecord.AcquireTime = oldLeaderElectionRecord.AcquireTime
		leaderElectionRecord.LeaderTransitions = oldLeaderElectionRecord.LeaderTransitions
	} else {
		leaderElectionRecord.LeaderTransitions = oldLeaderElectionRecord.LeaderTransitions + 1
	}

	// 更新锁本身。
	if err = le.config.Lock.Update(leaderElectionRecord); err != nil {
		klog.Errorf("Failed to update lock: %v", err)
		return false
	}
	le.observedRecord = leaderElectionRecord
	le.observedTime = le.clock.Now()
	return true
}
```

## 总结
我们看到了k8s资源的妙用。不仅可以作为一个资源，还可以作为一个分布式锁，实在妙极。虽然略有缺陷，但是在实时性要求不那么高的情况下，还是可以胜任的。





## 参考
<https://blog.csdn.net/weixin_39961559/article/details/81877056>
<http://liubin.org/blog/2018/04/28/how-to-build-controller-manager-high-available/>
<https://www.kubernetes.org.cn/1613.html>
