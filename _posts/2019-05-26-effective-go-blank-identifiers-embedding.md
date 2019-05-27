---
layout:     post
title:      "高效Go语言-空白符-内嵌"
subtitle:   " \"Effective-Go-Blank-Identifier-Embedding\""
date:       2019-05-26 20:00:00
author:     "会飞的蜗牛"
header-img: "img/effective-go.jpg"
tags:
    - Blank Identifier
    - Embeding
    
---
> 翻译主要是为了更好的掌握文中的内容，如有不妥之处或者想我交流，欢迎随时给我发邮件，谢谢。yongpingzhao1#gmail.com

# 空白标志符
到目前为止，我们已经在loop和maps的上下文中多次提过空白标志符。任何类型的值都可以分配或声明空白标识符，并且丢弃该值也不会任何害处。这有点像写入Unix /dev/null 文件，它表示一个只写值，用作需要变量但实际值无关的占位符。它的用途比我们已经看过的更广泛。

- 多个赋值中的空白标识符
- 在for range循环中使用空标识符是一般情况的特例：多重赋值。

如果赋值给多个值，但其中一个值并不会被程序使用，则赋值左侧的空白标识符将不会创建虚拟变量并清楚地表明该值将被丢弃。例如，当函数返回值和错误，但我们只需要错误时，这时就可以使用空白标识符来丢弃不相关的值。

```
if _, err := os.Stat(path); os.IsNotExist(err) {
	fmt.Printf("%s does not exist\n", path)
}
```

有时，您会看到丢弃错误值的代码，以便忽略错误，这是一种可怕的做法。写golang代码需要始终检查错误返回，这是有原因的。

```
// Bad! This code will crash if path does not exist.
fi, _ := os.Stat(path)
if fi.IsDir() {
    fmt.Printf("%s is a directory\n", path)
}
```


## 未使用的导入和变量
导入包或声明变量而不使用它是错误的。未使用的导入会破坏程序并减慢编译速度，而进行了初始化但未使用的变量则至少是计算的浪费，并且可能表示更大的错误。但是，当程序处于活跃开发状态时，通常会出现未使用的导入和变量，删除它们只是为了让编译继续进行，只是为了以后再次需要它们。空白标识符则为我们提供了一种解决方案。

这个尚未写完的程序有两个未使用的导入（fmt和io）和一个未使用的变量（fd），所以它不会编译，但是我们来看看到目前为止代码是否正确的。

```
package main

import (
    "fmt"
    "io"
    "log"
    "os"
)

func main() {
    fd, err := os.Open("test.go")
    if err != nil {
        log.Fatal(err)
    }
    // TODO: use fd.
}
```
要消除有关未使用导入的错误，请使用空白标识符来引用导入包中的符号。类似地，将未使用的变量fd分配给空白标识符将消除未使用的变量的错误。这个版本的程序确实剋编译成功。

```
package main

import (
    "fmt"
    "io"
    "log"
    "os"
)

var _ = fmt.Printf // For debugging; delete when done.
var _ io.Reader    // For debugging; delete when done.

func main() {
    fd, err := os.Open("test.go")
    if err != nil {
        log.Fatal(err)
    }
    // TODO: use fd.
    _ = fd
}
```

按照惯例，消除导入错误的声明应该在导入之后立即进行，并进行注释，便于查找，方便清理。


## 副作用式导入
最终，上一个示例中未使用的导入（如fmt或io）应该被使用或者删除。空白赋值将代码标识为WIP。 但有时有用的是包的副作用，代码中无需显示使用包。例如，在init函数期间，net/http/pprof包会注册提供调试信息的HTTP处理程序。它有一个导出的API，但大多数客户端只需要处理程序注册并通过网页访问其数据。要是仅仅为了包的副作用导入包，请将包重命名为空标识符：

```
import _ "net/http/pprof"
```

这种类型的导入很明显，就是为了包的副作用才导入的包，因为并没有其他的用途。在这个文件中，它没有名字。（如果它有，但是我们没有使用这个名字，编译器就会拒绝这个程序）

## 接口检查

正如我们在上面的接口讨论中所看到的，类型不需要明确声明它实现了某个接口。相反，类型只是通过实现接口的方法来实现接口。 实际上，大多数接口转换都是静态的，因此在编译时进行检查。 例如，将`*os.File`传递给期望参数是`io.Reader`的函数将不会编译，除非`*os.File`实现了`io.Reader`接口。

但是，某些接口检查确实是在运行时进行的。`encoding/json`包中一个实例，该包定义了Marshaler接口。 当JSON编码器接收到实现该接口的值时，编码器调用值的`marshaling`方法将其转换为JSON而不是执行标准转换。编码器在运行时使用类型断言检查此属性，如：

```
m, ok := val.(json.Marshaler)
```

如果只需要询问类型是否实现了某个接口，而不实际使用接口本身，请使用空白标识符忽略类型声明的值：

```
if _, ok := val.(json.Marshaler); ok {
    fmt.Printf("value %v of type %T implements json.Marshaler\n", val, val)
}
```

出现这种情况的一个原因必须确保在包中实现了满足该接口的类型。如果一个类型，例如json.RawMessage，需要一个自定义的JSON编码，它应该实现json.Marshaler，但没有静态转换，编译器就不会自动验证这一点。 如果类型无意中没有满足该接口，则JSON编码器仍然可以工作，但不会使用自定义实现。为了保证接口实现正确，可以在包中使用使用空标识符的全局声明：

