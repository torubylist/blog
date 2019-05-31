---
layout:     post
title:      "【译】Go内存模型"
subtitle:   " \"Go memory model\""
date:       2019-05-31 20:00:00
author:     "会飞的蜗牛"
header-img: "img/go_memory_model.jpg"
tags:
    - Go
    - Memory Model
---

> 翻译主要是为了更好的掌握文中的内容，如有不妥之处或者想我交流，欢迎随时给我发邮件，谢谢。yongpingzhao1#gmail.com

## 简介
Go内存模型指定了一个条件，在该条件下，可以保证在一个goroutine中读取变量，以观察通过不同goroutine写入同一变量而产生的值。

## 建议
修改由多个goroutine同时访问的数据的程序必须序列化此类访问。

要序列化访问，请使用通道操作或其他同步原语（例如sync和sync/atomic包中的那些）来保护数据。

## Happens Before

在单个goroutine里面，读写表现跟代码的顺序是一致的。虽然编译器和CPU由于开启了优化功能可能调整读写操作的顺序，但是这个调整是不会影响这个程序的执行正确性。

```
a := 1//1
b := 2//2
c := a + b //3
...
```
这里c的值总是3.跟a，b的执行顺序并无关系。

但是在多线程并发的情况下，因为这个重排，可观察到的执行顺序可能跟其他goroutine感知到的并不一致。例如，goroutine1执行a=1；b=2；另一个goroutine看到的可能是先给b赋值，再给a赋值。所以下图输出的值并不确定，可能1，也可能是0，也可能没有输出，因为a，b的值执行顺序可能跟代码并不一致。这是因为编译器或者CPU可能会对goroutine A中的指令做重排序，可能先执行了代码（2），然后在执行了代码（1）。假设当goroutine A执行代码（2）后，调度器调度了goroutine B执行，则goroutine B这时候会输出0。


```
//变量b初始化为0
 var b int

 //goroutine A
 go func() {
     a := 1     //1
     b := 2     //2
     c := a + b //3
 }()
 
 //goroutine B
 go func() {
     if 2 == b { //4
     		fmt.Println(a)//5
     }
 }()

```

为了保证读写的正确性，我们定义了happens before，即在go程序中定义了多个内存操作执行的一种偏序关系。如果事件e1发生在e2之前，我们就说e2发生在e1之后。同样的，如果e1没有发生在e2之前，也没有发生在e2之后，那么我们就说e1，e2并发执行。

在单个goroutine里，happens-before顺序就是代码的顺序表达。

happens before原则指出在单一goroutine 中当满足下面条件时候，对一个变量的写操作w1对读操作r1可见：

* **r操作没有在w操作之前**
* **在读操作r1之前，写操作w1之后没有其他的写操作w2对变量进行了修改**

在一个goroutine里面，不存在并发，所以对变量的读操作r1总是可以观察到最近的一个写操作w1，但是在多goroutine下则需要满足下面条件才能保证写操作w1对读操作r1可见：

* **w发生在r之前**
* **其他任何对v的写操作要么发生在w之前，要么发生在r之后。**

这对条件比之前的条件更苛刻。需要这里没有其他的w操作跟w和r并发执行。

当多个goroutine访问共享变量v，则必须使用同步时间来建立happens-before约束，确保读操作看的是预期的写。

变量v的零值初始化在内存模型中对应着一次写操作。

当读取或写一个类型大小比机器字长大的变量的值时候表现为是对多个机器字的多次读取，这个行为是未知的。


## 同步
### 初始化

程序的初始化发生在一个goroutine里面，但是这个goroutine可以创造其他goroutine，这些goroutine并发执行。

如果一个包p引入了包q，q的初始化函数happens before p的初始化。

函数main.main发生在所有init函数完成之后。

### Goroutine创建
新建goroutine发生在goroutine执行之前。这个意思是新建完goroutine，并不会直接进入goroutine执行，而是继续执行主goroutine。

```
var a string

func f() {
	print(a)
}

func hello() {
	a = "hello, world"
	go f()
}
```

调用hello会在某个时间点打印出“hello world”（可能在hello返回之后）

### Goroutine销毁
goroutine的结束并不确保happens before任何事件。举例来说：

```
var a string

func hello() {
	go func() { a = "hello" }()
	print(a)
}
```

对a的赋值并没有在任何同步事件之后，所以并不能确保它会被其他goroutine观察到。事实上，先进的编译器可能会删除整个go声明。

如果goroutine的作用必须被另一个goroutine观察到，必须使用同步机制，例如锁，管道等通信方式建立相对顺序。

### 通道通信
通道通信是goroutine之间同步的主要方式。对某个通道发送消息对应那个通道的消息接收，通常在别的goroutine中。

#### 有缓冲的通道
在有缓冲通道中通过向通道写入一个数据总是 happen before 这个数据被从通道中读取完成，这个happen before规则使多个goroutine中对共享变量的并发访问变成了可预见的串行化操作。

```
var c = make(chan int, 10)
var a string

func f() {
	a = "hello, world"
	c <- 0
}

func main() {
	go f()
	<-c
	print(a)
}
```

可以确保输出“hello world”。对a的写入发生在给c发送消息之前，而它有发生在c的接收消息之前。

另外关闭通道的操作 happen before 从通道接收0值（关闭通道后会向通道发送一个0值），修改上面代码：

```
var c = make(chan int, 10)
var a string

func f() {
	a = "hello, world"
	close(c)
}

func main() {
	go f()
	print(<-c)
	print(a)
}

```

从无缓冲的通道接收数据总是发生在通道的发送完成之前。

