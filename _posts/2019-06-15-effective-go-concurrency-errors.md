---
layout:     post
title:      "高效Go语言-并发-错误处理"
subtitle:   " \"Effective-Go-Concurrency-Errors\""
date:       2019-06-15 15:00:00
author:     "会飞的蜗牛"
header-img: "img/effective-go.jpg"
tags:
    - Concurrency
    - Errors
    - Effective-GO
    
---

> 翻译主要是为了更好的掌握文中的内容，如有不妥之处或者想我交流，欢迎随时给我发邮件，谢谢。yongpingzhao1#gmail.com

# 并发
## 通过通信共享数据
并发编程是一个很大的主题，这里只讨论一些Go语言的特有功能。

由于并发编程中要实现对共享变量的正确访问极其微妙，所以大多情况下并发编程显得很难。 所以Go语言鼓励采用一种完全不同的方法，在这种方法中，通过通道传递共享值。实际上，不会由单独的执行线程主动共享。在任何给定时间，只有一个goroutine可以访问该值。根据这种设计，就不会发生数据竞争。为了鼓励通过这种思维方式编码，我们将其简化为一个口号：

不要通过共享内存进行通信;相反，通过通信共享内存。

这种方法可以采取太多措施。例如，可以通过在整数变量周围放置互斥量来完成引用计数。但作为一种高级方法，使用通道来控制访问可以更容易地编写清晰，正确的程序。

考虑这种模型的一种方法是考虑在一个CPU上运行的典型单线程程序，它不需要同步原语。现在运行另一个这样的实例它也不需要同步，而是让这两个人沟通。如果通信是同步器，则仍然不需要其他同步。例如，Unix管道完美地适合这个模型。尽管Go的并发方法源于Hoare的顺序通信进程（CSP），但它也可以被视为Unix管道的类型安全泛化。


## Goroutines（协程）
之所以被称为goroutines，是因为现有的术语 - 线程，协同程序，进程等-都传达了不准确的含义。goroutine有一个简单的模型：它是一个与同一地址空间中的其他goroutine同时执行的函数。goroutine是轻量级的，几乎不比以分配的栈空间多多少。并且栈空间开始很小，因此goroutine开销确实很小，并且可以根据需要分配（和释放）堆存储来增长。

Goroutines被多路复用到多个OS线程上，因此如果出现阻塞的情况，例如在等待I/O时，其他线程还可以继续运行。这种设计隐藏了线程创建和管理的复杂性。

使用前置go关键字起一个新的goroutine，调用函数或方法。当调用完成时，goroutine静静地退出。（效果类似于Unix shell和后台运行命令的表示法。）


```
go list.Sort()  // run list.Sort concurrently; don't wait for it.
```
在goroutine中调用字面量函数也很方便。

```
func Announce(message string, delay time.Duration) {
    go func() {
        time.Sleep(delay)
        fmt.Println(message)
    }()  // Note the parentheses - must call the function.
}
```

在Go语言中，函数字面量是闭包：确保函数引用的变量只要它们处于活跃状态就可以一直存活。

这些示例不太实用，因为函数无法发出完成信号。为此，我们需要通道。



## Channels（通道）
与map类似，通道使用make分配，结果值用作对底层数据结构的引用。如果提供了整数参数（参数是可选的），则会设置通道的缓冲区大小。对于无缓冲或同步通道，则无需设置，默认值为零。

```
ci := make(chan int)            // 无缓冲通道
cj := make(chan int, 0)         // 无缓冲通道
cs := make(chan *os.File, 100)  // 缓冲通道 
```

无缓冲通道将通信-值的交换-与同步相结合-保证两个计算（goroutine）处于已知状态。

有很多很好的channel习惯用法。在上一节中，我们在后台启动了一个排序。通道允许等待排序goroutine完成。

