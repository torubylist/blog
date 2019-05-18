---
layout:     post
title:      "Errors are values"
subtitle:   " \"错误也是值\""
date:       2019-05-18 18:00:00
author:     "会飞的蜗牛"
header-img: "img/errors_are_values.jpg"
tags:
    - Errors
---


Go程序员，尤其是那些刚接触这门计算机语言的人们，常见的讨论点就是如何进行错误处理。
 
```
if err != nil {
    return err
}
```

我们最近扫描了所有的开源项目，发现上面的错误处理大概每页只出现一到两次，比你以为的要少的多。尽管如此，如果人们一直认为必须像下文这样处理错误，那肯定是有问题的，这时候，Go本身就变成了最明显的目标。

```
if err != nil
```

这是不幸的，容易误导的，并且也是容易纠正的。也许正在发生的事情是Go的新手程序员问“如何处理错误？”，学习这种模式，然后就在那里停下来了。在其他语言中，可以使用try-catch块或其他此类机制来处理错误。因此，很多程序员认为，当我需要在旧语言中使用try-catch时，我就在Go中输入`err != nil`。随着时间的推移，你的Go代码中就集结了许多这样的代码片段，这让人感觉Go很笨拙。

无论这种解释是否合理，很明显这些Go程序员错过了关于错误的基本观点：错误也是值。

值是可以被编程的，既然错误也是值，因此错误也是可以被编程。

当然，一种常见的涉及到错误值的声明就是测试它是否为nil，但是还有很多其他的事情可以通过错误值来处理。这些其他值的应用可以帮助我们的代码写的更好，消除大量此类error处理模版代码的出现。

这里有一个来自bufio包的Scanner类型的简单示例。它的Scan方法执行底层I/O，这当然会导致错误。 然而，Scan方法根本不会暴露错误。相反，它返回一个布尔值和一个单独的方法，这个方法在扫描结束时运行，报告是否发生了错误。客户端代码如下所示：

```
scanner := bufio.NewScanner(input)
for scanner.Scan() {
    token := scanner.Text()
    // process token
}
if err := scanner.Err(); err != nil {
    // process the error
}
```

当然，这里还是有一个error检查，但是仅出现了一次。Scan方法也可以如下定义，

```func (s *Scanner) Scan() (token []byte, error)```

这样，客户端代码就变成了（取决于token的获取）：

```
scanner := bufio.NewScanner(input)
for {
    token, err := scanner.Scan()
    if err != nil {
        return err // or maybe break
    }
    // process token
}
```

两种代码看起来并没有太大的不同，但有一个重要的区别。在此代码中，客户端必须在每次迭代时检查错误，但在真正的Scanner API中，错误处理是从真实API元素抽象出来的，而真实API元素正在迭代令牌。使用真API，客户端的代码因此看起来更自然：直到循环完成，然后担心错误。错误处理不会掩盖控制流。

当然，正在发生的事情是，只要Scan遇到I/O错误，它就会记录它并返回false。一个单独的方法Err在客户端询问时报告错误值。虽然这是微不足道的，但它与在任何地方放置错误或要求客户在每个令牌之后检查错误不同。它是用错误值编程的。简单的编程，是的，但仍然需要编程。

```if err != nil```


值得强调的是，无论如何设计，程序检查错误都是至关重要的。这里的讨论不是关于如何避免检查错误，而是关于如何优雅的处理错误。

当我参加2014年秋季东京的GoCon时，出现了重复性错误检查代码的主题。一个热情的gopher，Twitter账号@jxck_，回复了我一段关于错误检查的代码片段，这段代码很熟悉。代码看起来像这样：

```
_, err = fd.Write(p0[a:b])
if err != nil {
    return err
}
_, err = fd.Write(p1[c:d])
if err != nil {
    return err
}
_, err = fd.Write(p2[e:f])
if err != nil {
    return err
}
// and so on

```

这写代码看起来极其重复的。在更长的实际代码中，还有更多内容，因此使用辅助函数重构它并不容易，但在这种理想化的形式中，关闭错误变量的函数将会有所帮助：

```
var err error
write := func(buf []byte) {
    if err != nil {
        return
    }
    _, err = w.Write(buf)
}
write(p0[a:b])
write(p1[c:d])
write(p2[e:f])
// and so on
if err != nil {
    return err
}

```

这种模式很不错，但是在每个write里面都需要一个闭包；一个单独的helper函数使用起来很笨拙，因为需要跨函数调用维护err变量。

我们可以从Scan方法中获取灵感，使得这个函数更清晰，更通用，更可重用。我在讨论中提到了这个技术，但是@jxck_并不知道如何实现。一段时间的讨论之后，我们在某个点上被卡住了，然后我就说，让他共享屏幕，我来把这段代码展示给他看。

我定义了一个叫做errWriter的对象，如下：

```
type errWriter struct {
    w   io.Writer
    err error
}
```

并且给他定一个了方法，write。并不需要符合标准的Write函数签名，因此用小写来作区别。write方法调用Write函数实现底层的Writer，并记录将来调用可能产生的错误值。

```
func (ew *errWriter) write(buf []byte) {
    if ew.err != nil {
        return
    }
    _, ew.err = ew.w.Write(buf)
}
```

只要错误一出现，write的返回值就变成了空，但是错误却被记录了下来。就errWriter类型和write方法，以上代码可以重构为：

```
ew := &errWriter{w: fd}
ew.write(p0[a:b])
ew.write(p1[c:d])
ew.write(p2[e:f])
// and so on
if ew.err != nil {
    return ew.err
}
```

与使用闭包相比，这段代码更清晰，并且还使得页面上更容易看到实际的写入顺序。没有杂乱了。使用错误值（和接口）进行编程使代码更好。

很可能同一个包中的其他代码可以构建在同样的想法上，甚至可以直接使用errWriter。

此外，一旦errWriter存在，它还可以做更多事情，尤其是在人工干预较少的例子中。它可以累积字节数。它可以将写入合并到一个缓冲区中，然后可以原子方式传输。甚至更多。

实际上，这种模式通常出现在标准库中。 archive/zip和net/http包使用它。这个讨论更加突出，bufio包的Writer实际上是errWriter想法的实现。虽然bufio.Writer.Write返回了错误，但这主要是出于对io.Writer接口的尊重。 bufio.Writer的Write方法就像我们上面的errWriter.write方法一样，Flush报告错误，因此我们的示例可以像这样编写：


```
b := bufio.NewWriter(fd)
b.Write(p0[a:b])
b.Write(p1[c:d])
b.Write(p2[e:f])
// and so on
if b.Flush() != nil {
    return b.Flush()
}
```

这种方法有一个明显的缺点，至少对于某些应用程序来说是这样：在错误发生之前无法知道完成了多少处理。如果该信息很重要，则需要采用更细粒度的方法。 但是，通常来说，最后的错误检查就足够了。

我们只研究了一种避免重复错误处理代码的技术。请记住，使用errWriter或bufio.Writer并不是简化错误处理的唯一方法，并且这种方法并不适合所有情况。 然而，关键的一课是，错误是值，Go语言的全部功能都可以用于处理它们。

使用以上语言方法可以简化错误处理。

但请记住：无论你做什么，总是检查你的错误！

最后，关于我与@jxck_互动的完整故事，包括他录制的一个小视频，请访问他的博客。

By Rob Pike


原文：<https://blog.golang.org/errors-are-values>