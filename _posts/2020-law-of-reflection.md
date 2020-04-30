---
layout:     post
title:      "The Law of Reflection"
subtitle:   " \"Go语言反射定律\""
date:       2020-04-30 20:00:00
author:     "会飞的蜗牛"
header-img: "img/k8s.jpeg"
tags:
    - Golang
    - reflection

---

## 简介
在计算机领域里，反射是指程序检查自身结构的能力，尤其是通过类型自定义的结构体。它是元编程的一种形式，虽然这种能力很强大，但同时也会让我们在使用的时候感到困惑。

本文，我将尝试解释一下反射在Go语言中是如何工作的。每种语言的反射模型是不一样的（还有很多语言根本就不支持反射）。由于本文的主题是go，所以接下来的内容中的“反射”就是指代“Go语言中的反射”。

## 类型和接口
因为反射是建立在类型系统之上，所以让我们来复习下 Go 中的类型。

Go 是静态类型语言。每个变量都有一个静态类型，那就是编译的时候只有一个已知的固定类型：int，float32, *MyType, []byte等等。假如我们声明：

```
type MyInt int

var i int
var j MyInt
```

那么，变量i的静态类型就是int，j 的类型则是 MyInt。i 和 j 拥有不同的静态类型，尽管他们的底层类型是一致的，他们在没有强制转换的情况下是不能互相赋值的。

一个非常重要的类型就是接口类型，代表着一组方法集。一个接口变量能够存储任意的真实类型（非接口类型）值，只要这个值实现了接口中的方法集。一个比较有名的例子就是 io 包的 io.Reader 和 io.Writer

```
// Reader is the interface that wraps the basic Read method.
type Reader interface {
    Read(p []byte) (n int, err error)
}

// Writer is the interface that wraps the basic Write method.
type Writer interface {
    Write(p []byte) (n int, err error)
}
```

任意实现了Read（或Write）方法的类型都可以说实现了 io.Reader/io.Writer 接口。在本文讨论中，这意味着可以给 io.Reader 类型的变量赋任何实现了 Read 方法类型的值。

```
var r io.Reader
r = os.Stdin
r = bufio.NewReader(r)
r = new(bytes.Buffer)
// and so on
```

这里很重要的一点是，不管变量 r 持有的真实类型是什么，r 的类型始终是 io.Reader： 这就是说 Go 是静态类型语言， r 的静态类型是 io.Reader。

一个极其重要的接口类型的例子是 空接口：

```
interface{}
```

这就表示一个空的方法集，任何类型的值都有大于等于零个方法，所以他们都实现了该接口。

一些人会说Golang的接口类型是动态类型，这是不对的。一个接口类型的变量始终有相同的静态类型，就是接口。即使这个接口变量在运行时的真实值的类型是不断变化的，这个值的真实类型也必须一直满足这个接口。

由于反射和接口的紧密关系，我们必须要对这一切有个精准的认知。

