---
layout:     post
title:      "Go并发模式-超时处理"
subtitle:   " \"Go Concurrency Patterns: Timing out, moving on\""
date:       2019-06-01 20:00:00
author:     "会飞的蜗牛"
header-img: "img/go-channel.jpg"
tags:
    - Concurrency
    - Go
    
---

> 翻译主要是为了更好的掌握文中的内容，如有不妥之处或者想我交流，欢迎随时给我发邮件，谢谢。yongpingzhao1#gmail.com


众所周知，并发编程有自己特有的模式。其中一个很好的例子就是超时处理。尽管Golang channel并不能直接处理超时，但是我们却可以很容易使用channel来实现超时机制。也就是说当我们从ch接收数据，但是可以设置一个超时机制，超过设定时间的数据就不再接收。从最简单的做起，先创建一个信号通道，再创建一个goroutine，给通道发送数据前先睡眠一段时间。

```
timeout := make(chan bool, 1)
go func() {
    time.Sleep(1 * time.Second)
    timeout <- true
}()
```
然后使用select机制，从ch通道读取数据或者从timeout读取数据。如果一秒钟后，ch通道里面无数据，select就会从timeout通道读取数据，而不会从ch读取数据。


```
select {
case <-ch:
    // a read from ch has occurred
case <-timeout:
    // the read from ch has timed out
}
```

timeout通道是有缓冲的，缓冲值为1，可以向timeout发送一个数据，然后关闭通道。goroutine并不会关心数据是否被读取。这就是说goroutine并不会被通道阻塞，无论数据读取的事件是否发生。timeout通道最终会被垃圾回收器释放掉。

（在这个例子中，我们使用time.Sleep来演示goroutine和channel机制。在实际程序中，你应该使用 `time.After`，一个返回值为channel的函数，并在指定的时间后在该通道上发送数据。）

让我们看看这种模式的一个变体。在这个例子中，我们有一个程序可以同时从多个复制数据库中读取数据。这里只需要其中一个结果，它只接受最先到达的查询结果。

函数Query的输入是数据库连接切片和一个查询字符串。它并行查询每个数据库并返回它收到的第一个响应：	

```
func Query(conns []Conn, query string) Result {
    ch := make(chan Result)
    for _, conn := range conns {
        go func(c Conn) {
            select {
            case ch <- c.DoQuery(query):
            default:
            }
        }(conn)
    }
    return <-ch
}

```

在此示例中，闭包函数执行了非阻塞发送，它通过在select语句中使用默认情况下的send操作来实现。如果给通道发送数据无法立即通过，就会选择default选项。 使发送非阻塞保证循环中没有任何goroutine会挂起。但是，如果结果在主函数进入接收之前到达，则发送可能会失败，因为没有人准备就绪。

这个问题是竞争条件的教科书般的示例，要修复这个问题也是非常容易的。我们只需要使用缓冲通道ch（通过添加缓冲区长度作为第二个参数），保证第一个发送可送达。这确保了发送始终会成功，无论执行顺序如何，都将检索到达的第一个值。

这两个例子说明了Go可以简单地实现goroutines之间的复杂交互。

By Andrew Gerrand

## 原文
<https://blog.golang.org/go-concurrency-patterns-timing-out-and>