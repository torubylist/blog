---
layout:     post
title:      "高效Go语言-初始化-方法-接口"
subtitle:   " \"Effective-Go-Initialization-Methods-Interfaces\""
date:       2019-05-25 20:00:00
author:     "会飞的蜗牛"
header-img: "img/effective-go.jpg"
tags:
    - Initialization
    - Methods
    - Effective-Go
---

> 翻译主要是为了更好的掌握文中的内容，如有不妥之处或者想我交流，欢迎随时给我发邮件，谢谢。yongpingzhao1#gmail.com


# 初始化
虽然Golang的初始化与C或C++中的初始化看起来有点类似，但Go中的初始化更强大。不仅可以在初始化期间构建复杂数据结构，而且可以正确处理初始化对象之间的排序问题，即使在不同的包之间也是如此。

## 常量
Go中的常量就是常数，在编译时创建，即使在函数中定义为局部变量时也是如此，并且只能是数字，字符（runes），字符串或布尔值。由于编译时限制，定义它们的表达式必须是常量表达式，可由编译器进行计算。 例如，1 << 3 是常量表达式，而 math.Sin(math.Pi/4) 不是，因为函数 math.Sin 需要在运行时发生。

在 Go 中，使用 iota 枚举器创建枚举常量。由于 iota 可以是表达式的一部分，并且表达式可以隐式重复，因此很容易构建复杂的值集。 

```
type ByteSize float64

const (
    _           = iota // ignore first value by assigning to blank identifier
    KB ByteSize = 1 << (10 * iota)
    MB
    GB
    TB
    PB
    EB
    ZB
    YB
)
```

可以给任何用户自定义类型添加诸如 String 之类的方法，这样使得任意值都可以自动格式化打印。 尽管我们看到它最常用于结构体，但这种技术对于标量类型（如ByteSize等浮点类型）也很有用。

```
func (b ByteSize) String() string {
    switch {
    case b >= YB:
        return fmt.Sprintf("%.2fYB", b/YB)
    case b >= ZB:
        return fmt.Sprintf("%.2fZB", b/ZB)
    case b >= EB:
        return fmt.Sprintf("%.2fEB", b/EB)
    case b >= PB:
        return fmt.Sprintf("%.2fPB", b/PB)
    case b >= TB:
        return fmt.Sprintf("%.2fTB", b/TB)
    case b >= GB:
        return fmt.Sprintf("%.2fGB", b/GB)
    case b >= MB:
        return fmt.Sprintf("%.2fMB", b/MB)
    case b >= KB:
        return fmt.Sprintf("%.2fKB", b/KB)
    }
    return fmt.Sprintf("%.2fB", b)
}
```
表达式 YB 打印 1.00YB，而 ByteSize(1e13) 打印 9.09 TB

使用Sprintf来实现ByteSize的String方法是安全的（避免无限期重复），不是因为转换而是因为它调用的是Sprintf的％f，而不是%s，如果是%s则会陷入无限递归的情况。Sprintf只会在%s时调用String方法 ，而％f想要一个浮点值。


## 变量
变量可以像常量一样初始化，但变量初始化可以是在运行时计算的通用表达式。

```
var (
    home   = os.Getenv("HOME")
    user   = os.Getenv("USER")
    gopath = os.Getenv("GOPATH")
)
```

### init函数
最终，每个源文件都可以定义不带参数的 init 函数来设置所需的状态。（实际上每个文件可以有多个init函数。）最终意味着：init 函数是在程序包中所有变量声明都被初始化，以及所有被导入的程序包中的变量初始化之后才被调用。

除了用于无法通过声明来表示的初始化以外，init 函数的常见用途是在实际执行开始之前验证或修复程序状态的正确性。

```
func init() {
    if user == "" {
        log.Fatal("$USER not set")
    }
    if home == "" {
        home = "/home/" + user
    }
    if gopath == "" {
        gopath = home + "/go"
    }
    // gopath may be overridden by --gopath flag on command line.
    flag.StringVar(&gopath, "gopath", gopath, "override default GOPATH")
}

```
# 方法
## 指针和值
正如我们在ByteSize中看到的那样，可以为任何命名类型定义方法（指针或接口除外），接收者不一定要是结构体。

