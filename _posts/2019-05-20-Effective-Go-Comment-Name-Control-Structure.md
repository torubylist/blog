---
layout:     post
title:      "高效Go语言-注释-名字-控制结构"
subtitle:   " \"Effective-Comment-Names-Control-Structure\""
date:       2019-05-23 20:00:00
author:     "会飞的蜗牛"
header-img: "img/effective-go.jpg"
tags:
    - Comment
    - Names
    - Control Structure
    - Effective-go

---


[TOC]

---
## 简介
Go是一门全新的语言．虽然他从其他语言中借鉴了一些特性，但是Go语言的编程方式与其他语言是有本质区别的．如果只是简单的将C++或者Java的代码翻译成Go代码是不可能得到最优的Go代码的．Java程序员用的是Java的编程思维，并不是Go的思维方式．如果采用Go的思维方式，一个问题可能有完全不同的解法．因此，如果要真正用好Go，首先要理解它的语言特性和设计思想．另外，还要知道Go的编程风格，例如命名方式，格式化，程序结构等，采用通用的编程方式有利于和其他程序员交流．

### 例子
Go源代码不仅包含了核心库的实现，还有很多如何使用语言的例子。如果在使用Go的过程中遇到了问题，或者想了解某些库的内部工作机制，可以直接参考源代码的实现。他们或许可以给你想要的答案或者设计思想。

## 格式化
格式化问题是最具争议性但最不重要的问题。人们可以适应不同的风格的代码，但如果他们可以不必这样做，那就更好了。如果每个人都遵循相同的风格，那么专注于代码格式的时间就会减少。问题是如何在没有长久约定的风格指南的情况下去接近这个理想情况。

使用Go，我们采取了一种不寻常的方法，让机器处理大多数格式问题。gofmt（也可作为go fmt，在包级别而不是源文件级别运行）读取Go程序并以标准样式的缩进和垂直对齐方式修改该源文件，甚至在必要时重新格式化注释。如果您想知道如何处理一些新的布局情况，请运行gofmt; 如果运行过程中出现错误，请重新组织您的代码（或提交有关gofmt的错误），而不要临时解决它。

例如，没有必要花太多的事件去给结构对齐其注释。Gofmt将会处理它。以下声明：

```
type T struct {
    name string // name of the object
    value int // its value
}
```

gofmt对齐列:

```
type T struct {
    name    string // name of the object
    value   int    // its value
}
```

标准库里的所有代码均被gofmt格式化了。

一些格式化详情. 很简洁:

>缩进
>
>我们使用制表符进行缩进，默认情况下gofmt会使用它们。仅在必要时使用空格。
>
>行宽
>
>Go没有行长度限制。不要担心打孔卡溢出。如果线条感觉太长，请将其包裹并用额外的标签缩进。
>
>括弧
>
>Go只需要比C和Java更少的括号：控制结构（if，for，switch）的语法中没有括号。 此外，运算符优先级层次更短更清晰，因此与其他语言不同，这就说明空格的含义。
	
>  `x<<8 + y<<16`


## 注释
Go提供C风格的/**/块注释和C ++风格的//行注释。行注释是常态，块注释主要为包注释，但在表达式中也很有用或是注释大块代码。

程序和Web服务器-godoc处理Go源文件以提取有关包内容的文档。在顶级声明之前出现的注释（没有中间换行符）将与声明一起提取，作为项目的解释性文本。这些评论的性质和风格决定了godoc产生的文档的质量。

每个包都应该有一个包注释，在package子句之前有一个块注释。对于多文件包，包注释只需要存在于一个文件中，任何一个都可以。包注释应该介绍包的概要，并提供与整个包相关的信息。它将首先出现在godoc页面上，并应设置下面的详细文档。

