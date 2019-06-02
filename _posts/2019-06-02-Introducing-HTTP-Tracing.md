---
layout:     post
title:      "【译】引入HTTP跟踪"
subtitle:   " \"Introducing-HTTP-Tracing\""
date:       2019-06-02 20:00:00
author:     "会飞的蜗牛"
header-img: "img/Introducing-HTTP-Tracing.jpg"
tags:
    - HTTP Tracing
    - Go
    
    
---

> 翻译主要是为了更好的掌握文中的内容，如有不妥之处或者想我交流，欢迎随时给我发邮件，谢谢。yongpingzhao1#gmail.com


# 引言
在Golang 1.7中，我们引入了HTTP跟踪，用于在HTTP客户端发起请求的整个生命周期中收集详细信息。 net/http/httptrace 包提供了对HTTP跟踪的支持。收集的信息可用于调试延迟问题，服务监控，编写自适应系统等。


# HTTP 事件
httpstrace包提供了许多的钩子，用于HTTP数据包在服务器和客户端往返过程中收集事件信息。这些事件包括：

- 连接建立
- 连接重用
- DNS查询
- 将请求写入总线
- 读取返回值

# 事件跟踪
你可以通过将包含钩子函数的`*httptrace.ClientTrace`放入请求的context.Context来启用HTTP跟踪。 各种http.RoundTripper实现通过查找上下文的`*httptrace.ClientTrace`，并调用相关的钩子函数来报告内部事件。

跟踪范围限定在请求的上下文中，用户应在启动请求之前将`*httptrace.ClientTrace`放入请求上下文。

```
    req, _ := http.NewRequest("GET", "http://example.com", nil)
    trace := &httptrace.ClientTrace{
        DNSDone: func(dnsInfo httptrace.DNSDoneInfo) {
            fmt.Printf("DNS Info: %+v\n", dnsInfo)
        },
        GotConn: func(connInfo httptrace.GotConnInfo) {
            fmt.Printf("Got Conn: %+v\n", connInfo)
        },
    }
    req = req.WithContext(httptrace.WithClientTrace(req.Context(), trace))
    if _, err := http.DefaultTransport.RoundTrip(req); err != nil {
        log.Fatal(err)
    }
```
在数据往返过程中，http.DefaultTransport将在事件发生时调用每个钩子。 一旦DNS查找完成，上面的程序将打印DNS信息。当与请求的主机建立连接时，它将同样地打印连接信息。

# 通过http.Client来跟踪
跟踪机制旨在跟踪单个http.Transport.RoundTrip的生命周期中的事件。 但是，客户端可以进行多次往返以完成HTTP请求。 例如，在URL重定向的情况下，当客户端遵循HTTP重定向时，将多次调用已注册的挂钩，从而产生多个请求。 用户负责在http.Client级别识别此类事件。 下面的程序使用http.RoundTripper包装器标识当前请求。

```
ackage main

import (
    "fmt"
    "log"
    "net/http"
    "net/http/httptrace"
)

// transport is an http.RoundTripper that keeps track of the in-flight
// request and implements hooks to report HTTP tracing events.
type transport struct {
    current *http.Request
}

// RoundTrip wraps http.DefaultTransport.RoundTrip to keep track
// of the current request.
func (t *transport) RoundTrip(req *http.Request) (*http.Response, error) {
    t.current = req
    return http.DefaultTransport.RoundTrip(req)
}

// GotConn prints whether the connection has been used previously
// for the current request.
func (t *transport) GotConn(info httptrace.GotConnInfo) {
    fmt.Printf("Connection reused for %v? %v\n", t.current.URL, info.Reused)
}

func main() {
    t := &transport{}

    req, _ := http.NewRequest("GET", "https://google.com", nil)
    trace := &httptrace.ClientTrace{
        GotConn: t.GotConn,
    }
    req = req.WithContext(httptrace.WithClientTrace(req.Context(), trace))

    client := &http.Client{Transport: t}
    if _, err := client.Do(req); err != nil {
        log.Fatal(err)
    }
}

```

上面的代码将google.com重定向到www.google.com，并输出：

```
Connection reused for https://google.com? false
Connection reused for https://www.google.com/? false
```

`net/http`包中的`Transport`支持跟踪`HTTP/1`和`HTTP/2`请求。

如果你想实现自定义`http.RoundTripper`，则可以通过检查`*httptest.ClientTrace`的请求上下文并在事件发生时调用相关钩子来支持跟踪。

# 总结
对于那些有兴趣调试HTTP请求延迟和编写出站流量网络调试工具的人来说，HTTP跟踪对Go语言来说是很有价值的补充。通过启用这个新工具，我们希望看到来自社区的HTTP调试，基准测试和可视化工具 - 例如httpstat。

By Jaana Burcu Dogan

# 原文
<https://blog.golang.org/http-tracing>