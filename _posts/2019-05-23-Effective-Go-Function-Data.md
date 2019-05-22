---
layout:     post
title:      "高效Go语言-函数-数据结构"
subtitle:   " \"Effective-Go-Function-Data\""
date:       2019-05-23 20:00:00
author:     "会飞的蜗牛"
header-img: "img/effective-go.jpg"
tags:
    - Functions
    - Data
    - Effective-go
    
---

## 函数
### 多返回值
Go语言中不寻常的功能之一是函数和方法可以返回多个值。这种形式可用于改进C程序中的一些笨拙的习惯用法：错误返回，例如-1表示EOF和修改地址传递的参数。

在C中，一个写错误是由一个负的计数和一个隐藏在易变位置（a volatile location）的错误代码来表示的。在Go中，Write方法可以返回一个计数和一个错误：“是的，你写入了一些字节但并没有写完，因为设备被写满了”。来自包os文件的Write方法的签名是：

```
func (file *File) Write(b []byte) (n int, err error)
```

并且正如文档所述，当`n!=len(b)`时，它返回写入的字节数和非零error。这是一种常见的风格;有关更多示例，请参阅有关错误处理的部分。

类似的方法使得不再需要传递一个返回值指针来模拟一个引用参数。这里有一个非常简单的函数，用来从字节切片中的一个位置抓取一个数，返回该数和下一个位置。

```
func nextInt(b []byte, i int) (int, int) {
    for ; i < len(b) && !isDigit(b[i]); i++ {
    }
    x := 0
    for ; i < len(b) && isDigit(b[i]); i++ {
        x = x*10 + int(b[i]) - '0'
    }
    return x, i
}
```
你可以用它来扫描输入的切片b的数字：

```
    for i := 0; i < len(b); {
        x, i = nextInt(b, i)
        fmt.Println(x)
    }
```

### 命名返回值
Go函数的返回或结果“参数”可以给出名称并用作常规变量，就像传入参数一样。 命名时，它们在函数开始时被初始化为其类型的零值; 如果函数执行不带参数的return语句，则结果参数的当前值将用作返回值。

名字不是强制性的，但它们可以使代码更短更清晰：它们也是文档。如果我们将nextInt的结果进行命名，则其要返回的int是对应的哪一个就很显然了。

`func nextInt(b []byte, pos int) (value, nextPos int) {`

因为命名结果是被初始化的，并且与没有参数的return绑定在一起，所以它们即简单又清晰。这里是一个io.ReadFull的版本，很好地使用了这些特性：

```
func ReadFull(r Reader, buf []byte) (n int, err error) {
    for len(buf) > 0 && err == nil {
        var nr int
        nr, err = r.Read(buf)
        n += nr
        buf = buf[nr:]
    }
    return
}
```

### 延期执行
Go的defer语句调度函数调用（延迟函数），使其在执行defer的函数是在函数即将返回之前才被运行。这是处理必须释放的资源等情况的一种不寻常但有效的方法，无论函数通过哪个执行路径返回。典型的示例是解锁互斥锁或关闭文件。

```
// Contents returns the file's contents as a string.
func Contents(filename string) (string, error) {
    f, err := os.Open(filename)
    if err != nil {
        return "", err
    }
    defer f.Close()  // f.Close will run when we're finished.

    var result []byte
    buf := make([]byte, 100)
    for {
        n, err := f.Read(buf[0:])
        result = append(result, buf[0:n]...) // append is discussed later.
        if err != nil {
            if err == io.EOF {
                break
            }
            return "", err  // f will be closed if we return here.
        }
    }
    return string(result), nil // f will be closed if we return here.
}
```

推迟对诸如Close之类的函数的调用具有两个优点。首先，它保证你永远不会忘记关闭文件，如果你稍后编辑函数以添加新的返回路径，则很容易犯这个错误。其次，这意味着将关闭位于打开附近，这比将其放置在函数结束时要清晰得多。

延迟函数的参数（如果函数是一个方法，还包括接收者）在defer执行时计算，而不是在调用执行时计算。除了避免担心变量在函数执行时更改值，这还意味着单个被延期执行的调用点可以延期多个函数执行。这里有一个简单的例子。