## 接口的表征
Russ Cox写了一篇关于接口值表征的详细的[博客](http://research.swtch.com/2009/12/go-data-structures-interfaces.html)。在这里，我们不再重复所有的观点，只做一个简单的总结。

一个接口类型的变量实际存储了一个 pair：赋给该变量的实际值和该值的类型描述。说的更准确一点，值是实现该接口的底层确切的数据项，类型描述了该项的完整类型。例如：

```
var r io.Reader
tty, err := os.OpenFile("/dev/tty", os.O_RDWR, 0)
if err != nil {
	return nil, err
}

r = tty

```
r 包含了 (value, type) 对，即(tty, *os.File). 这里类型 *os.File 实现的方法不仅仅只是 Read；尽管接口值只允许我们访问 Read方法，但是这个值包含了该类型所有的信息。这就是为什么我们可以做下面的事情：

```
var w io.Writer
w = r.(io.Writer)
```

这里是一个类型断言；这个断言用于判断r 内部的变量是否也实现了 io.Writer 接口，如果是，可以把他赋值给 w。赋值之后， w 也会包含一个pair（tty，*os.File）。这个值跟 r持有的pair是一摸一样的。接口的静态类型决定了接口变量可以调用哪些方法，尽管里面的真实类型拥有更大的方法集。

```
var w io.Writer
w = r.(io.Writer)
```

接着讲，我们可以这样做：

```
var empty interface{}
empty = w
```

空接口值 empty 再次包含了相同的pair, (tty, *os.File)。 这很好理解：空接口能够持有任何值，并且包含所需的关于该值的所有类型信息。

（这里我们不需要类型断言，因为众所周知，w能够满足空接口。在前面的例子里，把一个io.Reader 赋值给 io.Writer类型，需要一个明确的断言，因为Writer方法不是Reader类型的发发集合的子集。）

一个重要的细节是，接口内部的值总是 （value， concrete type），而不是（value， interface type）这样的类型。接口不拥有接口值。

下面开始讨论反射。

## 反射第一定律
### Reflection goes from interface value to reflection object.
从最基础的层面上来说，反射只是一种机制，通过这种机制来校验存储在接口类型里面的真实类型和值。首先，我们需要了解reflect包里面的两种类型：Type 和 Value，通过这两种类型，可以访问接口变量的内容。其次是两个简单的函数，分别是 reflect.TypeOf 和 reflect.ValueOf，分别从接口值里面获取reflect.Type 和 reflect.Value。（同样的，可以通过reflect.Value获取 reflect.Type， 但是，目前我们还是分开来描述两个概念）。

先从TypeOf开始：

```
package main

import (
    "fmt"
    "reflect"
)

func main() {
    var x float64 = 3.4
    fmt.Println("type:", reflect.TypeOf(x))
}
```

输出:

```
type: float64
```

你可能会有点疑惑，接口在哪里。因为程序传递给 reflect.Type的是 float64类型的值x，不是接口值。我们来看看godoc里面所描述的。 reflect.TypeOf包括一个空接口：

```
// TypeOf returns the reflection Type of the value in the interface{}.
func TypeOf(i interface{}) Type
```

当我们调用reflect.TypeOf(x)，x 首先存储在空接口中，作为参数传递。reflect.TypeOf 解包 空接口，从而拿到类型信息。

同理，reflect.ValueOf 函数从interface变量中获取值：

```
var x float64 = 3.4
fmt.Println("value:", reflect.ValueOf(x).String())
```
打印：

```
value: <float64 Value>
```

(这里显示地调用String方法是因为 fmt包默认会深入到reflect.Value变量的内部获取确定的值， 而 String 方法不会)

reflect.Type和reflect.Value都有很多的方法来帮助我们校验和操作该值。其中一个总要的例子就是 Value 有一个 Type方法，返回 reflect.Value的类型。另一个是 Type 和 Value类型都有 Kind方法，并且返回一个常量，用来表示存储条目的类型。例如： Uint，Float64方法可以让我们获取到里面的值：

```
var x float64 = 3.4
v := reflect.ValueOf(x)
fmt.Println("type:", v.Type())
fmt.Println("kind is float64:", v.Kind() == reflect.Float64)
fmt.Println("value:", v.Float())
```

打印
```
type: float64
kind is float64: true
value: 3.4
```

这里也有一些方法类似 SetInt 和 SetFloat，但是在使用他们之前，我们需要理解可设置性。后文的反射第三定律将会详细讨论这点。

反射库有几个方法特性得单拎出来仔细讨论。第一，为了保持API的简练，Value的“getter”和“setter”方法基于能持有该类型的最大类型，例如对所有的signed int类似是int64，也就是说int方法返回的总是一个Int64类型的值。SetInt的值总是被当作int64；有时可能需要显式的转换为真实的类型。

```
var x uint8 = 'x'
v := reflect.ValueOf(x)
fmt.Println("type:", v.Type())                            // uint8.
fmt.Println("kind is uint8: ", v.Kind() == reflect.Uint8) // true.
x = uint8(v.Uint())                                       // v.Uint 
```
返回uint64.

第二个特性是反射对象的 Kind方法描述的是底层类型，不是静态类型。如果一个反射对象包含用户自定义的整型，例如：

```
type MyInt int
var x MyInt = 7
v := relect.ValueOf(x)

```
v 变量的 Kind方法返回的是 reflect.Int，尽管x的静态类型是 MyInt，不是int。换句话说就是Type可以区分 int和MyInt，而Kind不能区分。



## The second law of reflection
### 2. Reflection goes from reflection object to interface value.


就像物理意义的反射，Go 的反射也可以反向生成。

基于 reflect.Value，我们可以通过使用Interface 方法来恢复一个接口值。实际上，该方法打包类型和值信息进到一个接口表征，返回结果。

```
// Interface returns v's value as an interface{}.
func (v Value) Interface() interface{}

```
输出：

```
y := v.Interface().(float64) // y will have type float64.
fmt.Println(y)
```

打印反射对象 v 中保存的 float64 类型变量的值。

我们甚至可以更进一步，fmt.Println，fmt.Printf等方法的参数都是作为空interface值传递的。然后fmt包在内部会解开这个接口，正如前面的例子一样。因此，
正确打印 reflect.Value 的值，要做的就是把 Interface 方法传递给格式化打印程序。

```
fmt.Println(v.Interface())
```

（那为什么不是fmt.Println(v)呢）？因为v是 reflect.Value； 我们想要它持有的真实值。由于我们的值是 float64 类型，我甚至可以用一个 float 格式来打印：

```
fmt.Printf("value is %7.1e\n", v.Interface())
```
得到：

```
3.4e+00
```

再次说明，这里没有必要做类型断言，将 v.Interface()的值转换成 float64；空接口的值在内部有真实值的类型信息，Printf函数会恢复这个真实类型。

总的来说，Interface方法是 Valueof函数的反向操作，除了 类型一直都是静态 interface{} 类型。

再次重申: Reflection goes from interface values to reflection objects and back again.


## The third law of reflection
3. To modify a reflection object, the value must be settable.

第三定律有点隐晦，不是很好懂。但是如果我们了解了第一定律，那么理解起来就容易多了。

下面是一段有问题的代码：

```
var x float64 = 3.4
v := reflect.Value(x)
v.SetFloat(7.1) // Error: 抛出panic。

panic: reflect.Value.SetFloat using unaddressable value
```

问题不是在于 7.1不能寻址，而是 v 不能被设置。 可设置性是 反射值的一个属性，不是所有的反射 Value都有这个属性。

CanSet 方法返回 Value的可设置性；在下面的例子中：

```
var x float64 = 3.4
v := refelct.ValueOf(x)
fmt.Println("settablitiy of v: ", v.CanSet())
```
打印出：

```
settability of v: false
```

如果在不可设置 Value上调用 Set 方法会报错。但是什么是可设置性？

可设置性有点类似于可寻址，但是更加严格。这种特性使得反射对象可以修改实际存储反射对象的内存。 可设置性决定于反射对象是否持有原始项。当我们说：

```
var x float64 = 3.4
v := reflect.ValueOf(x)
```
我们把x的副本传给 reflect.ValueOf，所以创建的接口值实际上是 x的副本，而不是 x 本身。因此，如下声明：

```
v.SetFloat(7.1)
```

是可以成功的，只是它不会更新 x。即使 v 看起来像是从 x 创建而来。相反，它会更新存储在 reflection 值的x的副本，x 本身并不会受到影响。 这个操作让人困惑，并且没什么意义，所以这是非法操作，可设置性就是用来避免这个问题。

这是不是有点奇怪，但是其实一点也不。想想把 x 传给一个参数：

```
f(x)
```
我们不会期待 f 可以修改 x 的值，因为我们传的是一个副本，而不是 x 本身。如果我们想要修改 x，我们必须传递 x 的地址。即一个指向 x 的指针。

```
f(&x)
```

这很直白， 反射的工作方式跟这个是一样的。 如果我们想要通过反射修改 x的值，那么我们必须传递 一个指针给reflect.ValueOf。 称之为 p。

```
var x float64 = 3.4
p := reflect.ValueOf(&x)
fmt.Println("type of p: ", p.Type())
fmt.Println("settability of p: ", p.Canset())
```
 输出：
 
```
type of p: *float64
settability of p: false
```

反射对象 p 是不可设置的，而且我们要设置的对象也不是 p， 而是 *p。 为了拿到 p 指向的值，我们需要调用Elem 方法。这样简洁的通过指针将值保存到 反射值 v中：

```
v := p.Elem()
fmt.Println("settability of v:", v.CanSet())
```

现在 v 是一个可设置的反射对象，

```
settability of v: true
```

由于它代表 x，我们最终可以使用 v.SetFloat 来修改 x 的值。

```
v.SetFloat(7.1)
fmt.Println(v.Interface())
fmt.Println(x)
```

输出：

```
7.1
7.1
```


反射难以理解，但是它做的
Reflection can be hard to understand but it's doing exactly what the language does, albeit through reflection Types and Values that can disguise what's going on. Just keep in mind that reflection Values need the address of something in order to modify what they represent.

## Structs
在我们之前的例子中，v 本身不是指针类型。这种情况下，更普遍的方式是使用反射修改 结构体的字段。只要我们拥有结构体的地址。我们就可以 修改结构体的字段。

这里有一个简单的例子来分析结构体值，t。因为我们想要修改结构体的字段，所以我们通过 结构体地址创建一个反射对象。然后我们通过设置 typeOfT类型为reflection Type，遍历它的字段，注意我们可以通过Field导出每一个字段的名字，但是这些字段本身是 reflect.Value类型。


Here's a simple example that analyzes a struct value, t. We create the reflection object with the address of the struct because we'll want to modify it later. Then we set typeOfT to its type and iterate over the fields using straightforward method calls (see package reflect for details). Note that we extract the names of the fields from the struct type, but the fields themselves are regular reflect.Value objects.

```
type T struct {
    A int
    B string
}
t := T{23, "skidoo"}
s := reflect.ValueOf(&t).Elem()
typeOfT := s.Type()
for i := 0; i < s.NumField(); i++ {
    f := s.Field(i)
    fmt.Printf("%d: %s %s = %v\n", i,
        typeOfT.Field(i).Name, f.Type(), f.Interface())
}
```

输出

```
0: A int = 23
1: B string = skidoo
```

这里关于可设置性只有一点需要补充：T的字段名必须是大写的，因为只有可导出的字段才能设置。

因为s包含一个可设置的反射对象，我们可以修改结构体的值。

```
s.Field(0).SetInt(77)
s.Field(1).SetString("Sunset Strip")
fmt.Println("t is now", t)
```

输出:

```
t is now {77 Sunset Strip}
```

如果我们稍微修改下程序，s 创自 t，而不是&t，则调用 SetInt 和 SetString 不会成功。

## 总结
再次重申反射定律

反射-从接口值到反射对象
反射-从反射对象到接口值
要修改反射对象，这个值必须是可设置的。

一旦你理解了反射定律，那么使用起来就更加容易，尽管它依然优点隐晦。这个工具功能强大，应该尽量避免使用，除非确实需要。

反射还有很多其他的功能这里没有涉及到，通过通道发送和接收，分配内存，使用切片和映射，调用方法和函数。但是此文已经足够长了，我们后面再接着讨论这些尚未涉及的话题。