在上面的切片讨论中，我们编写了一个Append函数。 我们可以将它定义为切片上的方法。为此，我们首先声明一个命名类型，我们可以将该方法绑定到这个类型，然后使该类型的值作为该方法的接收器。

```
type ByteSlice []byte

func (slice ByteSlice) Append(data []byte) []byte {
    // Body exactly the same as the Append function defined above.
}
```
上面的代码仍然需要更新后的切片作为返回值返回。我们可以通过重新定义方法，将ByteSlice指针作为接收器来消除这种笨拙的写法，这样该方法就可以修改调用者的切片。

```
func (p *ByteSlice) Append(data []byte) {
    slice := *p
    // Body as above, without the return.
    *p = slice
}
```
事实上，我们可以做的更好，我们来修改函数让它看起来像一个标准的Write方法。

```
func (p *ByteSlice) Write(data []byte) (n int, err error) {
    slice := *p
    // Again as above.
    *p = slice
    return len(data), nil
}
```

这样`*ByteSlice`就满足了标准接口io.Writer。举例来说，我们可以往ByteSlice里面输入字符串。

```
    var b ByteSlice
    fmt.Fprintf(&b, "This hour has %d days\n", 7)
```
我们传递的是ByteSlice的地址，因为只有`*ByteSlice`满足`io.Writer`接口。有关接收器的规则是可以在指针和值上调用*值*方法，但只能在指针上调用*指针*方法。

这个规则是因为基于指针的方法可以修改接收器; 而在值上调用它们，接收器是值的副本，因此修改将被丢弃。所以，Golang不允许这种错误。但是有一个例外。当值是可寻址的时候，通过自动插入地址运算符来处理对值调用指针方法是很常见的情况。在我们的例子中，变量b是可寻址的，所以我们可以用b.Write调用它的Write方法。编译器会为我们重写为(＆b).Write。

顺便说一句，在字节切片上使用Write方法是bytes.Buffer实现的关键。

# 接口和其他类型
## 接口
Go语言中的接口提供了一种指定对象行为的方法：如果它走路像只鸭子，叫起来也像只鸭子，那么我们就认为它是只鸭子。这就是著名的鸭子类型。鸭子类型更关注你的行为，而不是你的类型。 我们已经看到了几个简单的例子，自定义打印可以通过String方法实现，而Fprintf可以使用Write方法生成输出。在Go代码中只有一个或两个方法的接口比较常见，并且通常给出从该方法派生的名称，例如用于实现Write方法的io.Writer。

一个类型可以实现多个接口。 例如，如果一个集合(类型)实现了 sort.Interface ，包含Len()， Less(i，j int) bool 和 Swap(i，j int)等方法，那么它就可以按sort包的函数进行排序，也可以实现自定义格式化程序。下面的例子中，Sequence满足以上两者。

```
type Sequence []int

// Methods required by sort.Interface.
func (s Sequence) Len() int {
    return len(s)
}
func (s Sequence) Less(i, j int) bool {
    return s[i] < s[j]
}
func (s Sequence) Swap(i, j int) {
    s[i], s[j] = s[j], s[i]
}

// Copy returns a copy of the Sequence.
func (s Sequence) Copy() Sequence {
    copy := make(Sequence, 0, len(s))
    return append(copy, s...)
}

// Method for printing - sorts the elements before printing.
func (s Sequence) String() string {
    s = s.Copy() // Make a copy; don't overwrite argument.
    sort.Sort(s)
    str := "["
    for i, elem := range s { // Loop is O(N²); will fix that in next example.
        if i > 0 {
            str += " "
        }
        str += fmt.Sprint(elem)
    }
    return str + "]"
}
```

## 强制类型转换
Sequence的String方法正在重新实现Sprint已经为切片做过的工作。（它的时间复杂度是很差的O(N²)），但是如果我们在调用Sprint之前将Sequence转换为[]int，我们就可以加快实现的速度。

```
func (s Sequence) String() string {
    s = s.Copy()
    sort.Sort(s)
    return fmt.Sprint([]int(s))
}
```

