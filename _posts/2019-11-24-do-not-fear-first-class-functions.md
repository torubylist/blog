---
layout:     post
title:      "【译】Do not fear first class functions"
subtitle:   " \"functional programming\""
date:       2019-11-24 20:00:00
author:     "会飞的蜗牛"
header-img: "img/go_memory_model.jpg"
tags:
    - Golang

---
原文：[do-not-fear-first-class-functions](https://dave.cheney.net/2016/11/13/do-not-fear-first-class-functions)

两年前，我在台上做了一个类似的演讲，告诉你们我在go语言里是如何处理配置选项的。这次演说是基于Rob Pike的一篇博客。[Self-referential functions and the design of options](https://commandcenter.blogspot.com.au/2014/01/self-referential-functions-and-design.html)

从那以后，非常开心的是看到这个模式的逐渐成熟，并运用与gRPC项目。我个人愚见，这个项目将options模式发挥的淋漓尽致。

但是，几个月前，我在伦敦的Gopher会议上，听到了一些不同的意见。他们对Go新手们对函数返回函数的理解表示担忧，而这正是函数式options的基石。他们担心新手们不能完全理解这种编程模式。

是的，这让我有点失落，因为我一直以为函数作为一等公民的特性是go语言最好的礼物。我们应该能够很好的加以利用。今天，我在这里通过一些实例的讲解，让你不再害怕函数式编程。

## 函数式options复盘
一个典型的函数式options编程如下：

```
type Config struct {
	....
}

func WithReticulatedSplines(c *Config) { ... }

type Terrain struct {
	config Config
}

func NewTerrain(options ...func(*Config)) *Terrain {
	var t Terrain
	for _, option := range options {
		option(&t.config)
	}
	
	return &t
}

func main() {
	t := NewTerrain(WithReticulatedSplines)
	// [ simulation intensifies ]
}

```

我们以某些选项开始，传递一个指向配置结构的指针到函数中。然后再把这些函数传递到一个类型构造器中，在构造器的内部，这些选项将被轮流调用，传递指向Config值的指针。最终。我们调用这些option函数，并将得到我们想要的结果。


是的，每个人都应该非常熟悉这个模式。我个人的理解是，造成困惑的原因是你的option函数需要携带一些参数。例如，我们有一个`WithCities`, 它的目的是将城市的个数添加到terrain模型中。

```
 // WithCities adds n cities to the Terrain model
func WithCities(n int) func(*Config) { ... }

func main() {        
        t := NewTerrain(WithCities(9))      
        // ...
}
```
因为`WithCities`带有一个参数，我们不能简单的简单的将`WithCities`传递给`NewTerrain`，因为签名不匹配。所以我们需要让`WithCities`返回一个函数，这个函数的签名跟`options`一致。这样我们就可以计算`WithCities`，然后将返回值传递给`NewTerrain`.

## 一等公民函数
接下来怎么分析呢？我们先把这个大的功能拆开。最基本的，计算一个函数返回一个值。下面这个函数传入两个数字，返回一个数字。

```
package math

func Min(a, b float64) float64
```
传入一个切片，范湖一个结构体指针。

```
package bytes

func NewReader(b []byte) *Reader
```

传入一个数字，返回一个函数。

```
func WithCities(n int) func(*Config)
```

`WithCities`返回的是一个函数，入参是一个Config类型指针。我们将这种把函数看作是一个常规的值的功能叫做：一等公民函数。


## interface.Apply
另一种常见的模式是使用接口来重写函数式options。

```
type Option interface {
        Apply(*Config)
}
```

不再是函数类型，而是一个接口。我们把它叫做Option。给他一个方法，Apply方法的入参是一个指向Congfig的指针。

```
func NewTerrain(options ...Option) *Terrain {
        var config Config
        for _, option := range options {
                option.Apply(&config)
        }
        // ...
}
```

这样，不论什么时候，我们调用`NewTerrain`，我们都会传入一个实现了Options的接口。在`NewTerrain`内部，正如之前一样，我们遍历options，并依次调用`Apply`方法。

这跟之前的案例是有点类似的，只不过这次传入的不再是函数类型，而是接口类型。我们再来看另外一个例子。申明了`WithReticulatedSplines `这样一个option。

```
type splines struct{}

func (s *splines) Apply(c *Config) { ... }

func WithReticulatedSplines() Option {
        return new(splines)
}
```

因为我们要传递的一个接口的实现，我们需要一个类型来定义一个Apply方法。同时也要定义个构造器函数。返回一个`splines`的实例。那么，在这里，我们可以实现更多的代码逻辑。

重写`WithCities`

```
type cities struct {
        cities int
}

func (c *cities) Apply(c *Config) { ... }

func WithCities(n int) Option {
        return &cities{
                cities: n,
        }
}

func main() {
        t := NewTerrain(WithReticulatedSplines(), WithCities(9))
        // ...
}
```

把这些糅合到一起，调用`NewTerrain`的时候，传入`WithReticulatedSplines` 和`WithCities`的计算结果。

在去年的GopherCon上，Tomás Senart谈到了两种方式，函数式编程和面向接口编程。在我们的例子中，这两种模式都有，看起来是相等的。

但是，其实你能看到。函数式编程需要的代码量更少。

## 行为封装
我们暂且将接口放一旁，继续讨论函数的其他特性。

当我们调用一个函数或者方法时，通常会传递一些数据。这个函数的功能通常就是转换数据并做一些操作。函数作为入参传递的时候通常是传递一些需要被执行的行为，而不是需要转换的数据。实际上。传递函数值是一种延迟执行，通常在不同的上下文中。

为了说明，下面以一个简单的计算器作为例子。

```
type Calculator struct {
        acc float64
}

const (
        OP_ADD = 1 << iota
        OP_SUB
        OP_MUL
)

func (c *Calculator) Do(op int, v float64) float64 {
        switch op {
        case OP_ADD:
                c.acc += v
        case OP_SUB:
                c.acc -= v
        case OP_MUL:
                c.acc *= v
        default:
                panic("unhandled operation")
        }
        return c.acc
}
```

只有一个方式，`Do`，传入一个操作符和操作数v。通常为了方便起见，Do一般会在操作被执行之后返回计算结果。

```
func main() {
        var c Calculator
        fmt.Println(c.Do(OP_ADD, 100))     // 100
        fmt.Println(c.Do(OP_SUB, 50))      // 50
        fmt.Println(c.Do(OP_MUL, 2))       // 100
}
```

到目前为止，这个计算器还只知道处理加，减，乘。如果我们想要实现除运算。我们需要改写Do方法，加上除运算相关的代码。听起来也不是不合理。只是几行代码而已，但是如果我们还想要添加平方根和幂乘呢？

每次有新的功能加入，我们都需要修改Do方法，这样会导致它很难维护。

那么让我们来一点一点改造它吧。


```
type Calculator struct {
        acc float64
}

type opfunc func(float64, float64) float64

func (c *Calculator) Do(op opfunc, v float64) float64 {
        c.acc = op(c.acc, v)
        return c.acc
}
```

之前的计算器，只管理它自己的运算符。Calculator通常只有一个Do方法，这次把函数作为一个操作符和一个值传入。这样，每次调用Do的时候，它会去调用我们传入的操作函数，并且使用它自己的运算器和我们提供的运算数。


```
func Add(a, b float64) float64 { return a + b }
func Sub(a, b float64) float64 { return a - b }
func Mul(a, b float64) float64 { return a * b }

func main() {
        var c Calculator
        fmt.Println(c.Do(Add, 5))       // 5
        fmt.Println(c.Do(Sub, 3))       // 2
        fmt.Println(c.Do(Mul, 8))       // 16
}

```

## 扩展计算器
现在我们用函数来描述操作符，我们继续尝试扩展我们的函数来处理平方根操作。

```
func Sqrt(n, _ float64) float64 {
        return math.Sqrt(n)
}
```

但是，这里有个问题，math.Sqrt只有一个参数。而我们的Calculator的Do方法传入的操作符函数有两个参数。

```
func main() {
        var c Calculator
        c.Do(Add, 16)
        c.Do(Sqrt, 0) // operand ignored
}
```

也许我们可以忽略这个操作数，看起来有点奇怪，我们是不是可以做的更好呢？

我们来重新定义Add函数，让他返回一个函数，入参是一个值，返回另一个值。

```
func Add(n float64) func(float64) float64 {
        return func(acc float64) float64 {
                return acc + n
        }
}

func (c *Calculator) Do(op func(float64) float64) float64 {
        c.acc = op(c.acc)
        return c.acc
}
```

现在，Do调用操作符函数，传入它自己的运算符，并且把结果记录回运算符中。

```
func main() {
        var c Calculator
        c.Do(Add(10))   // 10
        c.Do(Add(20))   // 30
}
```

现在，main函数中，我们调用Do函数的时候并不是直接传入Add函数，而Add函数计算后的结果即一个函数，满足Do函数传参的需求。

```
func Sub(n float64) func(float64) float64 {
        return func(acc float64) float64 {
                return acc - n
        }
}

func Mul(n float64) func(float64) float64 {
        return func(acc float64) float64 {
                return acc * n
        }
}

func Sqrt() func(float64) float64 {
        return func(n float64) float64 {
                return math.Sqrt(n)
        }
}

func main() {
        var c Calculator
        c.Do(Add(2))
        c.Do(Sqrt())   // 1.41421356237
}

```

这里，平方根的实现避免了之前的那种尴尬的实现，同时我们修改了之前运算符函数，只传递和返回一个参数。

希望你已经注意到了，我们的Sqrt函数跟math.Sqrt是一样的，我们可以直接将math.Sqrt作为传参传入进去。

```
func main() {
        var c Calculator
        c.Do(Add(2))      // 2
        c.Do(math.Sqrt)   // 1.41421356237
        c.Do(math.Cos)    // 0.99969539804
}
```
我们以一个硬编码，数据转换的模型作为例子开端，转入到传入函数行为的函数式模型。然后，通过进一步的优化，将我们的计算器进一步通用化，不再需要关注参数的个数。

## Actors
让我们换个话题吧，讨论一下大家更关心的话题：并发，尤其是actors（角色模型）。为了阐述的更加高效，下面这个例子来自 Bryan Boreham 在GolangUK的演说。

假设我们想构造一个chat服务，我们计划是成为下一个Hipchat和Slack，


```
type Mux struct {
        mu    sync.Mutex
        conns map[net.Addr]net.Conn
}

func (m *Mux) Add(conn net.Conn) {
        m.mu.Lock()
        defer m.mu.Unlock()
        m.conns[conn.RemoteAddr()] = conn
}

func (m *Mux) Remove(addr net.Addr) {
        m.mu.Lock()
        defer m.mu.Unlock()
        delete(m.conns, addr)
}

func (m *Mux) SendMsg(msg string) error {
        m.mu.Lock()
        defer m.mu.Unlock()
        for _, conn := range m.conns {
                err := io.WriteString(conn, msg)
                if err != nil {
                        return err
                }
        }
        return nil
}
```
给所有注册的链接发送消息。因为这是个server，所有的方法会被并发调用。所以我们需要锁来保护这个并发的map。但是这是一个理想的实现吗？


## 不要通过共享内存来通信，而要通过通信来共享内存

我们的第一个建议是，不要通过锁来共享内存，而要通过通信来共享内存。

除了使用锁来序列化共享内存以外，还可以通过channel等通信方法来共享内存。

```
type Mux struct {
        add     chan net.Conn
        remove  chan net.Addr
        sendMsg chan string
}

func (m *Mux) Add(conn net.Conn) {
        m.add <- conn
}

func (m *Mux) Remove(addr net.Addr) {
        m.remove <- addr
}


func (m *Mux) SendMsg(msg string) error {
        m.sendMsg <- msg
        return nil
}

func (m *Mux) loop() {
        conns := make(map[net.Addr]net.Conn)
        for {
                select {
                case conn := <-m.add:
                        m.conns[conn.RemoteAddr()] = conn
                case addr := <-m.remove:
                        delete(m.conns, addr)
                case msg := <-m.sendMsg:
                        for _, conn := range m.conns {
                                io.WriteString(conn, msg)
                        }
                }
        }
}

```
这里不是通过锁来进行序列化，而是通过一个select-for来循环获取相应的值，这些值通过add，remove，sendMsg等channel来通信。我们不再需要锁来共享conns map的状态。

但是，这里还是有很多的硬编码逻辑。loop只知道做三件事，add，remove和广播一个消息。正如在之前的例子一样，添加一个新的功能到Mux里面将会涉及到 

1. 创建一个channel。
2. 添加一个帮助方法来通过channel发送数据。
3. 通过循环来处理数据，扩展select函数的逻辑。

正如Calculator一样，我们可以使用函数来重写我们的Mux，传递我们想要的行为进去。而不再只是数据转换。现在，每一个方法都发送一个即将执行的操作到循环里面，通过唯一的一个ops通道。

```
type Mux struct {
        ops chan func(map[net.Addr]net.Conn)
}

func (m *Mux) Add(conn net.Conn) {
        m.ops <- func(m map[net.Addr]net.Conn) {
                m[conn.RemoteAddr()] = conn
        }
}
```

在这个案例中，操作符是一个函数，这个函数传递的是一个 `net.Addr’s` 到 `net.Conn’s` 的映射。在真实的编程中，你很有可能碰到的是比这更复杂的情况。但是这里足够表达我们想要的内容了。

```
func (m *Mux) Remove(addr net.Addr) {
        m.ops <- func(m map[net.Addr]net.Conn) {
                delete(m, addr)
        }
}

func (m *Mux) SendMsg(msg string) error {
        m.ops <- func(m map[net.Addr]net.Conn) {
                for _, conn := range m {
                        io.WriteString(conn, msg)
                }
        }
        return nil
}

```

```
func (m *Mux) loop() { 
        conns := make(map[net.Addr]net.Conn)
        for op := range m.ops {
                op(conns)
        }
}
```

如上所示，我们将loop函数的逻辑抽象成一个个匿名函数。所以现在loop的工作就是创建一个map，然后遍历ops channel，传递这个map，等待调用。

但是这里还是有几个问题需要进一步探讨。首要的问题是SendMsg函数缺乏错误处理。写入到链接中的错误信息不会通过通信来返回到调用者。我们可以如下处理。


```
func (m *Mux) SendMsg(msg string) error {
        result := make(chan error, 1)
        m.ops <- func(m map[net.Addr]net.Conn) {
                for _, conn := range m.conns {
                        err := io.WriteString(conn, msg)
                        if err != nil {
                                result <- err
                                return
                        }
                }
                result <- nil
        }
        return <-result
}
```

为了处理匿名函数中的错误信息，我们通过给匿名函数传入通道来传递操作的结果。这里同样增加了一个同步点。最后这句话会一直阻塞，直到ops中的函数被执行。

```
func (m *Mux) loop() {
        conns := make(map[net.Addr]net.Conn)
        for op := range m.ops {
                op(conns)
        }
}
```

注意，我们并没有改变loop来配合错误这次处理。当然处理起来也很容易，给Mux添加一个新的函数，给每个client发送一个私有消息。

```
func (m *Mux) PrivateMsg(addr net.Addr, msg string) error {
        result := make(chan net.Conn, 1)
        m.ops <- func(m map[net.Addr]net.Conn) {
                result <- m[addr]
        }
        conn := <-result
        if conn == nil {
                return errors.Errorf("client %v not registered", addr)
        }
        return io.WriteString(conn, msg)
}
```
为了达到这个目的，我们通过ops通道给循环传入一个“查找函数”，这函数的目的是去查找map是否有这个连接，主要是通过获取map的结果来判断。如果这个结果是nil，则说明这个连接不存在，报错即可。如果存在，就可以往这个连接中发送消息。

## 总结
1. 一等公民函数给我们带来了巨大的能量。可以让我们传递一种行为，一些操作，而不仅仅是一些需要转换的死数据。
2. 一等公民函数也不是什么新鲜玩意，很多古老的语言都提供这个功能，甚至包括C。事实上，面向对象编程的程序员只是在删除指针的过程中失去了一等公民函数的访问权限。如果你是JavarScript程序员，你可能会对我们过去15分钟搞的东西感到很奇怪--这有什么了不起的呢。
3. 一等公民函数，跟其他Go语言提供的特性一样，应该有克制的使用。过度使用channel会导致过度编程。过度使用函数式编程也会使得程序难以阅读。但是这并不意味着你不能使用他们，只能适度的使用。
4. 我认为每个Go程序员都需要掌握一等公民函数的使用。它并不是go语言唯一的特性。Go程序员不应该惧怕他们。
5. 如果你学习如何使用接口，你也能学会如何使用一等公民函数。这并不难，只是需要你花一些时间去熟悉他们。