```
c := make(chan int)  // Allocate a channel.
// Start the sort in a goroutine; when it completes, signal on the channel.
go func() {
    list.Sort()
    c <- 1  // Send a signal; value does not matter.
}()
doSomethingForAWhile()
<-c   // Wait for sort to finish; discard sent value.
```

接收器始终阻塞，直到有数据要接收。如果通道未缓冲，则发送方将阻塞，直到接收方收到该值。如果通道有缓冲区，则只有发送方只有在缓冲区满了才会阻塞; 如果缓冲区已满，则表示等待某个接收方接收到某个值。

可以像信号量一样使用缓冲信道，例如限制吞吐量。在此示例中，传入请求被传递给处理函数，处理函数将值发送到通道，处理请求，然后从通道接收值。通道缓冲区的容量限制了同时调用的数量。

```
var sem = make(chan int, MaxOutstanding)

func handle(r *Request) {
    sem <- 1    // Wait for active queue to drain.
    process(r)  // May take a long time.
    <-sem       // Done; enable next request to run.
}

func Serve(queue chan *Request) {
    for {
        req := <-queue
        go handle(req)  // Don't wait for handle to finish.
    }
}
```

一旦MaxOutstanding处理器正在执行进程，任何更多将阻止尝试发送到填充的通道缓冲区，直到其中一个现有处理程序完成并从缓冲区接收。

但是这种设计存在一个问题：Serve 为每个传入的请求创建一个新的 goroutine，即使任意时刻只有MaxOutstanding个goroutine可以同事运行。因此，如果请求进入太快，程序可以消耗无限的资源。我们可以通过修改代码以解决goroutines的创建来解决这个问题。下文是一种解决方案，但要注意它还有一个bug，我们随后会修复：

```
func Serve(queue chan *Request) {
    for req := range queue {
        sem <- 1
        go func() {
            process(req) // Buggy; see explanation below.
            <-sem
        }()
    }
}
```

这个bug是在 Go for 循环中，循环变量在每次迭代中都被重用，因此req变量在所有goroutine中共享。这不是我们想要的。我们需要确保req对于每个goroutine都是唯一的。 其中的一种方法是将req的值作为参数传递给goroutine中的闭包：

```
func Serve(queue chan *Request) {
    for req := range queue {
        sem <- 1
        go func(req *Request) {
            process(req)
            <-sem
        }(req)
    }
}
```

将此版本与之前版本进行比较，可以了解闭包声明和运行方式的不同之处。 另一种解决方案是创建一个具有相同名称的新变量，如下例所示：

```
func Serve(queue chan *Request) {
    for req := range queue {
        req := req // Create new instance of req for the goroutine.
        sem <- 1
        go func() {
            process(req)
            <-sem
        }()
    }
}
```

这么写看起来有点奇怪：

```
req := req
```

但是在Go语言中这样做是合法的，且是习惯用法。 你得到一个具有相同名称的新变量，这样每个goroutine的变量都是唯一的。

回到实现通用服务器的问题上来，另一个有效的资源管理途径是启动固定数量的handle Goroutine，每个Goroutine都直接从channel中读取请求。这个固定的数值就是同时执行process的最大并发数。Serve函数还需要一个额外的channel，用来等待退出通知；当创建完所有的Goroutine之后， Server 自身阻塞在该 channel 上等待结束信号。


```
func handle(queue chan *Request) {
    for r := range queue {
        process(r)
    }
}

func Serve(clientRequests chan *Request, quit chan bool) {
    // Start handlers
    for i := 0; i < MaxOutstanding; i++ {
        go handle(clientRequests)
    }
    <-quit  // Wait to be told to exit.
}
```

## Channels of channels(通道的通道)
Go语言的一个最重要的属性是通道是一等”公民“，可以像任何其他基本数据类型一样分配，传递。该属性的常见用途是实现安全、并行的解复用处理。

在上一节的示例中，handle是请求的理想化处理程序，但我们没有定义它正在处理的类型。 如果该类型包括一个用来响应的通道，则每个客户端可以提供其独特的响应方式。这是Request类型的定义。