此方法是从String方法安全地调用Sprintf进行转换的另一个示例。 如果忽略类型名称，两种类型（Sequence和[]int）本质是相同的，在它们之间进行转换是合法的。强制类型转换不会创建新值，它只是暂时表现为值具有新类型。还有其他的转换，例如从整数到浮点数，确实会创建一个新值。）

Go程序中的一个习惯用法就是转换表达式的类型以访问不同的方法集。作为示例，我们可以使用现有类型sort.IntSlice将整个示例缩减为：

```
type Sequence []int

// Method for printing - sorts the elements before printing
func (s Sequence) String() string {
    s = s.Copy()
    sort.IntSlice(s).Sort()
    return fmt.Sprint([]int(s))
}
```
现在，我们不是让Sequence实现多个接口（排序和打印），而是使用数据items（Sequence，sort.IntSlice和[]int）可以互相转换的能力，每个类型都做了一部分工作。这种实践不是很常见，但确实很有效。

## 接口转换与类型断言
类型开关是一种形式转换：对于开关中的每种情况，它们采用接口，在某种意义上将其转换为该案例的类型。这是fmt.Printf中使用类型开关将值转换为字符串代码的简化版。如果它已经是一个字符串，就返回该接口保存的实际字符串值，而如果它有一个String方法，就返回调用该方法的结果。

```
type Stringer interface {
    String() string
}

var value interface{} // Value provided by caller.
switch str := value.(type) {
case string:
    return str
case Stringer:
    return str.String()
}
```

第一种情况找到了具体的值，而后者将接口转换为另一个接口。以这种方式混合多种类型判断是完全没有问题的。

如果我们关心的只有一种类型怎么办？如果我们知道该值包含一个字符串，我们只想提取这个字符串？一个类型的案例的用类型开关当然可以实现，但类型断言也可以有同样的效果。类型断言获取接口值并从中提取指定显式类型的值。该语法借鉴了打开类型开关的子句，但是显式的使用了类型名字而不是type关键字：

```
value.(typeName)
```

结果是类型为typeName的静态类型的新值。该类型要么是接口持有的具体类型，要么是该值可以转换的第二接口类型。要提取我们知道的字符串，我们可以写：

```
str := value.(string)
```

但如果事实证明该值不包含字符串，程序将崩溃并出现运行时错误。为了防止这种情况，使用“逗号，ok”的习惯用法来测试值是否是字符串：

```
str, ok := value.(string)
if ok {
    fmt.Printf("string value is: %q\n", str)
} else {
    fmt.Printf("value is not a string\n")
}
```

如果类型断言失败了，str还是可以作为string类型存在，但是是string类型的零值，即空字符串。

作为该功能的说明，这里有一个if-else语句，它等同于类型开关。

```
if str, ok := value.(string); ok {
    return str
} else if str, ok := value.(Stringer); ok {
    return str.String()
}
```

## 通用性
如果某种类型仅用于实现接口，并且没有超出该接口的导出方法，则无需导出该类型本身。仅导出接口可以清楚地表明类型的实现，除了接口中描述的内容之外，该值没有任何别的有用的行为。这就避免了在每个实例上重复写文档的需要。

在这种情况下，构造函数应返回接口值而不是具体类型。例如，在散列库中，crc32.NewIEEE和adler32.New都返回接口hash.Hash32。在Go程序中用Adler-32替换CRC-32算法只需要改变构造函数调用，其余代码不受算法更改的影响。

类似的方法允许各种加密包中的流加密算法与它们链接在一起的分组密码分离。crypto/cipher包中的Block接口指定块密码的行为，它提供单个数据块的加密。通过类比bufio包，可以使用实现此接口的密码包来构造流接口，由Stream接口表示，而无需知道块加密的细节。

crypto/cipher 接口:

```
type Block interface {
    BlockSize() int
    Encrypt(src, dst []byte)
    Decrypt(src, dst []byte)
}

type Stream interface {
    XORKeyStream(dst, src []byte)
}
```

这是计数器模式（CTR）Stream的定义，它将块密码转换为流密码，注意块密码的细节被抽象掉了：

```
// NewCTR returns a Stream that encrypts/decrypts using the given Block in
// counter mode. The length of iv must be the same as the Block's block size.
func NewCTR(block Block, iv []byte) Stream
```