```
/*
Package regexp implements a simple library for regular expressions.

The syntax of the regular expressions accepted is:

    regexp:
        concatenation { '|' concatenation }
    concatenation:
        { closure }
    closure:
        term [ '*' | '+' | '?' ]
    term:
        '^'
        '$'
        '.'
        character
        '[' [ '^' ] character-ranges ']'
        '(' regexp ')'
*/

package regexp
```

如果是简单的包，那么包的注释就比较简单.

```
// Package path implements utility routines for
// manipulating slash-separated filename paths.

```

注释不需要比如像星条幅这样额外的格式。生成的输出甚至可能不会以固定宽度的字体显示，因此不依赖于alignment-godoc的间距，就像gofmt一样处理它。注释是未解释的纯文本，因此HTML和其他注释（如_this_）将逐字重现，不应该被代码使用。godoc做的一个调整是以固定宽度字体显示缩进文本，这适用于程序片段。fmt软件包的包注释使用了这个效果。

根据上下文，godoc可能不会重新格式化注释，因此请确保它们看起来很好：使用了正确的拼写，标点符号和句子结构，长行折叠等等。

在包中，紧接在顶级声明之前的任何注释都充当该声明的doc注释。程序中的每个导出（大写）名称都应具有doc注释。

Doc注释最好是使用完整的句子，以适应各种展示的宽度。第一句应该是一个句子摘要，以该声明的名称开头。

```
// Compile parses a regular expression and returns, if successful,
// a Regexp that can be used to match against text.
func Compile(str string) (*Regexp, error) {
```

如果每个doc注释都以它描述的项的名称开头，则可以使用go工具的doc子命令并通过grep运行输出。想象一下你不记得名称“Compile”，但是正在寻找正则表达式的解析函数，然后你运行了下面的命令。

`$ go doc -all regexp | grep -i parse`

如果包中的所有doc注释开始都是“This function ...”，grep将无法帮助你找到这个名字。 但是因为软件包以名称开始的文档注释，你会看到类似这样的内容，它就会帮你想起你正在寻找的单词。

```
$ go doc -all regexp | grep -i parse
    Compile parses a regular expression and returns, if successful, a Regexp
    MustCompile is like Compile but panics if the expression cannot be parsed.
    parsed. It simplifies safe initialization of global variables holding
$
```

Go的声明语法允许对声明进行分组。单个doc注释可以引入一组相关的常量或变量。既然整个代码声明已经展现了，这样的注释往往是极其敷衍的。

```
// Error codes returned by failures to parse an expression.
var (
    ErrInternal      = errors.New("regexp: internal error")
    ErrUnmatchedLpar = errors.New("regexp: unmatched '('")
    ErrUnmatchedRpar = errors.New("regexp: unmatched ')'")
    ...
)
```

分组还可以表示各个项之间的关系，例如一组受互斥锁保护的变量。

```
var (
    countLock   sync.Mutex
    inputCount  uint32
    outputCount uint32
    errorCount  uint32
)
```

## 名字
与任何其他语言一样，名称在Go语言中很重要。它们甚至具有语义效果：在包外，名字的可见性取决于其第一个字符是否为大写。 因此，花一点时间讨论Go程序中的命名约定是值得的。


### 包名

导入包时，包名称将成为内容的访问者。

```
import "bytes"
```

导入包之后,我们就可以谈谈`bytes.Buffer`。 如果每个使用该包的人都可以使用相同的名称来引用其内容，这意味着这个包的名称应该是好的：简短而又简洁，令人回味。 按照惯例，软件包被赋予小写单字名称，应该不需要下划线或者混合使用。 简而言之，因为每个使用你的包的人都会输入这个名字。 并且不要担心其他先行包的碰撞，包名称只是为导入设置的默认名称。 它不需要在所有源代码中都是唯一的，并且在极少数情况下，导入包可以选择设置一个在本地使用的其他名称。 在通常情况下，混淆很少见，因为导入中的文件名确定了正在使用的包。

另一个约定是包名是基于其源目录的基本名字; `src/encoding/base64`中的包被导入为`encoding/base64`，但名称为`base64`，既不是`encoding_base64`,也不是`encodingBase64`。

