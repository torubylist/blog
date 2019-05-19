---
layout:     post
title:      "[译]Go语言错误处理"
subtitle:   " \"Error handling and go\""
date:       2019-05-19 18:00:00
author:     "会飞的蜗牛"
header-img: "img/errors_handling_and_go.jpg"
tags:
    - Errors
    - Go
---

## 简介
如果你已经写过Go代码，你很可能已经遇到了内建错误类型。Go代码使用错误值来表示异常状态。例如，os.Open函数当出现错误时会返回一个non-nil错误值。

`func Open(name string) (file *File, err error)`

下面的代码使用os.Open打开一个文件。如果出现错误，它就调用log.Fatal来打印错误信息并停止程序。

```
f, err := os.Open("filename.ext")
if err != nil {
    log.Fatal(err)
}
// do something with the open *File f
```

了解Go语言的error类型有助于你做很多事情。但是这篇文章我们将近距离学习error类型，并讨论一下错误处理的最佳实践。

## error类型
error类型是接口类型。error变量表示任何可以将自身描述为字符串的值。下文是一个接口申明：

```
type error interface {
    Error() string
}
```

与所有内置类型一样，error类型在全局块中预先声明。

最常用的error实现是errors包中未导出的errString类型。

```
// errorString是一个简单的error实现。
type errorString struct {
    s string
}

func (e *errorString) Error() string {
    return e.s
}
```

你可以用errors.New函数构造任意值。它将字符串转换为errors.errorString实例，并返回一个错误值。

```
// New 返回格式为给定文本的错误。
func New(text string) error {
    return &errorString{text}
}
```

你可以这样使用errors.New:

```
func Sqrt(f float64) (float64, error) {
    if f < 0 {
        return 0, errors.New("math: square root of negative number")
    }
    // implementation
}
```

将负数传递给Sqrt，调用者会收到一个非零错误值（其具体表示形式为errors.errorString值）。调用者可以通过调用错误的Error方法或只打印错误字符串（“math：square root of ...”）：

```
f, err := Sqrt(-1)
if err != nil {
    fmt.Println(err)
}
```

fmt包通过调用`Error() string`格式化了一个error值。

总结上下文内容是error实现的任务。os.Open返回的错误格式为“open /etc/passwd: permission denied”，而不仅仅是“permission denied”。 我们的Sqrt返回的错误缺少无效参数的信息。

要添加该信息，一个有效的方法是使用fmt包的Errorf函数。它根据Printf的规则格式化一个字符串，并将其作为error.New创建的错误返回。

```
if f < 0 {
    return 0, fmt.Errorf("math: square root of negative number %g", f)
}
```

通常，在许多情况下，fmt.Errorf是足够好的。但是既然error是一个接口，你就可以使用任意数据结构作为错误值，以允许调用者检查错误的详细信息。

例如，假如我们的调用者可能想要获取传递给Sqrt的无效参数。我们可以通过定义新的错误实现而不是使用errors.errorString来实现：

```
type NegativeSqrtError float64

func (f NegativeSqrtError) Error() string {
    return fmt.Sprintf("math: square root of negative number %g", float64(f))
}
```