```
for i := 0; i < 5; i++ {
    defer fmt.Printf("%d ", i)
}
```
被延迟的函数以LIFO顺序执行，因此当函数返回时，这段代码将打印4 3 2 1 0。一个更合理的例子是通过程序跟踪函数执行的方法。我们可以编写几个简单的跟踪程序，如下所示：

```
func trace(s string)   { fmt.Println("entering:", s) }
func untrace(s string) { fmt.Println("leaving:", s) }

// Use them like this:
func a() {
    trace("a")
    defer untrace("a")
    // do something....
}
```
利用被延期的函数的参数是在defer执行的时候被求值这个事实，我们可以做的更好些。 跟踪程序可以设置跟未踪程序的参数。这个例子：


```Golang
func trace(s string) string {
    fmt.Println("entering:", s)
    return s
}

func un(s string) {
    fmt.Println("leaving:", s)
}

func a() {
    defer un(trace("a"))
    fmt.Println("in a")
}

func b() {
    defer un(trace("b"))
    fmt.Println("in b")
    a()
}

func main() {
    b()
}
```

打印出

```
entering: b
in b
entering: a
in a
leaving: a
leaving: b
```
对于习惯于其它语言中的块级别资源管理的程序员，defer可能看起来很奇怪，但是它最有趣和强大的应用正是来自于这样的事实，这是基于函数的而不是基于块的。我们将会在panic和recover章节中看到它另一个可能的例子。

## 数据结构
### 使用new分配对象

Go语言中有两个分配原语，内置函数new和make。他们有不同的分工并适用于不同的类型，很多时候这令人困惑，但其实规则很简单。我们来谈谈第一个函数new。它是一个内置函数，可以分配内存，但与其他语言中的new不同，它不会初始化内存，它只会将内存归零。也就是说new(T)为类型为T的新项目分配归零存储并返回其地址，类型为*T的值。 在Golang术语中，它返回一个值为零的类型为T的指针。

由于new返回的内存为零值，因此在设计数据结构时分配每种类型的零值而无需进一步初始化是有帮助的。这意味着数据结构使用者可以使用new创建一个新对象并正常工作。例如，bytes.Buffer的文档声明“Buffer的零值是一个可以使用的空缓冲区”。 同样，sync.Mutex没有显式构造函数或Init方法。 相反，sync.Mutex的零值被定义为未锁定的互斥锁。

有用零值的属性是可以传递的，考虑下面这种类型声明：

```
type SyncedBuffer struct {
    lock    sync.Mutex
    buffer  bytes.Buffer
}
```

SyncedBuffer类型的值一旦被分配内存或者只是声明就可以使用了。在下面的代码片段中，p和v都都可以正常工作，而无需进一步内存分配。

```
p := new(SyncedBuffer)  // type *SyncedBuffer
var v SyncedBuffer      // type  SyncedBuffer
Constructors and composite literals
```
有时候零值还不够好，这个时候就需要进行初始化了，例如下面来自os包的例子。

```
func NewFile(fd int, name string) *File {
    if fd < 0 {
        return nil
    }
    f := new(File)
    f.fd = fd
    f.name = name
    f.dirinfo = nil
    f.nepipe = 0
    return f
}
```

还有很多模版。我们可以使用复合字面量来简化它，复合字面量是一个表达式，每次使用时都会创建一个新实例。

```
func NewFile(fd int, name string) *File {
    if fd < 0 {
        return nil
    }
    f := File{fd, name, nil, 0}
    return &f
}
```

请注意，与C语言不同，返回局部变量的地址是完全可以的。与函数变量关联的存储在函数返回后仍然存在。 实际上，获取复合字面量的地址在每次求值时都会分配一个新实例，因此我们可以将这两行结合起来。

```
    return &File{fd, name, nil, 0}
```
 
复合字面量的字段按顺序排列，并且必须全部存在。但是，通过将元素明确标记为key:value对，初始值设定项可以按任何顺序出现，缺少的元素保留为各自的零值。因此可以像下面这样表示。

```
    return &File{fd: fd, name: name}
```

有一种极限情况，如果复合字面量根本不包含任何字段，则会为该类型创建零值。表达式new(File)和＆File{}是等效的。