```
type Request struct {
    args        []int
    f           func([]int) int
    resultChan  chan int
}
```

客户端提供函数及其参数，以及一个内部的通道来接收回答消息。

```
func sum(a []int) (s int) {
    for _, v := range a {
        s += v
    }
    return
}

request := &Request{[]int{3, 4, 5}, sum, make(chan int)}
// Send request
clientRequests <- request
// Wait for response.
fmt.Printf("answer: %d\n", <-request.resultChan)
```

在服务器端，只需要修改处理函数。

```
func handle(queue chan *Request) {
    for req := range queue {
        req.resultChan <- req.f(req.args)
    }
}
```

显然还有很多其他方面可优化以提高其可用性，但是这套代码已经可以作为一类限速，并行，无阻塞的RPC系统的框架了，并且没有使用锁。

## Parallelization（并行）
并发的另一个应用是跨多CPU核心实现并行计算。如果计算可以分成独立执行的多份，那就可以并行计算，每份完成之后通过通道发送信号。

假设我们有一个大的向量需要执行昂贵的计算开销，这个操作可以分成独立的多份，那么这就是理想的案例。

```
type Vector []float64

// Apply the operation to v[i], v[i+1] ... up to v[n-1].
func (v Vector) DoSome(i, n int, u Vector, c chan int) {
    for ; i < n; i++ {
        v[i] += u.Op(v[i])
    }
    c <- 1    // signal that this piece is done
}
```

我们在循环中独立启动各个部分，每个CPU执行一片。 他们可以按任何顺序完成都不影响结果。 我们只是在启动所有goroutine之后通过排空通道来计算完成信号。

```
const numCPU = 4 // number of CPU cores

func (v Vector) DoAll(u Vector) {
    c := make(chan int, numCPU)  // Buffering optional but sensible.
    for i := 0; i < numCPU; i++ {
        go v.DoSome(i*len(v)/numCPU, (i+1)*len(v)/numCPU, u, c)
    }
    // Drain the channel.
    for i := 0; i < numCPU; i++ {
        <-c    // wait for one task to complete
    }
    // All done.
}
```

除了为numCPU创建常量值，我们还可以向运行时询问适当的值。函数runtime.NumCPU返回机器中的硬件CPU核心数，因此我们可以这样编码：

```
var numCPU = runtime.NumCPU()
```

还有一个函数runtime.GOMAXPROCS，它报告（或设置）Go程序可以同时运行的用户指定的最大核心数，默认为runtime.NumCPU的值，但可以通过设置类似的shell环境变量或通过使用正数调用函数来覆盖它。 如果想查询它的值可以零值调用。因此，如果我们想要查询用户的资源请求，我们应该这样写

```
var numCPU = runtime.GOMAXPROCS(0)
```

一定不要混淆并发和并行，并发-将程序结构化为独立执行组件，并行-多CPU上执行并行计算。 尽管Go的并发特性可以使一些问题易于构造为并行计算，但Go是一种并发语言，而不是并行语言，并非所有并行化问题都适合Go的模型。有关两者区别的讨论，请参阅此博客文章中的演讲。

## 一个“leaky buffer”示例
并发编程的工具甚至可以更简单的表达一些非并发的想法。下面是从RPC包中抽象出来的一个例子。客户端goroutine循环从某个源（可能是网络）接收数据。 为了避免频繁内存分配和缓冲区释放，程序在内部保留了一个空闲列表，并使用缓冲通道来表示。 如果通道为空，则分配新的缓冲区。 一旦消息缓冲准备就绪，它就经由serverChan发送到服务器端。

```
var freeList = make(chan *Buffer, 100)
var serverChan = make(chan *Buffer)

func client() {
    for {
        var b *Buffer
        // Grab a buffer if available; allocate if not.
        select {
        case b = <-freeList:
            // Got one; nothing more to do.
        default:
            // None free, so allocate a new one.
            b = new(Buffer)
        }
        load(b)              // Read next message from the net.
        serverChan <- b      // Send to server.
    }
}
```
服务器循环从客户端接收每条消息，对其进行处理，并将缓冲区返回到空闲列表。