包的导入器将使用该名称来引用其内容，因此包中的导出名称可以基于此来避免重复。（不要使用`import .`表示法，它确实可以简化那些必须在包外运行的测试，否则就应该尽量避免。）例如，`bufio`包中的缓冲读取器类型为`Reader`，而不是`BufReader`，因为用户将其视为`bufio.Reader`，这是一个清晰，简洁的名称。 此外，由于导入的实体始终使用其包名称进行寻址，因此`bufio.Reader`不会与`io.Reader`冲突。 类似地，创建`ring.Ring`的新实例的函数-这是Go中构造函数的定义-通常称为`NewRing`，但由于`Ring`是包导出的唯一类型，并且因为包被称为`ring`，所以它是只叫`New`，这个包的客户端看起来像`ring.New`。使用包结构可以帮助您选择一个好的名称。

另一个简短的例子是`once.Do`; `once.Do(setup)`读起来很好，使用`once.DoOrWaitUntilDone(setup)`并不会更好。一个更长名称不会使事物更具可读性，而有用的文档注释通常比超长名称更有价值。

### Getters
Go语言并没有自动提供`getter`和`setter`函数支持。亲自写一个`getter`和`setter`并没有什么不妥，这通常也是更合适的。但是将`Get`放入`getter`的名字既不是惯用用法也不是必需的。 如果您有一个名为`owner`（小写，未导出）的字段，则`getter`方法应该称为`Owner`（大写，导出），而不是`GetOwner`。使用大写名称进行导出提供了将字段与方法区分开的钩子。 如果需要，`setter`函数可能会被称为`SetOwner`。 这两个名字在实践中都很好读：

```
owner := obj.Owner()
if owner != user {
    obj.SetOwner(user)
}
```

### 接口名字
按照惯例，单方法接口名由方法名称加上-er后缀或类似修改来命名，以构造代理名词：`Reader，Writer，Formatter，CloseNotifier`等。

有许多这样的名称，尊重他们以及他们的函数名称是会很有成效的。`Read，Write，Close，Flush，String`等具有规范的签名和含义。 为避免混淆，请不要让您的方法使用他们中的任意一个作为名称，除非它具有相同的签名和含义。 相反，如果您的类型实现的方法与众所周知类型的方法具有相同的含义，请为其指定相同的名称和签名; 调用字符串转换器方法`String`而不是`ToString`。

### 混合大小写
最后，按照`Go`语言的约定，请使用`MixedCaps`或者`mixedCaps`，而不是下划线线或者多词名字。

## 分号
与`C`一样，`Go`的正式语法使用分号来终止语句，但与`C`语言不同，这些分号不会出现在源代码中。相反，词法分析器使用一个简单的规则在扫描时自动插入分号，因此输入文本大多没有分号。

规则是这样的。如果换行符之前的最后一个标记是一个标识符（包括`int`和`float64`之类的单词），则说明是一个基本语句，例如数字或字符串常量，或者其中一个标记

```
break continue fallthrough return ++ -- ) }
```

词法分析器总是在令牌后插入分号。这可以概括为“如果新行出现在可以结束语句的标记之后，则插入分号”。

在结束括号之前也可以省略分号，所以例如下面的语句就不需要分号。

```
    go func() { for { dst <- <-src } }()
```

通常`Go`程序仅在诸如`for`循环子句之类的地方使用分号，以分隔初始化对象，条件和迭代元素。 如果您在一行里面写多个语句，那么分号也是必需的。

分号插入规则的一个结果是您不能将控制结构`（if，for，switch或select）`的大括号放在下一行。 如果这样做，将在大括号之前插入分号，这可能会导致不好的影响。像这样写代码，

```
if i < f() {
    g()
}
```
而不是这样，

```
if i < f()  // wrong!
{           // wrong!
    g()
}
```