还可以为array，slice和maps创建复合字面量，并且字段标签适当地为索引或贴图键。 在这些示例中，只要它们是不同的，无论Enone，Eio和Einval的值如何，初始化都会起作用。

```
a := [...]string   {Enone: "no error", Eio: "Eio", Einval: "invalid argument"}
s := []string      {Enone: "no error", Eio: "Eio", Einval: "invalid argument"}
m := map[int]string{Enone: "no error", Eio: "Eio", Einval: "invalid argument"}
```

### 使用make分配对象
回到对象分配。内置函数make(T，args)用于与new(T)不同的用途。它仅创建slice，maps和channels，并返回类型为T(不是*T)的初始化（非零值）。不同的原因是这三种类型在字面量的表面之，对象使用前必须初始化数据结构的引用。例如，切片是一个三字段描述符，包含指向数据的指针(在数组内)，长度和容量，并且在初始化这些项之前，切片为零。对于slices，maps和channels，请初始化内部数据结构并准备好要使用的值。例如，

`make([]int, 10, 100)`

分配一个100个整数的数组，然后创建一个长度为10且容量为100的切片结构，指向数组的前10个元素。（制作切片时，可以省略容量;有关详细信息，请参阅切片部分。）相反，new([]int)返回指向新分配的零值化的切片结构的指针，即指向a的指针的零值片。

下面的例子说明了new和make之间的区别。

```
var p *[]int = new([]int)       // allocates slice structure; *p == nil; rarely useful
var v  []int = make([]int, 100) // the slice v now refers to a new array of 100 ints

// Unnecessarily complex:
var p *[]int = new([]int)
*p = make([]int, 100, 100)

// Idiomatic:
v := make([]int, 100)
```

记住，make只适合maps，slices和maps，并且不返回指针。获取显式指针，请使用new分配或明确获取变量的地址。

### Arrays（数组）
在规划内存的详细布局时，数组非常有用，有时可以帮助避免临时分配，但主要是它们是切片的内存构建块，这是下一节的主题。为这个主题奠定基础，这里有一些关于数组的指导。

数组在Go和C中的工作方式有很大差异。在Go中，

数组是值对象。将一个数组分配给另一个数组会复制所有元素。
特别的，如果将数组传递给函数，它将接收数组的副本，而不是指向它的指针。
数组的大小是其类型的一部分。类型[10]int和[20]int是不同的。
值属性可能很有用，但同时开销也很大，如果你想要类似C的行为和效率，你可以传递一个指向数组的指针。

```
func Sum(a *[3]float64) (sum float64) {
    for _, v := range *a {
        sum += v
    }
    return
}

array := [...]float64{7.0, 8.5, 9.1}
x := Sum(&array)  // Note the explicit address-of operator
```

但即便是这种风格也不是Go语言的习惯用法。习惯用法是使用切片。

### Slices（切片）
切片对数组进行了包装，为数字序列提供了更通用，功能更强大且更方便的接口。除了具有显式维度的项（如转换矩阵）之外，Go中的大多数数组编程都是使用切片而不是简单数组完成的。

切片保存对基础数组的引用，如果将一个切片分配给另一个切片，则两者都引用相同的数组。如果函数采用切片参数，则对切片元素所做的更改将对调用者可见，类似于将指针传递给基础数组。因此，Read函数可以接受slice参数而不是一个指针和计数，切片内的长度字段设置了要读取的数据量的上限。这是包os中File类型的Read方法的签名：

```
func (f *File) Read(buf []byte) (n int, err error)
```

该方法返回读取的字节数和错误值（如果有）。要读入更大缓冲区buf的前32个字节，所以将缓冲区切片化。

```
    n, err := f.Read(buf[0:32])
```    
这种切片是常见且有效的。 事实上，暂时不考虑效率，下面的代码片段也会读取缓冲区的前32个字节。

```
    var n int
    var err error
    for i := 0; i < 32; i++ {
        nbytes, e := f.Read(buf[i:i+1])  // Read one byte.
        n += nbytes
        if nbytes == 0 || e != nil {
            err = e
            break
        }
    }
```
只要切片长度仍然在底层数组的容量限制内，就可以改变切片的长度; 只需将切片自己的一部分分出来。可通过内置函数cap访问的切片容量，返回切片能够使用的最大长度。这是一个将数据附加到切片的函数。 如果数据量超出容量，则重新分配切片。返回结果是一个新切片。将函数len和cap应用于nil slice是合法，返回值为0。