```
var _ json.Marshaler = (*RawMessage)(nil)
```

在此声明中，涉及将`*RawMessage`转换为`Marshaler`的赋值，则要求`*RawMessage`实现`Marshaler`，并且会在编译时检查该属性。 如果`json.Marshaler`接口发生更改，此程序包将不能编译，我们就会注意到代码需要更新。

此代码中，空白标识符表示声明仅存在于类型检查中，而不是创建变量。但是，不要为满足接口的每种类型执行此操作。按照惯例，只有在代码中不存在静态类型转换时才会使用此类声明，这并不常见。

# 内嵌
Go没有提供典型的类型驱动的子类化概念，但它确实能够通过在结构体或接口中嵌入类型来“借用”已经实现的各个部分。

接口嵌入非常简单。我们之前提到过io.Reader和io.Writer接口，看看他们的定义。

```
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}
```

io 包还导出了几个其他的接口，这些接口指定了可以实现多个这样的方法的对象。 例如，有io.ReadWriter，一个包含Read和Write的接口。我们也可以通过明确地列出这两个方法来指定io.ReadWriter，但是将两个接口嵌入以形成新接口的设计更容易理解也更令人回味，如下所示：

```
// ReadWriter is the interface that combines the Reader and Writer interfaces.
type ReadWriter interface {
    Reader
    Writer
}
```

这看起来就如它定义所示：一个ReadWriter既能做Reader的事情也可以做Writer的事情。这是一个嵌入接口的联合体（这些接口的方法不能有交集）。注意：只有接口能够嵌入接口。

类似的想法也可以应用于结构体的定义，其实现稍稍复杂一些。bufio 包有两个结构类型， bufio.Reader 和 bufio.Writer，它们分别实现了 io 包的类似接口。bufio 还实现了一个缓冲的 reader/writer，它通过使用嵌入将 reader 和 writer 组合到一个结构中来实现：列出了结构体中的类型但却不命名。

```
// ReadWriter stores pointers to a Reader and a Writer.
// It implements io.ReadWriter.
type ReadWriter struct {
    *Reader  // *bufio.Reader
    *Writer  // *bufio.Writer
}
```
嵌入的元素是结构体的指针，当然必须初始化以指向有效的结构体，然后才能使用它们。该
ReadWriter结构也可以写成

```
type ReadWriter struct {
    reader *Reader
    writer *Writer
}
```

但是为了满足 io 接口，必须提升字段的方法，我们必须提供一个转发方法，如下所示：

```
func (rw *ReadWriter) Read(p []byte) (n int, err error) {
    return rw.reader.Read(p)
}
```
通过对结构体直接进行“内嵌”，我们避免了一些复杂的记录。 内嵌类型的方法是免费使用的，这意味着bufio.ReadWriter不仅具有bufio.Reader和bufio.Writer的方法，它还满足所有三个接口：io.Reader，io.Writer和io.ReadWriter。

嵌入与子类化有很重要的不同之处。当我们内嵌一个类型时，该类型的方法会自动成为外部类型的方法，但是当它们被调用时，方法的接收者是内部类型，而不是外部类型。在我们的示例中，当调用bufio.ReadWriter的Read方法时，它与上面写出的转发方法具有完全相同的效果。接收器是ReadWriter的的reader字段，而不是ReadWriter本身。

内嵌也可以简洁方便。 此示例显示嵌入字段以及常规命名字段。

```
type Job struct {
    Command string
    *log.Logger
}
```
现在，Job 类型具有 Print，Printf，Println 和 *log.Logger 的其他方法。当然，我们可以给 Logger 一个字段名称，但是没有必要这样做。现在，一旦初始化，我们就可以记录 Job：

```
job.Println("starting now...")
```

Logger 是 Job 结构体的常规字段，因此我们可以在Job的构造函数中以常规的方式初始化它，就像这样，


```
func NewJob(command string, logger *log.Logger) *Job {
    return &Job{command, logger}
}
```

或者更复杂点,

```
job := &Job{command, log.New(os.Stderr, "Job: ", log.Ldate)}
```

如果我们需要直接引用嵌入字段，则忽略包限定符，将字段的类型名称用作字段名称，就像 ReadWriter 结构的 Read 方法中一样。在这里，如果我们需要访问 Job 变量作业的 *log.Logger，我们将编写 job.Logger，如果我们想要优化 Logger 的方法，这将非常有用。

```
func (job *Job) Printf(format string, args ...interface{}) {
    job.Logger.Printf("%q: %s", job.Command, fmt.Sprintf(format, args...))
}
```

内嵌类型引入了命名冲突的问题，但要解决这个问题也不难。

首先，如果内嵌类型跟外部类型有相同的字段或者方法X，则内嵌的字段或方法 X 会被隐藏. 如果 log.Logger 包含一个名为 Command 的字段或方法，则 Job 的 Command 字段将占主导地位。

其次，同一名称出现在同一嵌套级别是错误的，如果 Job 结构已经包含另一个名为 Logger 的字段或方法，那么再嵌入 log.Logger 就是错误的。但是，如果重复名称在类型定义之外的程序中从未使用过，则不会产生错误。这个限定为在外部进行类型嵌入修改提供了保护。如果新添加的字段与另一个子类型中的字段冲突，如果该字段没有被访问过，也同样不会有问题。