```
func server() {
    for {
        b := <-serverChan    // Wait for work.
        process(b)
        // Reuse buffer if there's room.
        select {
        case freeList <- b:
            // Buffer on free list; nothing more to do.
        default:
            // Free list full, just carry on.
        }
    }
}
```

客户端尝试从freeList获取缓冲数据; 如果没有，它会分配一个新的缓冲区。 除非列表已满，否则服务器发送到freeList会将b放回到空闲列表中，在这种情况下，缓冲区将被丢弃以供垃圾回收器回收。（select语句中的默认子句在其他情况都未就绪时执行，这意味着select永远不会阻塞。）此实现只需几行就构建一个漏桶空闲链表，依赖于缓冲通道和垃圾收集器进行记录。

# 错误处理
库例程通常必须向调用者返回某种错误指示。如前所述，Go的多值返回可以很容易地返回详细的错误描述和正常的返回值。使用这种方式能够提供详细的错误信息，这是一种很好的编码方式。例如，正如我们将看到的，os.Open不仅在失败时返回一个nil指针，它还返回一个描述错误的错误值。

按照惯例，错误也有内置类型error，一个简单的内置接口。

```
type error interface {
    Error() string
}
```

库编写者可以自由地使用更丰富的模型来实现此接口，这样不仅可以查看错误，还可以提供一些上下文。 如上所述，除了通常的*os.File返回值之外，os.Open还返回错误值。 如果文件成功打开，则错误将为nil，但是当出现问题时，它将返回 os.PathError：

```
// PathError records an error and the operation and
// file path that caused it.
type PathError struct {
    Op string    // "open", "unlink", etc.
    Path string  // The associated file.
    Err error    // Returned by the system call.
}

func (e *PathError) Error() string {
    return e.Op + " " + e.Path + ": " + e.Err.Error()
}
```

PathError的Error生成如下字符串:

```
open /etc/passwx: no such file or directory
```

这样一个包括文件名，操作和由它触发的操作系统错误的详细信息，即使远离调用者打印出来也是有用的， 它比简单的“no such file or directory”提供了更多信息。

在可行的情况下，错误字符串应标识其来源，比如在前面加上一些诸如操作名称或包名称的前缀信息。例如，在包图像中，由于未知格式导致的解码错误应标示为“image: unknown format”。

关注精确详细的错误信息的调用者可以使用类型开关或类型断言来查找特定错误并提取详细信息。 对于PathErrors，这可能包括检查内部Err字段是否存在可恢复的故障。

```
for try := 0; try < 2; try++ {
    file, err = os.Create(filename)
    if err == nil {
        return
    }
    if e, ok := err.(*os.PathError); ok && e.Err == syscall.ENOSPC {
        deleteTempFiles()  // Recover some space.
        continue
    }
    return
}
```

这里的第二个if语句是另一个类型断言。 如果失败，ok将是false，e将为nil。 如果成功，ok将为true，这意味着错误的类型为* os.PathError，然后我们可以通过e检查有关错误的更多信息。


## Panic
向调用者报告错误的常用方法就是将错误作为额外的返回值返回。 Read方法是一个众所周知的例子，它返回字节数和错误。但如果错误无法恢复怎么办？有时程序根本无法继续。

为此，有一个内置函数panic，它实际上会产生一个运行时错误导致程序停止。 该函数可以采用任意类型的单个参数-通常是一个字符串，并在程序死亡时打印。 它也是一种表明不可能发生的事情的方式，例如退出无限循环。

