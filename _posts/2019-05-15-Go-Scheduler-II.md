---
layout:     post
title:      "[译]Go调度II - Go调度器"
subtitle:   "Go Scheduling II - Go Scheduler"
date:       2019-05-11 21:00:00
author:     "会飞的蜗牛"
header-img: "img/go-scheduler-ii.jpg"
tags:
    - Go调度器
    - Go Scheduler
---



## 前言
这是三篇文章中的第一篇来帮助我们更好地理解Go调度器背后的机制和语法，所以本文将着重讲述GO语言调度器。

三篇系列文章的地址：

1. [Scheduling In Go : Part I - OS Scheduler](https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part1.html)
2. [Scheduling In Go : Part II - Go Scheduler](https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part2.html)
3. [Scheduling In Go : Part III - Concurrency](https://www.ardanlabs.com/blog/2018/12/scheduling-in-go-part3.html)

## 简介
在调度器系列文章的第一章中，我讲述了操作系统调度器的部分知识，因为我认为那对我们理解Go调度器背后的语法至关重要。在这篇文章中，我将在语法层面讲述解释Go调度器如何工作以及更高层次的行为。Go调度器是个非常复杂的系统，小的细枝末节对我们来说并不重要。重要的是有一个很好的模型让系统正常工作。这将有助你做出更好的工程决策。

## 开启你的程序
当Go语言程序启动时，它会为主机的虚拟核心提供一个逻辑处理器。如果你的处理器拥有多个硬件线程（超线程），那么每个硬件线程在Go程序里都将作为一个虚拟核心。为了更好的理解这些，请看我的MacBook Pro的系统报告：

图1

![图1](https://www.ardanlabs.com/images/goinggo/94_figure1.png)

你可以看到这里是拥有4核心的单处理器。这里没有展示的是每个核心上有几个硬件线程。因特尔I7处理器有超线程，每核拥有2个硬件线程。这将告诉 Go 程序有 8 个虚拟核心可用于并行执行系统线程。

通过一个程序来验证一下：

```
package main

import (
    "fmt"
    "runtime"
)

func main() {

    // NumCPU 返回当前可用的逻辑处理核心的数量
    fmt.Println(runtime.NumCPU())
}
```

当我在本地机器跑这个程序时，NumCPU()返回的结果一直是8。这样，这个主机上跑的任何Go程序都将分配8个P.

每个 P 会分配一个操作系统线程（"M"），M表示机器。这个线程还是被内核控制，内核负责将线程放入核心上执行，正如前文所示。这意味着当我在机器上跑一个Go程序，我有8个线程可执行任务，每个线程都和一个P绑定。

每个Go程序都会分配一个初始Go协程("G")，它是Go程序的执行路径。一个Goroutine本质上是一个协程，因为是Go语言，所以把"C"变成"G"，就是Goroutine。你可以认为Goroutine就是一个应用级别的线程并且它们跟操作系统线程有很多的类似之处。正如操作系统线程是在核心上进行上下文切换，Goroutine是在M上进行上下文切换。

最后一个重点是运行队列。Go语言调度器中有两个不同的运行队列：全局队列(GRQ)和本地队列(LRQ)。每个P会给定一个LRQ，用于管理分配给在P上执行的 Goroutines。这些Goroutine轮流在跟P绑定的M上做上下文切换。而全局队列保存的则是还没有分配给P的Goroutines。有一个进程负责将Goroutine从全局队列移到本地队列中，这点我们后面再讨论。

下面图示展示了它们之间的关系：

![](https://www.ardanlabs.com/images/goinggo/94_figure2.png)

## 协作式调度器
正如我们在第一篇文章中所讨论的，操作系统调度器是一个抢占式调度器。本质来说任何时刻你都无法预测调度器将要做什么。内核正在作出决策，而一切都是不确定的。应用跑在操作系统的上层，它无法控制内核调度行为，除非采取一些类似原子指令或者锁之类的同步操作。

Go语言调度器是Go运行时的一部分，而Go运行时则内嵌在你的应用程序中。这意味着Go调度器运行在用户态，在内核之上。目前，Go语言调度器的实现并不是一个抢占式调度器，而是一个协作式调度器。作为一个协作式调度器，意味着调度器需要明确定义用户空间事件，这些事件发生在代码中的安全点，以做出调度决策。

而更聪明的是关于Go语言协作式调度器让人看起来感觉是抢占式的。你无法预测Go调度器将要作什么。这是因为决策调度没有掌握在开发人员手中，而是在Go运行时中。把Go语言调度器看作是一个抢占式调度器是很重要的，由于调度程序是非确定性的，因此这并不是一件容易的事情。 

## Goroutine状态
正如线程一样，Goroutines也有相同的三个高级状态，它们就标识了Go语言调度器可以给任何Goroutines绑定角色。一个Goroutine可以是一下三种状态之一：Waiting，Runnable，Executing。

**Waiting**：这表示Goroutine是暂停的，正在等待某些资源以求可以继续执行。这可能是以下原因之一，例如系统调用，同步调用（原子和锁操作）。这些延迟是程序性能差的根本原因。

**Runnable**：这意味着Goroutine想要拥有M的时间片以便可以执行分配给它的指令。如果你有很多的Goroutines想要时间，那么Goroutines需要等待更长的时间来获取时间片。所以，单个Goroutine的时间就变少了因为很多的Goroutines在竞争M的时间。这种延迟也是应用性能差的一个原因。

**Executing**：这意味着Goroutine跟M绑定且正在执行它的指令。应用程序相关的工作正在被完成。这正是大家想要的。

## 上下文切换
Go调度器需要明确定义的用户空间事件，这些事件发生在要进行上下文切换的代码中的一个安全点。这些事件和安全点都列在函数调用里。函数调用对于Go调度器的健康状态至关重要。如今（在 Go 1.11或者之前的版本中），如果你运行没有函数调用的紧凑循环，这将导致调度和垃圾回收的延迟。让函数调用发生在合理的时间帧内至关重要。

请注意：最新的1.12里面接收了一个提议，允许非协作抢占技术，允许调度器对紧凑循环的抢占式调度。

在你的Go语言程序里，这里有4个等级的事件允许调度器作出调度决策。但这并不表示有任意事件产生，调度就会发生。只是说调度器有机会进行调度。


* 使用go关键字
* 垃圾回收
* 系统调用
* 同步和编排

### 使用go关键字
你通过关键字go来创建Goroutines。一旦一个新的Goroutine被创建，这就给了调度器一个作出调度决策的机会。

### 垃圾回收
由于垃圾回收有自己的协程集，这些协程也需要绑定M运行。这就导致GC创造了许多的调度混乱。然而，调度器非常智能，知道一个协程正在干什么，所以能做出正确的决策。一个智能的决策就是把这些还没有分配到堆上的协程在GC期间从上下文切换中切换出来，禁止分配内存。当GC时，调度器需要做出非常多的调度决策。

### 系统调用
当一个Goroutine产生一个系统调用，这会导致Goroutine阻塞M。有时候，调度器可以将该线程通过上下文切换跟M分离，在调度一个新的Goroutine到相同的M上。然而，有时候，需要一个新的M来执行P队列中的Goroutines。在下一个章节中，我们将对这一点做进一步的解释。

### 同步和编排
一个原子，锁和通道操作将导致Goroutine阻塞，调度器会通过上下文切换一个新的Goroutine来运行。一旦之前的Goroutine可以再次执行了，它会重新入队并且最终重新切回到M上来执行。

## 异步系统调用
当你的操作系统有能力异步地处理系统调用，可以使用网络轮询器来更高效地系统调用。在不同的操作系统中有不同的实现，例如MacOS的kqueue，Linux的epoll或者Windows的iocp。

如今，很多的操作系统都能异步处理网络系统调用。网络轮询器名字的由来是因为它最初是用来处理网络操作的。通过使用网络轮询器来处理网络系统调用，当这些系统调用发生的时候，调度器能够阻止Goroutine阻塞M。这使得M可以继续执行P队列中的其他Goroutines而没必要创建一个新的M。这能很好的减轻OS的调度压力。

最好的方式就是来看一个例子来解释这是怎么工作的。

![figure3](https://www.ardanlabs.com/images/goinggo/94_figure3.png)

G1正在M上执行，LRQ中还有3个协程在等待M的时间片。网络轮询器目前还是空闲的。

![figure4](https://www.ardanlabs.com/images/goinggo/94_figure4.png)

在上图中，G1想要进行网络系统调用，所以G1被移到网络轮询器上，异步的网络调用正在进行。一旦G1被移动到网络轮询器中，那么M就空下来执行LRQ中的其他Goroutine。在这个例子中，G2就被切换到M上执行了。

![](https://www.ardanlabs.com/images/goinggo/94_figure5.png)

在上图中，异步网络调用由网络轮询器完成，然后G1就会被移到了P的LRQ中，一旦Goroutine重新切换到M上，它负责的Go相关代码的工作就又可以被执行了。这里最大的好处就是，执行网络系统调用不需要额外的M了。网络轮询器有一个操作系统线程，它时刻处理一个有效的事件循环。

## 同步系统调用

当一个Goroutine想要执行一个不能异步执行的系统调用会发生什么呢？这种情况下，就不能使用网络轮询器了，产生系统调用的Goroutine将会阻塞M，这很不幸，但是没有办法阻止事情的发生。一个不能异步系统调用的例子就是基于文件的系统调用。如果你在使用CGO，可能还会有其他的例如调用C函数也会阻塞M的场景。

注意：Windows操作系统有能力异步处理文件系统调用。原理上来讲，当程序运行在Windows上时，网络轮询器会被使用。

让我们来看看会阻塞M的同步系统调用（例如文件I/O）发生时会是什么情况。

图6
![Figure6](https://www.ardanlabs.com/images/goinggo/94_figure6.png)

上图，G1会产生一个系统调用，并阻塞M。

图7
![Figure 7](https://www.ardanlabs.com/images/goinggo/94_figure7.png)

图7中，调度器能够发现G1导致M被阻塞。此时，调度器把M1和P分离，M1和G1还是绑定的。调度器会把M2调过来跟P绑定。此时，G2会从LRQ中选出来调度到M2上去执行。如果之前的交换有多余的M存在，这种转换比去创造一个新的M要快。

图8

![Figure 8](https://www.ardanlabs.com/images/goinggo/94_figure8.png)

图8中，G1产生的阻塞的系统调用完成。此时，G1又回到P的LRQ中。M1就会被放置一旁，以备将来这种情况再次发生。

## 任务窃取
调度器的另一个方面是任务窃取。这在以下几个方面有助于保持调度的高效性。首先，你最不希望看到的就是M进入等待状态，一旦这种情况发生，操作系统将通过上下文切换将M从核心中切换出去。这意味着P将无事可做了，即使这里还有Runnable状态的Goroutine。其次，任务窃取也是有助于多个P之间的任务分布更平衡，从而使得程序更高效。

让我们来看个例子。

图9
![Figure9](https://www.ardanlabs.com/images/goinggo/94_figure9.png)

图10
![Figure10](https://www.ardanlabs.com/images/goinggo/94_figure10.png)

如你所见，P1的任务完成了，但是GRQ和P2的LRQ都还有Goroutine可运行。这是P2就会去窃取任务。

窃取的规则在这里定义了。 <https://golang.org/src/runtime/proc.go>

```
if gp == nil {
        // 1/61的概率检查一下全局可运行队列，以确保公平。否则，两个 goroutine 就可以通过不断地相互替换来完全占据本地运行队列。
        if _g_.m.p.ptr().schedtick%61 == 0 && sched.runqsize > 0 {
            lock(&sched.lock)
            gp = globrunqget(_g_.m.p.ptr(), 1)
            unlock(&sched.lock)
        }
    }
    if gp == nil {
        gp, inheritTime = runqget(_g_.m.p.ptr())
        if gp != nil && _g_.m.spinning {
            throw("schedule: spinning with local work")
        }
    }
    if gp == nil {
        gp, inheritTime = findrunnable()
    }
```

根据规则，P1将窃取P2中一半的 Goroutines，窃取完成后的样子如下：

图11

![Figure11](https://www.ardanlabs.com/images/goinggo/94_figure11.png)

图11中，P2一半的任务将被P1窃取。

如果P2也完成了，但是同时P1也什么都没有呢？

图12
![Figure12](https://www.ardanlabs.com/images/goinggo/94_figure12.png)

图12中，P2也完成了所有的工作，需要偷取一些任务。首先，它会去看P1的LRQ队列，但是一无所获，然后，它就会去检查GRQ。就找到了G9.

图13

![Figure13](https://www.ardanlabs.com/images/goinggo/94_figure13.png)

在图13中，P2从GRQ中偷了G9，开始工作。以上任务窃取的好处在于它使M不会闲着。在窃取任务时，M是自旋的。这种自旋还有其他的好处，可以参考 [work-stealing](https://rakyll.org/scheduler/) 。

## 实例
有了相应的语意和机制，我想向你展示如何将所有这些融合在一起允许调度器执行更多的工作。设想一个用 C 编写的多线程应用程序，其中程序管理两个操作系统线程，这两个线程相互传递消息。

![Figure14](https://www.ardanlabs.com/images/goinggo/94_figure14.png)

下面有两个线程，线程 T1 在内核 C1 上进行上下文切换，并且正在运行中，这允许 T1 将其消息发送到 T2。

> 注意，消息是如何传递的并不重要，重要的是当调度进行时线程的状态。

图15
![Figure15](https://www.ardanlabs.com/images/goinggo/94_figure15.png)

在图15中，一旦T1完成了消息发送，它就需要等待消息相应。这就会导致T1被上下文切换从C1切换下来并进入等待状态。一旦T2收到了消息，他就进入了可执行状态。操作系统调度器进行上下文切换，并将T2放到核心上执行，就在C2上执行。然后线程2处理消息并发送消息会T1。

图16

![Figure16](https://www.ardanlabs.com/images/goinggo/94_figure16.png)

图16中，线程上下文切换再次发生，因为T2收到了T1传来的信息。现在T2从执行中切换到了等待状态而T1从等待状态切换到可执行状态并最终回到了执行中的状态。这允许它执行程序并再次发回新的消息。

所有这些上下文切换和状态改变都需要消耗时间片，这限制了任务完成的速度。每次上下文切换大约需要花费50ns的时间，理想状态下硬件每纳秒可以执行12个指令，所以一次上下文切换差不多有600个指令未执行。由于这些线程还需要在不同的核心上跳转，所以存在缓存失效而导致的额外延迟会更高。

让我们来看一个相同的例子，但是使用Goroutine和Go调度器来代替线程和操作系统调度器。

图17

![Figure17](https://www.ardanlabs.com/images/goinggo/94_figure17.png)

图17中，调度程序中有两个Goroutines来回传递消息。G1在M1上做上下文切换，而M1正好核心1上运行，这样G1就可以正常执行任务。而当前任务就是G1给G2发送消息。

图18
![Figure18](https://www.ardanlabs.com/images/goinggo/94_figure18.png)

图18中，一旦G1完成消息发送就进入了等待相应的状态。这会导致G1从M1上换下而进入等待状态。一旦G2收到了消息通知，就进入了可执行状态。现在Go调度器就可以通过一次上下文切换将G2在M1上执行，还是核心1上运行。然后G2处理这个消息并发送消息会G1。

图19

![Figure19](https://www.ardanlabs.com/images/goinggo/94_figure19.png)

在图19中，上下文在此发生，消息从G2发出，G1接收。现在G2从执行中转换到了等待状态，而G1从等待状态切换到了可执行状态并最终进入执行中状态，这样G1就可以处理并返回新的消息。

事情表面上看起来并没有任何不同。不管是Goroutine还是Thread，上下文切换和状态改变都同样发生。是的，乍看一眼，Goroutines和Threads的区别并不是很明显。

在使用Goroutine的案例中，进程的整个状态中操作系统线程和核心都是同一个。这意味着，从操作系统的角度来说，操作系统线程从来没进入过等待状态；一次也没有。结果就是，当我们使用Goroutine时，因为线程上下文切换而导致指令丢失的事件从未发生。

本质来讲，Go把操作系统级别的IO/阻塞型工作转换成CPU-Bound类型了。既然所有的上下文切换都发生在应用级别了，我们就不会丢失因为发生线程上下文切换而导致的600个指针。调度器
有助于获得更高的缓存效率和NUMA。这就是为甚么我们不需要比核心更多的线程。在Go语言里，相同的时间里可以完成更多的工作，因为Go调度器尝试用更少的线程而每个线程做更多工作，这对减少OS和硬件的负载很有帮助。

## 总结

如何设计这样复杂的系统，需要考虑操作系统和硬件如何工作，Go调度器做的非常棒。把操作系统几倍的IO/Blocking类型的工作转换为CPU-Bound类型的能力，是我们充分利用CPU的能力来获取最大的收益。这就是为什么我们不需要每个核心上更多的线程。你可以合理的期望把所有的工作（CPU和IO/Blocking-bound）都在一核一线程上完成。对于网络应用程序和其他不会阻塞操作系统线程的系统调用的应用程序来说，这样做是可能的。

作为一个开发者，你还当然需要知道你的程序做了什么工作。你不能创建无限多的Goroutines来期望获得很好的性能。少就是多，但是理解了Go调度器的语意之后，你就能做更好的工程决策。在接下来的文章中，我将探讨以保守的方式利用并发性以获得更好的性能，同时平衡可能需要增加到代码中的复杂性

##原文
<https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part2.html>