NewCTR不仅适用于一种特定的加密算法和数据源，还适用于任何Block接口和Stream的实现。 因为它们返回的是接口值，所以将CTR加密替换为其他加密模式仅需要在本地进行更改。首先必须修改构造器，但由于仅将结果看作是一个Stream，因此不会有差异。

## 接口和方法
可以在任何事物上添加方法，所以任何事物都可以满足某个接口。下面的示例是在http包中，其定义了Handler接口，任何实现了Handler的对象都可以提供HTTP请求。

type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}

ResponseWriter本身就是一个接口，可以访问将响应返回给客户端所需的方法。这些方法包括标准的Write方法，因此可以在任何可以使用io.Writer的地方使用http.ResponseWriter。Request是一个结构体，包含来自客户端的请求的解析表示。

为简洁起见，我们忽略POST并假设HTTP请求总是GET，简化不会影响处理程序的编码方式。这是一个简单但完整的处理程序实现，用于计算访问页面的次数。

```
// Simple counter server.
type Counter struct {
    n int
}

func (ctr *Counter) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    ctr.n++
    fmt.Fprintf(w, "counter = %d\n", ctr.n)
}
```
请注意Fprintf如何打印到http.ResponseWriter里。

```
import "net/http"
...
ctr := new(Counter)
http.Handle("/counter", ctr)
```

然而，我们为什么需要一个Counter结构体呢？整型就可以满足我们的需求了。（接受者需要是指针，这样递增数据才能对调用者可见）

```
// Simpler counter server.
type Counter int

func (ctr *Counter) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    *ctr++
    fmt.Fprintf(w, "counter = %d\n", *ctr)
}
```
如果您的程序有一些内部状态需要通知页面已被访问，该怎么办？将网页绑定一个通道。

```
// A channel that sends a notification on each visit.
// (Probably want the channel to be buffered.)
type Chan chan *http.Request

func (ch Chan) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    ch <- req
    fmt.Fprint(w, "notification sent")
}
```

最后，假设我们想要在 /args 上显示调用服务器二进制文件时使用的参数，编写一个函数来打印参数很容易。

```
func ArgServer() {
    fmt.Println(os.Args)
}
```

我们如何将它转换为一个HTTP服务呢？ 我们可以使ArgServer成为某种类型的方法，我们忽略这个类型的值，但还有一种更干净的方法。由于我们可以为除指针和接口之外的任何类型定义方法，因此我们可以为函数编写方法。http包中包含以下代码：

```
// The HandlerFunc type is an adapter to allow the use of
// ordinary functions as HTTP handlers.  If f is a function
// with the appropriate signature, HandlerFunc(f) is a
// Handler object that calls f.
type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP calls f(w, req).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, req *Request) {
    f(w, req)
}
```

HandlerFunc是一个带有ServeHTTP方法的类型，因此该类型的值可以为HTTP请求提供服务。 看一下方法的实现：接收器是函数f，方法调用f。这可能看起来很奇怪，但它与接收器是一个通道和在通道上发送方法没有什么不同。

要将ArgServer变为HTTP服务，我们首先将其修改为具有正确的签名。

```
// Argument server.
func ArgServer(w http.ResponseWriter, req *http.Request) {
    fmt.Fprintln(w, os.Args)
}
```

ArgServer现在具有与HandlerFunc相同的签名，因此可以将ArgServer转换为HandlerFunc类型以访问后者的方法，就像我们将Sequence转换为IntSlice以访问IntSlice.Sort一样。实现代码很简洁：

```
http.Handle("/args", http.HandlerFunc(ArgServer))
```

当有人访问/args页面时，在该页面上装有类型为HandlerFunc值为ArgServer的处理器。 HTTP服务器将调用该类型的ServeHTTP方法，ArgServer作为接收器，它将依次调用ArgServer（通过HandlerFunc.ServeHTTP内的调用f（w，req））。 然后将显示参数。

在本节中，我们通过struct，int，channel和function等类型创建了一个HTTP服务器。主要是因为接口只是几组方法，所以可以为(几乎)任何类型定义接口。


## 参考
<https://golang.org/doc/effective_go.html>