## 控制结构
`Go`的控制结构与`C`的控制结构有关，但在很多的重要方面又有所不同。没有`do`或`while`循环，只是一个更通用的`for`关键词; `switch`更灵活; `if`和`switch`接受类似于`for`的可选初始化语句; `break`和`continue`语句采用可选标签来标识要中断或继续的内容; 并且存在新的控制结构，包括类型`switch`和通信多路复用器，`select`。 语法也略有不同：没有括号，并且主体必须始终以大括号分隔。

### If

```
if x > 0 {
    return y
}
```

强制括号鼓励在多行上写简单的`if`语句。无论怎样，这样的做法都很好，特别是当正文包含一个控制语句，如`return`或`break`时。

由于`if`和`switch`接受初始化语句，因此通常会看到用于设置局部变量的语句。

```
if err := file.Chmod(0664); err != nil {
    log.Print(err)
    return err
}
```
在`Go`语言库中，您会发现当`if`语句没有流入下一个语句时-即，body以`break，continue，goto`或`return`结束,则省略了不必要的`else`。

```
f, err := os.Open(name)
if err != nil {
    return err
}
codeUsing(f)
```

这是一个常见的示例，其中代码必须对一系列错误条件做出防卫。一个有继的控制流在页面上运行，则代码可读性好，从而消除尽可能出现的错误情况。由于错误情况倾向于在`return`语句中结束，继而生成的代码不需要else语句。

```
f, err := os.Open(name)
if err != nil {
    return err
}
d, err := f.Stat()
if err != nil {
    f.Close()
    return err
}
codeUsing(f, d)
```

### 重新声明和重新分配
旁白：上一节中的最后一个示例演示了:=简短声明的工作原理。调用os.Open的声明，

```
f, err := os.Open(name)
```

这个语句申明了两个变量，f和err。几行之后，调用f.Stat读取。

```
d, err := f.Stat()
```

看起来好像是声明了d和err两个变量。但请注意，err出现在两个语句中，但是这种重复是合法的，err由第一个语句声明，在第二个语句中仅做了重新赋值。这意味着对f.Stat的调用使用了上面声明的现有错误变量，并给它一个新值。

在一个:=声明中，即使变量v已经被声明了，也可能出现变量v，前提是：

1. 此声明与v的现有声明在同一作用域范围内（如果v已在外部作用域中声明，声明将创建一个新变量）
2. 初始化中的相应值可分配给v，但声明中至少有一个其他变量是全新的声明。
3. 这种不寻常的属性是纯粹的实用主义，因此很容易使用单个错误值，例如，在长if-else链中。你会看到它经常使用。

值得注意的是，在Go中，函数参数和返回值的范围与函数体相同，即使它们在包围体的括号之外的词法上出现。

### For
`Go`语言中的`for`循环类似于`C`，但与`C`不同。 它统一了`for`和，`while`，而且没有`do-while`。 有三种形式，其中只有一种有分号。

```
// Like a C for
for init; condition; post { }

// Like a C while
for condition { }

// Like a C for(;;)
for { }
```

短变量声明可以很容易地在循环中声明索引变量。

```
sum := 0
for i := 0; i < 10; i++ {
    sum += i
}
```

如果你在遍历一个`array，slice，string，map`或者从通道读取数据，可以用`range`控制循环。

```
for key, value := range oldMap {
    newMap[key] = value
}
```

如果你仅仅需要`range`的第一个项(key或者index)，可以丢掉第二个项：

```
for key := range m {
    if key.expired() {
        delete(m, key)
    }
}
```
如果您只需要range中的第二项（value），请使用空白标识符（下划线）放弃第一项：

```
sum := 0
for _, value := range array {
    sum += value
}
```
空白标识符有许多用途，正如后文所述。

对于字符串，该`range`对您来说更有用，通过解析UTF-8来分解单个Unicode代码点。错误的编码消耗一个字节并产生替换符文U+FFFD。 （rune（关联的内置类型）是单个Unicode代码点的Go术语。有关详细信息，请参阅语言规范。）