```
func Append(slice, data []byte) []byte {
    l := len(slice)
    if l + len(data) > cap(slice) {  // reallocate
        // Allocate double what's needed, for future growth.
        newSlice := make([]byte, (l+len(data))*2)
        // The copy function is predeclared and works for any slice type.
        copy(newSlice, slice)
        slice = newSlice
    }
    slice = slice[0:l+len(data)]
    copy(slice[l:], data)
    return slice
}
```
最后我们必须返回切片，因为虽然Append可以修改切片的元素，但切片本身（包含指针，长度和容量的运行时数据结构）是按值传递的。

Append到切片的想法非常有用，它可以通过追加内置函数捕获。 但是要了解该功能的设计，我们需要更多信息，因此我们稍后会再回过头来看。

### 二维切片
Go的数组和切片是一维的。要创建二维数组或切片的等效项，有必要定义一个数组的数组或切片的切片，如下所示：

```
type Transform [3][3]float64  // A 3x3 array, really an array of arrays.
type LinesOfText [][]byte     // A slice of byte slices.
```
因为切片是可变长度的，所以可以使每个内切片具有不同的长度。这可能是一种常见情况，如在LinesOfText示例中：每一行都有一个独立的长度。

```
text := LinesOfText{
	[]byte("Now is the time"),
	[]byte("for all good gophers"),
	[]byte("to bring some fun to the party."),
}
```
有时需要分配2D切片，例如，在处理像素的扫描线的情况。有两种方法可以实现这一目标。 一种是每个元素独立分配切片; 另一种是分配单个数组并将各个切片指向它。使用哪个取决于你的应用程序。如果切片可能增长或缩小，则应单独分配它们以避免覆盖下一行; 如果不是，使用单个分配构造对象可能更有效。作为参考，这里是两种方法的草图。 首先，独立分配切片：

```
// Allocate the top-level slice.
picture := make([][]uint8, YSize) // One row per unit of y.
// Loop over the rows, allocating the slice for each row.
for i := range picture {
	picture[i] = make([]uint8, XSize)
}
```
其次，分配单个数组并将各个切片指向它

```
// Allocate the top-level slice, the same as before.
picture := make([][]uint8, YSize) // One row per unit of y.
// Allocate one large slice to hold all the pixels.
pixels := make([]uint8, XSize*YSize) // Has type []uint8 even though picture is [][]uint8.
// Loop over the rows, slicing each row from the front of the remaining pixels slice.
for i := range picture {
	picture[i], pixels = pixels[:XSize], pixels[XSize:]
}
```

### Maps（映射）
Maps是一种使用方便且功能强大的内置数据结构，它将一种类型（键）的值与另一种类型（元素或值）的值相关联。键可以是定义了相等运算符的任何类型，例如整数，浮点数和复数，字符串，指针，接口（只要动态类型支持相等运算），结构和数组。 切片不能用作映射键，因为切片没有定义相等性。与切片一样，Maps保存对基础数据结构的引用。如果将Maps传递给更改Maps内容的函数，则更改将在调用者中可见。

可以使用通常的复合字面量语法和冒号分隔的键值对来构造maps，因此在初始化期间很容易构建映射。

```
var timeZone = map[string]int{
    "UTC":  0*60*60,
    "EST": -5*60*60,
    "CST": -6*60*60,
    "MST": -7*60*60,
    "PST": -8*60*60,
}
```

分配并获取map值看起来切片和数组并无二致，只是索引键无需为整型数字。

```
offset := timeZone["EST"]
```
尝试使用map中不存在的键获取映射值将返回映射值类型的零值。例如，如果映射包含整数，则查找不存在的键将返回0。可以将集合实现为值类型为bool的映射。将映射值设置为true以将值放入集合中，然后通过简单索引对其进行测试。

```
attended := map[string]bool{
    "Ann": true,
    "Joe": true,
    ...
}

if attended[person] { // will be false if person is not in the map
    fmt.Println(person, "was at the meeting")
}
```
有时你需要将缺失的条目与零值区分开来。是否有“UTC”的条目或是值为0，还是因为它根本不存在map中？你可以多变量赋值的形式来进行区分。

