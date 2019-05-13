---
layout:     post
title:      "[译]Go调度III-并发"
subtitle:   "Go Scheduling III - Concurrency"
date:       2019-05-12 21:00:00
author:     "会飞的蜗牛"
header-img: "img/go-scheduler-iii.jpg"
tags:
    - Go Scheduler
    - Concurrency
    - 并发
    - 并行
---

##前言
这是三篇帮助我们更好的理解Go调度器背后的机制和语意的系列文章的第三篇。本文主要讲述并发。

三篇系列文章的地址：

1. [Scheduling In Go : Part I - OS Scheduler](https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part1.html)
2. [Scheduling In Go : Part II - Go Scheduler](https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part2.html)
3. [Scheduling In Go : Part III - Concurrency](https://www.ardanlabs.com/blog/2018/12/scheduling-in-go-part3.html)

## 简介

当我试图解决一个问题时，尤其是一个新问题时，我不会一开始就认为并发好还是不好。我会一开始寻找线性解决方案并确保这是可以工作的。然后经过可读性和技术复查之后，我会开始思考并发是否更合理和可行。有时候，并发明显是一个好的选择，而其他时候却不是。

在这个系列文章的第一篇，我解释了操作系统调度器的机制和语意，因为我认为如果你想写多线程代码，理解这些至关重要。在第二部分，我解释了Go语言调度器的语意，因为这对我们理解Go语言并发至关重要。而在这部分，我会将操作系统和Go调度器的机制和语意融合在一起，让我们更深层次的理解并发是什么以及它的本质。

此文的目的：

给你在考虑编写并发代码时候必须考虑的因素提供一些指导。

给你展示如何根据不同类型的工作负载来编写你的代码以及因此而必须作出的工程决策。

## 并发
并发意味着“乱序”执行。乱序执行一系列本来可以顺序执行的指令，并得到相同的结果。摆在面前的问题是，很明显乱序执行会增加价值。当我说价值的时候，我是指对复杂的代码增加足够的性能。乱序执行是否可行取决你的问题。

理解并发和并行的不同也非常重要。并行意味着执行同时执行两个或者更多的指令。这跟并发是完全不同的概念。并行至少需要你有两个操作系统线程和2个Goroutines，每个Goroutine在操作系统线程上独立执行指令。

图1:并发和并行
![Figure1](https://www.ardanlabs.com/images/goinggo/96_figure1.png)

在图一中，你可以看到两个逻辑处理器，每个逻辑处理器跟单独的操作系统线程绑定。你可以看到两个Goroutine（G1和G2）并行执行，同时在它们的操作系统线程上执行指令。在每一个逻辑处理器上，三个Goroutine正在轮流共享他们的操作系统线程。所有这些Goroutines都是并发执行的，共享OS线程时间片，无序地执行它们的指令。

这里有一个问题，有时候利用没有并行的并发并没有降低程序的吞吐量。更有趣的是，有时候利用并行的并发并没有给你一个更好的你本来以为可以获得的性能表现。

## 工作负载
你怎么知道乱序执行是可行的？理解问题的工作类型是处理此类问题一个很好的着手点。这里有两种工作类型的理解对于思考并发至关重要。

* CPU-Bound: 这种负载类型是Goroutines从来不会进入等待状态。这种工作是持续计算的。一个计算Pi的第N位的工作就是CPU-Bound类型。

* IO-Bound: 这种负载类型会引起Goroutines进入等待状态。这种类型的工作可能是访问网络资源或者发起系统调用或等待某件事情发生。一个需要读写文件的Goroutine就是IO-Bound，我会将同步事件（原子操作，锁）包括进来，那也会导致Goroutine进入等待状态。

对于CPU-Bound类型，你需要使用并行的并发。一个单一的操作系统/硬件线程操作多个Goroutines并不高效，因为Goroutine在这种类型的工作下并不会进入等待状态。比系统线程更多的Goroutine反而会降低执行速度，因为从操作系统上做Goroutine的上下文切换会产生额外的延迟，上下文切换相当于“暂停”了程序，因为在上下文切换期间没有工作会被完成。

对于IO-Bound类型，你并不需要使用并行的并发。一个线程能够高效的处理多个Goroutines，因为任务的需要，Goroutines很自然的在操作系统线程上切入切出，比操作系统线程更多的Goroutines能够加速任务的执行，因为将Goroutine从操作系统线程上切换的花销并不是“暂停”程序。你的应用程序很自然的暂停，而这允许一个不同的Goroutines高效地利用相同的操作系统线程，而不是让操作系统线程进入空闲状态。

那你又怎么知道一个硬件线程需要绑定几个Goroutine才能获得最佳性能呢？太少的Goroutines会让你等待更长的时间。而太多的Goroutines则会让你疲于应付上下文切换的延迟。这就是需要你自己去思考，但是已超出本文的范畴。

现在，让我们看一些代码来巩固你处理判断一个工作负载是否需要并发编码的能力，亦或者是否需要并行。

##加法
我们并不需要复杂的代码来理解这些语法。接下来这个函数就是将整数相加。

**Listing 1**

<https://play.golang.org/p/r9LdqUsEzEz>

```
36 func add(numbers []int) int {
37     var v int
38     for _, n := range numbers {
39         v += n
40     }
41     return v
42 }
```

在listing1的36行，申明了一个add函数，这个函数将整数集合求和并返回和。从37行开始，变量v并最终作为sum返回。38行，顺序遍历整数集合，并将元素相加。最终第41行，函数返回和给调用者。

问题：add函数适合用乱序来执行么？我的答案是合适。这些整数集合可以分成小的列表并并发处理。一旦所有小的集合求和完毕，然和sum集合可以添加在一起，并产生最终的答案。

然而，这里还有一个问题。我们应该将整数集合分成多少个小的集合才能得到最佳性能呢？为了回答这个问题，你应该知道add函数的工作类型。add函数是CPU-Bound类型，因为算法是纯粹的数学加法，并没有什么会让它自然进入等待状态。这意味着一个操作系统绑定一个Goroutine就能获得最佳的性能。

Listing2是并发版本的add

*注意：这里有几种方法和选项可以实现代码的并发版本。不要被我这种特殊的版本卡住，如果你有可读性更高的代码可以达到同样的效果甚至更好的话，如果你愿意分享的话，我将非常开心。*

**Listing 2**
<https://play.golang.org/p/r9LdqUsEzEz>

```
44 func addConcurrent(goroutines int, numbers []int) int {
45     var v int64
46     totalNumbers := len(numbers)
47     lastGoroutine := goroutines - 1
48     stride := totalNumbers / goroutines
49
50     var wg sync.WaitGroup
51     wg.Add(goroutines)
52
53     for g := 0; g < goroutines; g++ {
54         go func(g int) {
55             start := g * stride
56             end := start + stride
57             if g == lastGoroutine {
58                 end = totalNumbers
59             }
60
61             var lv int
62             for _, n := range numbers[start:end] {
63                 lv += n
64             }
65
66             atomic.AddInt64(&v, int64(lv))
67             wg.Done()
68         }(g)
69     }
70
71     wg.Wait()
72
73     return int(v)
74 }
```
在Listing2中，addConcurrent函数是add函数的并发版本。并发版本用了26行代码来实现5行顺序版本。代码很多，所以我只会着重讲解几行重要的代码。

48行: 每个Goroutine只会得到它们自己唯一的更小的数组列表。列表的尺寸通过集合尺寸除以Goroutines的个数来计算。

53行：创建Goroutines池来进行计算。

57-59行：最后的Goroutine会计算剩下的列表，可能会被其他的Goroutine里列表更大。

66行：最终，这些小列表的和都计算进最终的变量sum里。

并发版本比顺序版本复杂的多，但是这个复杂度值吗？最佳答案是来做一个基准测试。这个基准测试中，使用了1000万数字的集合，并且关闭了垃圾回收。

**Listing3**

```
func BenchmarkSequential(b *testing.B) {
    for i := 0; i < b.N; i++ {
        add(numbers)
    }
}

func BenchmarkConcurrent(b *testing.B) {
    for i := 0; i < b.N; i++ {
        addConcurrent(runtime.NumCPU(), numbers)
    }
}

```

列表3展示了基准函数。这里是单操作系统线程多Goroutine的结果。顺序版本使用1个Goroutine，而并发版本使用了runtime.NumCPU或者8个Goroutines（本机）。这个并发版本并没有使用并行。

**Listing4**

```
10 Million Numbers using 8 goroutines with 1 core
2.9 GHz Intel 4 Core i7
Concurrency WITHOUT Parallelism
-----------------------------------------------------------------------------
$ GOGC=off go test -cpu 1 -run none -bench . -benchtime 3s
goos: darwin
goarch: amd64
pkg: github.com/ardanlabs/gotraining/topics/go/testing/benchmarks/cpu-bound
BenchmarkSequential      	    1000	   5720764 ns/op : ~10% Faster
BenchmarkConcurrent      	    1000	   6387344 ns/op
BenchmarkSequentialAgain 	    1000	   5614666 ns/op : ~13% Faster
BenchmarkConcurrentAgain 	    1000	   6482612 ns/op
```

*在本地机器上跑基准测试是很复杂的。会有很多的因素会导致结果并不是很精确。确保你的机器尽可能的空闲，然后多跑几次基准测试。你要确保结果的一致性。跑两次基准测试会得到一致性更好的结果*

基准测试的结果是顺序版本比并发版本大约快10%～13%，在没有并行的时候。这个结果正是跟期望值一致。因为Goroutine的管理和上下文切换需要花费时间。

下文是单核单Goroutine的顺序版本，并发版本利用多核，多个Goroutine的结果。这是使用了并行的并发。

**Listing5**
```
10 Million Numbers using 8 goroutines with 8 cores
2.9 GHz Intel 4 Core i7
Concurrency WITH Parallelism
-----------------------------------------------------------------------------
$ GOGC=off go test -cpu 8 -run none -bench . -benchtime 3s
goos: darwin
goarch: amd64
pkg: github.com/ardanlabs/gotraining/topics/go/testing/benchmarks/cpu-bound
BenchmarkSequential-8        	    1000	   5910799 ns/op
BenchmarkConcurrent-8        	    2000	   3362643 ns/op : ~43% Faster
BenchmarkSequentialAgain-8   	    1000	   5933444 ns/op
BenchmarkConcurrentAgain-8   	    2000	   3477253 ns/op : ~41% Faster
```

Listing5的基准测试显示并发版本比顺序版本快大约41～43%。这跟我们的预期也是一致的。因为多Goroutine并行执行，8个Goroutine同时执行他们的并发工作。

## 排序

理解不是所有的CPU-Bound类型都是适合并发的至关重要。把任务分成8份然后又把所有的结果合并的开销是非常昂贵的。一个例子是一个叫Bubble的排序可以看到这个问题所在。看如下的Go版本的Bubble排序的代码实现。

**Listing 6**

<https://play.golang.org/p/S0Us1wYBqG6>

```
01 package main
02
03 import "fmt"
04
05 func bubbleSort(numbers []int) {
06     n := len(numbers)
07     for i := 0; i < n; i++ {
08         if !sweep(numbers, i) {
09             return
10         }
11     }
12 }
13
14 func sweep(numbers []int, currentPass int) bool {
15     var idx int
16     idxNext := idx + 1
17     n := len(numbers)
18     var swap bool
19
20     for idxNext < (n - currentPass) {
21         a := numbers[idx]
22         b := numbers[idxNext]
23         if a > b {
24             numbers[idx] = b
25             numbers[idxNext] = a
26             swap = true
27         }
28         idx++
29         idxNext = idx + 1
30     }
31     return swap
32 }
33
34 func main() {
35     org := []int{1, 3, 2, 4, 8, 6, 7, 2, 3, 0}
36     fmt.Println(org)
37
38     bubbleSort(org)
39     fmt.Println(org)
40 }
```

在listing6里，Go实现的Bubble排序的例子。这个排序算法遍历集合里的所有元素。在数组有序之前，会多次进行排序和比较。

问题：bubbleSort适合乱序执行吗？我想答案是不适合。这个数组集合被分成多个小份，每份分别并发的进行排序。然而，在所有的并发工作完成之后，并没有一个有效的方法来将这些小份合并到一起。下文是一个并发Bubble排序的例子。

**Listing8**

```
01 func bubbleSortConcurrent(goroutines int, numbers []int) {
02     totalNumbers := len(numbers)
03     lastGoroutine := goroutines - 1
04     stride := totalNumbers / goroutines
05
06     var wg sync.WaitGroup
07     wg.Add(goroutines)
08
09     for g := 0; g < goroutines; g++ {
10         go func(g int) {
11             start := g * stride
12             end := start + stride
13             if g == lastGoroutine {
14                 end = totalNumbers
15             }
16
17             bubbleSort(numbers[start:end])
18             wg.Done()
19         }(g)
20     }
21
22     wg.Wait()
23
24     // Ugh, we have to sort the entire list again.
25     bubbleSort(numbers)
26 }
```

**Listing 9**

```
Before:
  25 51 15 57 87 10 10 85 90 32 98 53
  91 82 84 97 67 37 71 94 26  2 81 79
  66 70 93 86 19 81 52 75 85 10 87 49

After:
  10 10 15 25 32 51 53 57 85 87 90 98
   2 26 37 67 71 79 81 82 84 91 94 97
  10 19 49 52 66 70 75 81 85 86 87 93
```
由于冒泡排序的本质是依次扫描，第 25 行对 bubbleSort 的调用将掩盖使用并发解决问题带来的潜在收益。结论是：在冒泡排序中，使用并发不会带来性能提升。

## 读文件
两个CPU-Bound类型的工作负载已经在上文呈现了，但是IO-Bound类型的呢？Goroutines在等待状态中自然切换的语意会有什么不同吗？来看看IO-Bound类型的工作读写文件的表现并做一个性能测试。

第一版是一个顺序执行的函数，我们叫做find。

**Listing10**

<https://play.golang.org/p/8gFe5F8zweN>

```
42 func find(topic string, docs []string) int {
43     var found int
44     for _, doc := range docs {
45         items, err := read(doc)
46         if err != nil {
47             continue
48         }
49         for _, item := range items {
50             if strings.Contains(item.Description, topic) {
51                 found++
52             }
53         }
54     }
55     return found
56 }
```

在Listing10中，你可以看到find函数的顺序版本。在第43行中，申明了一个found变量去保存特定话题在文档中被找到的次数。然后在44行中，所有的文档都被遍历了一次，每个文档都通过read函数读。最后，在49-53行中，strings包中的Contains函数用来检查话题是否在文档里。如果能找到，则将变量found递增1.

**Listing 11**

<https://play.golang.org/p/8gFe5F8zweN>

```
33 func read(doc string) ([]item, error) {
34     time.Sleep(time.Millisecond) // Simulate blocking disk read.
35     var d document
36     if err := xml.Unmarshal([]byte(file), &d); err != nil {
37         return nil, err
38     }
39     return d.Channel.Items, nil
40 }
```
read函数由一个Sleep函数睡眠1分钟开始。这次调用主要是模拟实际从磁盘读写文件产生的延迟。延迟的一致性对我们精确测试find函数顺序版本的性能很重要。在35-39行，模拟xml文档存储函数，并将文件反序列化进一个数据结构。最终，所有这些结果都会返回给39行的调用者。

接下来我们看看并发版本。
*注意：这里有几种方法和选项可以实现代码的并发版本。不要被我这种特殊的版本卡住，如果你有可读性更高的代码可以达到同样的效果甚至更好的话，如果你愿意分享的话，我将非常开心。*

**Listing 12**

<https://play.golang.org/p/8gFe5F8zweN>

```
58 func findConcurrent(goroutines int, topic string, docs []string) int {
59     var found int64
60
61     ch := make(chan string, len(docs))
62     for _, doc := range docs {
63         ch <- doc
64     }
65     close(ch)
66
67     var wg sync.WaitGroup
68     wg.Add(goroutines)
69
70     for g := 0; g < goroutines; g++ {
71         go func() {
72             var lFound int64
73             for doc := range ch {
74                 items, err := read(doc)
75                 if err != nil {
76                     continue
77                 }
78                 for _, item := range items {
79                     if strings.Contains(item.Description, topic) {
80                         lFound++
81                     }
82                 }
83             }
84             atomic.AddInt64(&found, lFound)
85             wg.Done()
86         }()
87     }
88
89     wg.Wait()
90
91     return int(found)
92 }
```

在Listing12中，findConcurrent函数是find函数的并发版本。相对于顺序版本的13行代码，并发版本使用了30行代码。实现并发版本的关键是控制协程的个数，我是通过通道来控制协程的并发，用一个通道来满足线程池资源的需要。

这里有几个代码点需要我着重介绍下，有助于我们理解。

61-64行：通道被创建，并用来操作所有的文档

65行：通道被关闭，所以当所有文档都处理完了的话，线程池会自然的终止。

70行：线程池被创建。

73-83行：池里的每一个协称都会从通道里接收一个文档，并把文档读进内存里，检查内容，是否包含topic。如果有的话，局部变量found就会自增1.

84行：每个独立线程的个数和相加就是最终的结果。

这个并发版本比顺序版本要复杂的多。但是复杂度是否值呢？对于这些，我用1000个文档进行了benchmark，同时关闭了垃圾回收。顺序版本使用find函数，而并发版本使用findConcurrent函数。

**Listing 13**

```
func BenchmarkSequential(b *testing.B) {
    for i := 0; i < b.N; i++ {
        find("test", docs)
    }
}

func BenchmarkConcurrent(b *testing.B) {
    for i := 0; i < b.N; i++ {
        findConcurrent(runtime.NumCPU(), "test", docs)
    }
}
```
Listing13是benchmark的结果。这里的结果是基于单个操作系统线程被多Goroutines使用的情况。这种情况下就不存在并行了。

**Listing 14**

```
10 Thousand Documents using 8 goroutines with 1 core
2.9 GHz Intel 4 Core i7
Concurrency WITHOUT Parallelism
-----------------------------------------------------------------------------
$ GOGC=off go test -cpu 1 -run none -bench . -benchtime 3s
goos: darwin
goarch: amd64
pkg: github.com/ardanlabs/gotraining/topics/go/testing/benchmarks/io-bound
BenchmarkSequential      	       3	1483458120 ns/op
BenchmarkConcurrent      	      20	 188941855 ns/op : ~87% Faster
BenchmarkSequentialAgain 	       2	1502682536 ns/op
BenchmarkConcurrentAgain 	      20	 184037843 ns/op : ~88% Faster
```
listing14的benchmark结果显示了并发版本比顺序版本快87%～88%，仅单个操作系统线程的工作情况。这跟我们预期基本一致。Goroutine的自然切换对IO-Bound类型有帮助，可以更高效的工作。

下文是使用并行的并发版本的基准测试。

**Listing 15**

```
10 Thousand Documents using 8 goroutines with 1 core
2.9 GHz Intel 4 Core i7
Concurrency WITH Parallelism
-----------------------------------------------------------------------------
$ GOGC=off go test -run none -bench . -benchtime 3s
goos: darwin
goarch: amd64
pkg: github.com/ardanlabs/gotraining/topics/go/testing/benchmarks/io-bound
BenchmarkSequential-8        	       3	1490947198 ns/op
BenchmarkConcurrent-8        	      20	 187382200 ns/op : ~88% Faster
BenchmarkSequentialAgain-8   	       3	1416126029 ns/op
BenchmarkConcurrentAgain-8   	      20	 185965460 ns/op : ~87% Faster
```

Listing15显示了引入额外的操作系统线程并不能带来更好的性能。

## 总结
这片文章的目的是在你需要考虑和决定引入并发编程的时候提供了一下知道。我尝试提供不同的算法类型和工作负载来帮助你理解不同的工程决策下，并考虑到语法和语意的不同。

你可以很清楚的看到并行并发对IO-Bound类型的工作负载并不会有性能飞跃。这跟CPU-Bound类型正好相反。而当谈到计算密集型例如排序算法之类的，并发只会增加复杂性，而对性能毫无益处。你的工作负载是否适合并发这点至关重要。其次就是根据工作负载的类型来编写正确的代码。