```
for pos, char := range "日本\x80語" { // \x80 is an illegal UTF-8 encoding
    fmt.Printf("character %#U starts at byte position %d\n", char, pos)
}
```

上面的循环打印出：

```
character U+65E5 '日' starts at byte position 0
character U+672C '本' starts at byte position 3
character U+FFFD '�' starts at byte position 6
character U+8A9E '語' starts at byte position 7
```

最后，Go没有逗号运算符，++和--语句都不是表达式。 因此，如果你想在a中运行多个变量，你应该使用并行赋值（虽然这里不能使用++和--）。

```
// Reverse a
for i, j := 0, len(a)-1; i < j; i, j = i+1, j-1 {
    a[i], a[j] = a[j], a[i]
}
```

### Switch
Go的switch比C语言的更通用。表达式不必是常量或者是整数，在找到匹配项之前从上到下评估案例，如果switch没有表达式，则默认将其置为true。因此，将if-else-if-else链作为switch编写是可能的，也是Go语言的习惯用法。

```
func unhex(c byte) byte {
    switch {
    case '0' <= c && c <= '9':
        return c - '0'
    case 'a' <= c && c <= 'f':
        return c - 'a' + 10
    case 'A' <= c && c <= 'F':
        return c - 'A' + 10
    }
    return 0
}
```

不支持自动fall-through，但是支持逗号分隔的列表。

```
func shouldEscape(c byte) bool {
    switch c {
    case ' ', '?', '&', '=', '#', '+', '%':
        return true
    }
    return false
}
```
尽管它们在Go中并不像其他类C语言那样常见，但break语句可用于提前终止switch。 但有时候，有必要打破整个循环，而不是仅仅在switch中，而在Go中，可以通过在循环上放置一个标签并“break”到该标签来实现。 下面的示例显示了这两种用法。

```
Loop:
	for n := 0; n < len(src); n += size {
		switch {
		case src[n] < sizeOne:
			if validateOnly {
				break
			}
			size = 1
			update(src[n])

		case src[n] < sizeTwo:
			if n+1 >= len(src) {
				err = errShortInput
				break Loop
			}
			if validateOnly {
				break
			}
			size = 2
			update(src[n] + src[n+1]<<shift)
		}
	}
```	

当然，continue语句也接受可选标签，但它仅适用于循环。

要结束此部分，下文是两个使用switch语句的字节切片的比较程序：

```
// Compare returns an integer comparing the two byte slices,
// lexicographically.
// The result will be 0 if a == b, -1 if a < b, and +1 if a > b
func Compare(a, b []byte) int {
    for i := 0; i < len(a) && i < len(b); i++ {
        switch {
        case a[i] > b[i]:
            return 1
        case a[i] < b[i]:
            return -1
        }
    }
    switch {
    case len(a) > len(b):
        return 1
    case len(a) < len(b):
        return -1
    }
    return 0
}
```

### Type switch
switch还可用于发现接口变量的动态类型。这种类型的switch使用括号内关键字类型断言的语法。如果switch在表达式中声明了一个变量，则该变量将在每个子句中具有相应的类型。在这种情况下重用名称也是惯用的，实际上声明了一个具有相同名称但在每种情况下具有不同类型的新变量。

```
var t interface{}
t = functionOfSomeType()
switch t := t.(type) {
default:
    fmt.Printf("unexpected type %T\n", t)     // %T prints whatever type t has
case bool:
    fmt.Printf("boolean %t\n", t)             // t has type bool
case int:
    fmt.Printf("integer %d\n", t)             // t has type int
case *bool:
    fmt.Printf("pointer to boolean %t\n", *t) // t has type *bool
case *int:
    fmt.Printf("pointer to integer %d\n", *t) // t has type *int
}
```

## 参考
<https://golang.org/doc/effective_go.html>