```
var seconds int
var ok bool
seconds, ok = timeZone[tz]
```

显而易见的，这就是被称为“逗号,ok”习惯用法。在这个例子中，如果tz存在，则将适当地设置seconds，并且ok设置为true，如果不存在，seconds将被设置为零，ok为false。 下面这个函数很好的错误处理放在一起：

```
func offset(tz string) int {
    if seconds, ok := timeZone[tz]; ok {
        return seconds
    }
    log.Println("unknown time zone:", tz)
    return 0
}
```

要测试map中键的存在性而不担心实际值，可以使用空白标识符（_）代替值的常用变量。

```
_, present := timeZone[tz]
```
要删除映射项，请使用内置函数delete，其参数是映射变量和要删除的键。即使地图上已经没有对应的key，这样做也是安全的。

```
delete(timeZone, "PDT")  // Now on Standard Time
```

### Printing（打印）
Go中的格式化打印的风格类似于C的printf系列，但功能更丰富，更通用。这些函数位于fmt包中，并具有大写名称：fmt.Printf，fmt.Fprintf，fmt.Sprintf等。字符串函数（Sprintf等）返回一个字符串而不是填充提供的缓冲区。

你无需提供格式化的字符串。对于Printf，Fprintf和Sprintf中的每一个函数都有对应的另一对函数，例如Print和Println。这些函数不采用格式化字符串，而是为每个参数生成默认格式。Println版本还在参数之间插入空格并在输出中附加换行符，而Print版本仅在两侧的操作数都是字符串时才添加空格。在此示例中，每行产生相同的输出。

```
fmt.Printf("Hello %d\n", 23)
fmt.Fprint(os.Stdout, "Hello ", 23, "\n")
fmt.Println("Hello", 23)
fmt.Println(fmt.Sprint("Hello ", 23))
```

格式化的打印函数fmt.Fprint和friends将任何实现io.Writer接口的对象作为第一个参数; 变量os.Stdout和os.Stderr是熟悉的实例。

事情从这里开始跟C语言不同。首先，数字格式，像％d，并不接受正负号和大小的标记; 相反，打印例程使用参数的类型来决定这些属性。

```
var x uint64 = 1<<64 - 1
fmt.Printf("%d %x; %d %x\n", x, x, int64(x), int64(x))
```
打印出

```
18446744073709551615 ffffffffffffffff; -1 -1
```

如果您只想要默认转换，例如十进制整数，则可以使用catchall格式％v（用于“value”）; 结果跟Print和Println产生的结果一致。而且，该格式可以打印任何值，甚至是数组，切片，结构体和映射。以下是上一节中定义的时区映射的print语句。

```
fmt.Printf("%v\n", timeZone)  // or just fmt.Println(timeZone)
```
输出：

```
map[CST:-21600 PST:-28800 EST:-18000 UTC:0 MST:-25200]
```
对于映射来说，可以以任何顺序输出键指对。打印结构体时，修改后的格式`％+v`使用其名称注释结构的字段，对于任何值，备用格式`%#v`以完整的Go语法打印值。

For maps the keys may be output in any order, of course. When printing a struct, the modified format %+v annotates the fields of the structure with their names, and for any value the alternate format %#v prints the value in full Go syntax.

```
type T struct {
    a int
    b float64
    c string
}
t := &T{ 7, -2.35, "abc\tdef" }
fmt.Printf("%v\n", t)
fmt.Printf("%+v\n", t)
fmt.Printf("%#v\n", t)
fmt.Printf("%#v\n", timeZone)
```
打印出

```
&{7 -2.35 abc   def}
&{a:7 b:-2.35 c:abc     def}
&main.T{a:7, b:-2.35, c:"abc\tdef"}
map[string]int{"CST":-21600, "PST":-28800, "EST":-18000, "UTC":0, "MST":-25200}
```

（注意＆符号）当应用于string或[]byte类型的值时，引用字符串格式也可通过％q获得。 如果可能，备用格式%#q将使用反引号。%q格式也适用于整型和runes，产生单引号rune常量。）此外，`%x`适用于字符串，字节数组和字节切片以及整数，生成十六进制字符串，带有空格的格式`(％ x)`中，它会在字节之间放置空格。