```
// A toy implementation of cube root using Newton's method.
func CubeRoot(x float64) float64 {
    z := x/3   // Arbitrary initial value
    for i := 0; i < 1e6; i++ {
        prevz := z
        z -= (z*z*z-x) / (3*z*z)
        if veryClose(z, prevz) {
            return z
        }
    }
    // A million iterations has not converged; something is wrong.
    panic(fmt.Sprintf("CubeRoot(%g) did not converge", x))
}
```

这只是一个例子，但真正的库函数应该避免panic。 如果问题可以被掩盖或解决，那么更好的方式是让事情继续运行而不是结束整个程序。 一个可能的反例是在初始化期间：如果库函数真的无法自我修复，那么panic可能也是合理的。

```
var user = os.Getenv("USER")

func init() {
    if user == "" {
        panic("no value for $USER")
    }
}
```

## Recover
当panic被调用时，运行时错误也隐式地包含在内，例如切片索引越界或类型断言失败，程序会立即停止执行并开始展开goroutine的堆栈，沿途会运行所有的defered函数。如果该展开到达goroutine堆栈的顶部，则意味着程序死亡。但是，其实可以使用内置函数recover来重新获得对goroutine的控制并恢复正常执行。

对recover的调用会停止展开并返回一个参数传递给panic。因为在展开时运行的唯一代码是在延迟函数内部，所以recover仅在延迟函数内有用。

recover的一个应用是关闭服务器内的失败goroutine而不会杀死其他正在执行的goroutine。

```
func server(workChan <-chan *Work) {
    for work := range workChan {
        go safelyDo(work)
    }
}

func safelyDo(work *Work) {
    defer func() {
        if err := recover(); err != nil {
            log.Println("work failed:", err)
        }
    }()
    do(work)
}
```
在这个例子中，如果do(work)函数发生了panic，将结果收集到日志中，并且干净的退出不会影响任何其他线程。在延期闭包函数中不需要做任何其他事情，调用recover可以处理好一切。

除非直接从延迟函数调用，不然recover总是返回nil，所以延迟函数可以调用拥有恢复机制而不会崩溃的库例程。例如，safelyDo中的延迟函数可能在调用recover之前调用日志记录函数，并且日志记录代码将不受panic状态的影响。

有了修复模式，do函数（以及它的调用）可以通过调用panic来清晰的摆脱所有的不良情况。 我们可以使用这种模式来简化复杂软件中的错误处理。让我们看一个理想版本的regexp的，它通过调用具有本地错误类型的panic来报告错误解析。下面是Error的定义，错误方法和Compile函数。

```
// Error is the type of a parse error; it satisfies the error interface.
type Error string
func (e Error) Error() string {
    return string(e)
}

// error is a method of *Regexp that reports parsing errors by
// panicking with an Error.
func (regexp *Regexp) error(err string) {
    panic(Error(err))
}

// Compile returns a parsed representation of the regular expression.
func Compile(str string) (regexp *Regexp, err error) {
    regexp = new(Regexp)
    // doParse will panic if there is a parse error.
    defer func() {
        if e := recover(); e != nil {
            regexp = nil    // Clear return value.
            err = e.(Error) // Will re-panic if not a parse error.
        }
    }()
    return regexp.doParse(str), nil
}
```


如果doParse函数发生了panic，恢复函数将返回值设置为nil，延迟函数可以修改命名返回值。错误恢复代码会对返回的错误类型进行类型断言，判断其是否属于Error类型。如果类型断言失败，则会引发运行时错误，并继续进行栈展开，最后终止程序 —— 这个过程将不再会被中断。 此检查意味着如果发生意外情况，例如索引超出范围，即使我们使用panic和recover来处理解析错误，代码也会失败。

有了上面的错误处理过程，调用error方法（由于它是一个类型的绑定的方法，因而即使与内建类型error同名，也不会带来什么问题，甚至是一直更加自然的用法）使得“解析错误”的报告更加方便，无需费心去考虑手工处理栈展开过程的复杂问题。


With error handling in place, the error method (because it's a method bound to a type, it's fine, even natural, for it to have the same name as the builtin error type) makes it easy to report parse errors without worrying about unwinding the parse stack by hand:

```
if pos == 0 {
    re.error("'*' illegal at start of expression")
}
```

尽管这种模式很有效，但是也只应该在包里使用。Parse方法将其内部对panic的调用隐藏在error之中；而不会将panics信息暴露给外部使用者。这是一个设计良好且值得学习的编程技巧。

顺便说一下，如果发生实际错误，这种re-panic的习惯用法会改变panic值。但是，原始故障和新故障都将显示在崩溃报告中，因此问题的根本原因依然可见。 因此，这种简单的重新启动方法通常就足够了-程序最终还是会崩溃-但如果您只想显示原始值，则可以编写更多代码来过滤意外问题并使用原始错误重新发生panic。这就留给读者课后练习吧。

# A web server
让我们完成一个完整的Go程序，实现一个Web服务。这个实际上是一类“Web reserver”，对web服务的二次封装。 Google在chart.apis.google.com上提供了一项服务，可以将数据自动格式化为图表和图形。但是，交互式使用很难，因为您需要将数据作为查询放入URL中。这里的程序为一种形式的数据提供了一个更好的界面：给定一小段文本，它调用图表服务器来生成QR码，对文本进行编码后的方型图。可以使用手机的相机抓取该图像并将其解释为例如URL，从而节省您在手机的小键盘中键入URL。

下面是完成的代码。 解释如下。

```
package main

import (
    "flag"
    "html/template"
    "log"
    "net/http"
)

var addr = flag.String("addr", ":1718", "http service address") // Q=17, R=18

var templ = template.Must(template.New("qr").Parse(templateStr))

func main() {
    flag.Parse()
    http.Handle("/", http.HandlerFunc(QR))
    err := http.ListenAndServe(*addr, nil)
    if err != nil {
        log.Fatal("ListenAndServe:", err)
    }
}

func QR(w http.ResponseWriter, req *http.Request) {
    templ.Execute(w, req.FormValue("s"))
}

const templateStr = `
<html>
<head>
<title>QR Link Generator</title>
</head>
<body>
{{if .}}
<img src="http://chart.apis.google.com/chart?chs=300x300&cht=qr&choe=UTF-8&chl={{.}}" />
<br>
{{.}}
<br>
<br>
{{end}}
<form action="/" name=f method="GET"><input maxLength=1024 size=70
name=s value="" title="Text to QR Encode"><input type=submit
value="Show QR" name=qr>
</form>
</body>
</html>
`
```


这部分代码的逻辑应该很容易看懂。addr标志为我们的服务器设置默认HTTP端口。模板变量templ是整个程序的核心逻辑，构建一个HTML模板，由服务器执行以显示页面。

main函数解析标志，并使用我们上面讨论的机制将函数QR绑定到服务器的根路径。然后调用http.ListenAndServe来启动服务器，它在服务器运行时阻塞。

QR只接收包含表单数据的请求，并以名为s的表单值对数据执行模板生成QR图。

模板包html/template功能强大，这个程序只涉及它最基本的功能。本质上，它通过替换从传递给templ.Execute的数据项派生的元素来动态重写一段HTML文本，在本例中为表单值。 在模板文本（templateStr）中，双括号分隔的片段表示模板操作。 从`{{if.}}` 到 `{{end}}`的部分仅在调用当前数据项的值时执行。也就是` . `的值非空的时候。当字符串为空时，这一部分模板代码不会被执行。

两个代码段`{{.}}`表示在网页上显示呈现给模板的数据-查询字符串。HTML模板包自动提供恰当的转义，以便文本可以安全显示。

模板字符串的其余部分只是页面加载时显示的HTML。 如果这里的解释比较粗陋，请参阅`template`包的文档以进行更全面的讨论。

这样你就拥有了一个有用的Web服务器，仅包含几行代码和一些数据驱动的HTML文本。 Go足够强大，可以在几行中实现一个这样的server。
