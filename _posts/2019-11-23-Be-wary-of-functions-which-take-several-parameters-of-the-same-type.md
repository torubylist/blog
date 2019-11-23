---
layout:     post
title:      "Be wary of functions which take several parameters of the same type"
subtitle:   " \"golang knowledge\""
date:       2019-11-23 20:00:00
author:     "会飞的蜗牛"
header-img: "img/http-restapi.jpg"
tags:
    - Golang
    - Guides

---
> APIs should be easy to use and hard to misuse.

> — Josh Bloch

原文：[Be wary of functions which take several parameters of the same type](https://dave.cheney.net/2019/09/24/be-wary-of-functions-which-take-several-parameters-of-the-same-type)

## 案例分析
先举个看起来简单，但用起来容易出错的栗子，一个API，拥有两个或者多个相同类型的参数。如下所示：

```
func Max(a, b int) int
func CopyFile(to, from string) error
```

这两个函数的不同之处在哪？答案很明显。一个是返回两者的最大值，另一个是复制一个文件。但这都不是重点。

```
Max(8, 10) // 10
Max(10, 8) // 10
```

最大值的参数位置是可以互换的，对返回值不会有任何影响。不论是8跟10比，还是10跟8比，最大值都是返回10.

但是，这个特性，在CopyFile函数中就不存在了。

```
CopyFile("/tmp/backup", "presentation.md")
CopyFile("presentation.md", "/tmp/backup")
```

上面两个调用看起来就很让人困惑，到底哪个文件复制到哪个文件。为了搞懂他，你不得不查阅文档，甚至翻阅源代码。代码复查者也不能确定这样的代码会不会有问题，除非他去查阅文档。

通常的建议是去避免这种情况的发生。就像又臭又长的参数列表一样，无区分的的参数同样散发着烂代码的味道。

## 一种解决方案
如果这种情况无法避免的话，我的答案通常是引入一个帮助类型，它的职责是使得调用CopyFile不容易出错。

```
type Source string

func (src Source) CopyTo(dest string) error {
	return CopyFile(dest, string(src))
}

func main() {
	var from Source = "presentation.md"
	from.CopyTo("/tmp/backup")
}
```

通过这种方法，CopyFile总是能够轻易的正确调用，并且这些简陋的api可以设置为私有，这样可以进一步减少误用的可能性。

你还有更好的方法吗？