```
var c = make(chan int)
var a string

func f() {
	a = "hello, world"
	<-c
}
func main() {
	go f()
	c <- 0
	print(a)
}
```

所以这里可以确保可以输出“hello world”。

如果是缓冲通道（`c=make(chan int, 1)`），那么这里就不能保证会输出“hello world”。

```
var c = make(chan int， 1)
var a string

func f() {
	a = "hello, world"
	<-c
}
func main() {
	go f()
	c <- 0
	print(a)
}
```

从容量为C的通道接收第K个元素 happen before 向通道第k+C次写入完成，比如从容量为1的通道接收第3个元素 happen before 向通道第3+1次写入完成。

这个规则通用化了之前的缓冲通道的规则。它允许计数信号量由缓冲信道建模：信道中的项目数量对应于活跃的使用次数，信道容量对应于最大同时使用次数，发送项目获取信号量，以及接收项目会释放信号量。这是限制并发的习惯用法。

下面这个程序的代码main goroutine里面为work列表里面的每个方法的执行开启了一个单独的goroutine，这里有6个方法，正常情况下这7个goroutine可以并发运行，但是本程序使用缓存大小为3的通道来做并发控制，导致同时只有4个goroutine可以并发运行，包括主goroutine。


```
func sayHello(index int){
    fmt.Println(index )
}
var work []func(int)

var limit = make(chan int, 3)

func main() {
	work = append(work, sayHello, sayHello, sayHello, sayHello, sayHello, sayHello)
	for i, w := range work {
		go func(w func(int), i int) {
			limit <- 1
			w(i)
			<-limit
		}(w, i)
	}
	select{}
}
```

## 锁
sync包里有两个锁的实现，sync.Mutex和sync.RWMutext。

对于任意sync.Mutex或者sync.RWMutex的变量l，和n<m，调用n次l.unlock happens-before调用m次l.Lock() 。

```
var l sync.Mutex
var a string

func f() {
	a = "hello, world"
	l.Unlock()
}

func main() {
	l.Lock()
	go f()
	l.Lock()
	print(a)
}
```
可以确保输出“print world”。第一次调用l.Unlock()发生在第二次调用l.Lock()之前。

另外对任何一个sync.RWMutex类型的变量l来说，存在一个次数n,调用 l.RLock操作happens after 调用n次 l.Unlock（释放写锁）并且相应的 l.RUnlock happen before 调用n+1次 l.Lock（写锁）

```
package main

import (
    "fmt"
    "sync"
)
 
var l sync.RWMutex
var a string
 
func unlock() {
    a = "Unlock" //1
    l.Unlock()   //2
}
 
func runlock() {
    a = "Runlock" //3
    l.RUnlock()   //4
}
 
func main() {
    l.Lock()    //5
    go unlock() //6
 
    l.RLock()      //7
    fmt.Println(a) //8
    go runlock()   //9
    l.Lock()     //10
    fmt.Print(a) //11
    l.Unlock()
}

```

## 一次执行
sync包提供了一个多goroutines时候安全初始化的机制，就是通过使用Once类型。多个线程对于某个f能够执行once.Do(f).但是仅执行一个f。其他调用将被阻塞直到f返回。

once.Do(f)单次调用函数f happens-before任意其他once.Do(f)返回。

```
var a string
var once sync.Once

func setup() {
	a = "hello, world"
	fmt.Println("setup over")
}

func doprint() {
	once.Do(setup)
	print(a)
}

func twoprint() {
	go doprint()
	go doprint()
}
```

twoprint函数仅调用setup一次。setup函数将在print之前返回。所以“hello world”会被打印两次，“setup over”仅打印一次。

## 不正确的同步
注意：并发执行的时候，一个goroutine对变量的读取操作r可能会观察到另外一个goroutine对变量执行的写操作。即使这样，并不意味着r之后的读会知道w之前的写。

```
var a, b int

func f() {
	a = 1
	b = 2
}

func g() {
	print(b)
	print(a)
}

func main() {
	go f()
	g()
}
```

比如上面代码一个可能的输出为先打印2，然后打印0。因为这里的f里面可能会有重排。

双重检查锁尝试避免同步开销。例如，twoprint函数可能会不正确的写为：

```
var a string
var done bool

func setup() {
	a = "hello, world" //1
	done = true //2
}

func doprint() {
	if !done { //3
		once.Do(setup)
	}
	print(a) //4
}

func twoprint() {
	go doprint()
	go doprint()
}
```

这里重排会导致a的值不确定。1和2的顺序不确定。3和4的顺序同样不确定。

另外一个常见的不正确的同步是等待某个变量的值满足一定条件：

```
var a string
var done bool

func setup() {
	a = "hello, world"
	done = true
}

func main() {
	go setup()
	for !done {
	}
	print(a)
}
```

该案例同理，并不能确保在main函数内即使可以看到对变量done的写操作，也可以看到对变量a的操作，所以main函数还是可能会输出空串。更糟糕的是由于两个goroutine没有对变量done做同步措施，main函数所在goroutine可能看不到对done的写操作，从而导致main函数所在goroutine一直运行在for循环出。


下面是这个函数的变体

```
type T struct {
	msg string
}

var g *T

func setup() {
	t := new(T)
	t.msg = "hello, world"
	g = t
}

func main() {
	go setup()
	for g == nil {
	}
	print(g.msg)
}
```

及时主函数main观察到了g!=nil并结束了循环。也还是不能保证该goroutine会观察到g.msg的初始化。

以上例子可以看出，解决办法只有一个：使用显式同步。


原文：<https://golang.org/ref/mem>