另一个很方便的格式是%T，它打印一个值的类型。

```
fmt.Printf("%T\n", timeZone)
```

打印出

```
map[string]int
```

如果要控制自定义类型的默认格式，则只需要在类型上定义带有string签名的String()方法即可。对于简单类型T，可能看起来像这样。

```
func (t *T) String() string {
    return fmt.Sprintf("%d/%g/%q", t.a, t.b, t.c)
}
fmt.Printf("%v\n", t)
```

打印出

```
7/-2.35/"abc\tdef"
```
（如果需要打印类型T的值以及需要指向T的指针，String的接收器必须是值类型，此示例使用指针，因为这对于结构类型更有效和惯用。请参阅一下有关[指针接收器-值接收器](https://golang.org/doc/effective_go.html#pointers_vs_values)以获取更多信息。）

我们的String方法能够调用Sprintf，因为打印程序是完全可重入的，并且可以这种方式包装。但是，对于这种方式，有一个重要的细节需要明白：不要将调用Sprintf的String方法构造成无穷递归。如果Sprintf尝试直接调用并将接收者作为字符串打印，就会导致再次调用该方法，就陷入了无限递归。 正如这个例子所示，这是一个常见且容易犯的错误。

```
type MyString string

func (m MyString) String() string {
    return fmt.Sprintf("MyString=%s", m) // Error: will recur forever.
}
```
这很容易修复，把它转换为没有这个方法的基础string类型。

```
type MyString string
func (m MyString) String() string {
    return fmt.Sprintf("MyString=%s", string(m)) // OK: note conversion.
}
```
在[初始化](https://golang.org/doc/effective_go.html#initialization)章节，我们将会看到另一种避免这种无限递归的技术。

另一种打印技术是将打印程序的参数直接传递给另一个这样的例程。Printf函数的签名使用类型`... interface{}`作为其最终参数，以指定在格式之后可以出现（任意类型）任意数量的参数。

```
func Printf(format string, v ...interface{}) (n int, err error) {
```

在函数Printf中，v的作用类似于[]interface{}类型的变量，但是如果它被传递给另一个可变参数函数，它就像一个常规的参数列表。这是我们上面使用的函数log.Println的实现。它将其参数直接传递给fmt.Sprintln以进行实际的格式化。

```
// Println prints to the standard logger in the manner of fmt.Println.
func Println(v ...interface{}) {
    std.Output(2, fmt.Sprintln(v...))  // Output takes parameters (int, string)
}
```
我们在对Sprintln的嵌套调用中写入`...`后告诉编译器将v视为参数列表，否则它只会将v作为单个切片参数传递。

关于print，还有比我们在这里介绍的更多的内容。有关详细信息，请参阅package fmt的godoc文档。

顺便说一下，`...`参数可以是特定类型，例如`...int`用于选择整数列表最小值的Min函数：

```
func Min(a ...int) int {
    min := int(^uint(0) >> 1)  // largest int
    for _, i := range a {
        if i < min {
            min = i
        }
    }
    return min
}
```

### Append（附加）
现在我们需要解释附加内置函数的设计所缺少的部分。 append的签名与上面的自定义Append函数不同。 原理上，它是这样的：

```
func append(slice []T，elements ...T) []T
```

其中T是任何给定类型的占位符。你实际上不能在Go中编写一个函数，而类型T由调用者确定。 这就是为什么追加内置类型：因为它需要编译器的支持。

附加的作用是将元素追加到切片的末尾并返回结果。结果需要返回，因为与我们手写的Append一样，底层数组可能会发生变化。 这个简单的例子。
```
x := []int{1,2,3}
x = append(x, 4, 5, 6)
fmt.Println(x)
```

打印[1 2 3 4 5 6]，所以append跟Printf有点类似，可以接收任意数量的参数。

但是假如我们想让append跟我们的Append做的一样，添加一个slice到另一个slice的末尾呢？答案也很简单：在调用点使用`...`。这个片段跟上文一样输出相同的结果。

```
x := []int{1,2,3}
y := []int{4,5,6}
x = append(x, y...)
fmt.Println(x)
```

没有`...`,编译器将会出错。因为y不是整型。

## 参考
<https://golang.org/doc/effective_go.html>