复杂的调用者可以使用类型[断言](https://golang.org/doc/go_spec.html#Type_assertions)来检查NegativeSqrtError并专门处理它，而将错误传递给fmt.Println或log.Fatal的调用者将看不到行为的变化。

作为另一个示例，[json](https://golang.org/pkg/encoding/json/)包指定了一个SyntaxError类型，当遇到解析JSON blob的语法错误时，json.Decode函数返回该类型。

```
type SyntaxError struct {
    msg    string // description of error
    Offset int64  // error occurred after reading Offset bytes
}

func (e *SyntaxError) Error() string { return e.msg }
```

偏移字段甚至未显示在错误的默认格式中，但调用者可以使用它将文件和行信息添加到其错误消息中：

```
if err := dec.Decode(&val); err != nil {
    if serr, ok := err.(*json.SyntaxError); ok {
        line, col := findLine(f, serr.Offset)
        return fmt.Errorf("%s:%d:%d: %v", f.Name(), line, col, err)
    }
    return err
}
```

(这是[Camlistore](http://camlistore.org/)项目[实战代码](https://github.com/camlistore/go4/blob/03efcb870d84809319ea509714dd6d19a1498483/jsonconfig/eval.go#L123-L135)的简化版)

实现error接口仅仅需要添加一个Error方法; 特定错误实现可能需要其他方法。 例如，[net](https://golang.org/pkg/net/)按照通常的约定返回error类型的错误，但是一些错误实现具有net.Error接口定义的其他方法：

```
package net

type Error interface {
    error
    Timeout() bool   // Is the error a timeout?
    Temporary() bool // Is the error temporary?
}
```

客户端代码可以使用类型断言测试net.Error，然后将瞬态网络错误与永久网络错误区分开来。 例如，网络爬虫在遇到临时错误时可能会睡眠并重试，否则就会放弃。

```
if nerr, ok := err.(net.Error); ok && nerr.Temporary() {
    time.Sleep(1e9)
    continue
}
if err != nil {
    log.Fatal(err)
}
```

简化重复的错误处理，在Go语言中，错误处理很重要。Go语言的设计和约定鼓励您明确地检查它们发生的错误（与其他语言中抛出异常并有时捕获它们的约定不同）。但是在某些情况下，这会使Go代码变得冗长，但幸运的是，您可以使用一些技术来最小化重复的错误处理。

让我们来看一下HTTP处理的[App Engine](https://cloud.google.com/appengine/docs/go/)应用程序，该处理程序从数据存储区检索记录并使用viewTemplate.Execute对其进行格式化。

```
func init() {
    http.HandleFunc("/view", viewRecord)
}

func viewRecord(w http.ResponseWriter, r *http.Request) {
    c := appengine.NewContext(r)
    key := datastore.NewKey(c, "Record", r.FormValue("id"), 0, nil)
    record := new(Record)
    if err := datastore.Get(c, key, record); err != nil {
        http.Error(w, err.Error(), 500)
        return
    }
    if err := viewTemplate.Execute(w, record); err != nil {
        http.Error(w, err.Error(), 500)
    }
}
```

该函数处理datastore.Get和viewTemplate.Execute方法返回的错误。在这两种情况下，它都会向用户显示一条简单的错误消息，其中包含HTTP状态代码500（“Internal Server Error”）。这看起来像是一个可管理的代码量，但是当添加更多的HTTP处理程序时，你很快就会得到许多相同的错误处理代码的副本。

为了减少重复代码，我们定义一个自己的HTTP appHandler类型，包括一个错误返回值：

`type appHandler func(http.ResponseWriter, *http.Request) error`

然后我们改变我们的viewRecorde函数：

```
func viewRecord(w http.ResponseWriter, r *http.Request) error {
    c := appengine.NewContext(r)
    key := datastore.NewKey(c, "Record", r.FormValue("id"), 0, nil)
    record := new(Record)
    if err := datastore.Get(c, key, record); err != nil {
        return err
    }
    return viewTemplate.Execute(w, record)
}
```

这次的代码比原始版本更简洁，但是[http](https://golang.org/pkg/net/http/)包并不能理解返回值为error的函数。为了修复这个问题，我们实现了http.Handler接口的ServerHTTP方法。

```
func (fn appHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    if err := fn(w, r); err != nil {
        http.Error(w, err.Error(), 500)
    }
}
```

ServeHTTP方法调用appHandler函数并向用户显示返回的错误（如果有）。请注意，该方法的接收器fn是一个函数（Go可以这样做！）。该方法通过在表达式fn(w，r)中调用接收器来调用该函数。

现在，当使用http包注册viewRecord时，我们使用Handle函数（而不是HandleFunc），因为appHandler是一个http.Handler（不是http.HandlerFunc）。

```
func init() {
    http.Handle("/view", appHandler(viewRecord))
}
```

有了这个基本的错误处理方法，我们可以使它对用户更加友好。 不仅仅显示错误字符串，最好为用户提供带有适当HTTP状态代码的简单错误消息，同时将完整错误记录到App Engine控制台以便开发人员进行调试。

为此，我们创建一个包含错误和其他一些字段的appError结构：

```
type appError struct {
    Error   error
    Message string
    Code    int
}
```

接下来我们修改appHandler类型来返回*appError值：

`type appHandler func(http.ResponseWriter, *http.Request) *appError`

（传递错误的具体类型而不是error接口通常是错误的，Go [FAQ](https://golang.org/doc/go_faq.html#nil_error)中已经讨论了的具体的原因，但在这里是正确的做法，因为ServeHTTP是唯一看到错误值并使用其内容的地方。）

并使appHandler的ServeHTTP方法展示使用正确的HTTP状态代码的appError的消息，并将完整的错误记录到开发人员控制台：

```
func (fn appHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    if e := fn(w, r); e != nil { // e is *appError, not os.Error.
        c := appengine.NewContext(r)
        c.Errorf("%v", e.Error)
        http.Error(w, e.Message, e.Code)
    }
}
```
最后，我们将viewRecord更新为新的函数签名，并在遇到错误时返回更多的内容：

```
func viewRecord(w http.ResponseWriter, r *http.Request) *appError {
    c := appengine.NewContext(r)
    key := datastore.NewKey(c, "Record", r.FormValue("id"), 0, nil)
    record := new(Record)
    if err := datastore.Get(c, key, record); err != nil {
        return &appError{err, "Record not found", 404}
    }
    if err := viewTemplate.Execute(w, record); err != nil {
        return &appError{err, "Can't display record", 500}
    }
    return nil
}
```

此版本的viewRecord与原始版本的长度相同，但现在的每一行都具有特定含义，我们提供了更友好的用户体验。

并没有到结束的时候;我们还可以进一步改进我们应用程序中的错误处理。一些想法：

给错误处理程序一个漂亮的HTML模板，

通过在用户是管理员时将堆栈跟踪写入HTTP响应来简化调试，

为appError编写一个构造函数，用于存储堆栈跟踪以便于调试，

从appHandler中的panic中恢复，将错误记录到控制台为“Critical”，同时告诉用户“发生了严重错误”。这是一个很好的尝试，以避免使用户暴露给由于编程错误导致的不可思议的错误消息。相关详细信息，请参阅[Defer，Panic，Recover](https://golang.org/doc/articles/defer_panic_recover.html)。

## 结论
正确的错误处理是良好软件的基本要求。通过使用本文中描述的技术，您应该能够编写更可靠和简洁的Go代码。

## 原文
<https://blog.golang.org/error-handling-and-go>