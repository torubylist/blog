---
layout:     post
title:      "交叉编译Go程序"
subtitle:   " \"Go Cross Compile\""
date:       2019-06-15 20:00:00
author:     "会飞的蜗牛"
header-img: "img/go-cross-build.jpg"
tags:
    - Go
    - Cross Compile
    
---

## 环境：
- mac
- go 1.12.5

Go里面交叉编译很简单，只需设置 GOOS 和 GOARCH 两个环境变量就能生成所需平台的Go程序。

```
package main

import (
    "fmt"
    "runtime"
)

func main() {
    fmt.Printf("OS: %s\nArchitecture: %s\n", runtime.GOOS, runtime.GOARCH)
}

$GOOS=linux GOARCH=386 go build main.go
```
编译出来的就是可以在linux上跑的程序。

```
GOOS: darwin, dragonfly freebsd linux netbsd plan9 solaris windows
GOARCH: 386 amd64 arm ppc64 ppc64le 
```
如果你想查看完整的列表，可以运行：

```
go tool dist list
```

不同的操作系统下的库可能有不同的实现， 比如syscall库。看go runtime里面就会发现基于不同架构的包。通常文件名的形式表示出来。

golang里面有两种方式来表示这种区别
1. 在包里面加tag
2. 文件名后缀的方式，这是一种独占的方式。


## tag

1. 同一行的tag是或关系，表示这些操作系统都适用，任选其一都可以编译。

	```
	// +build darwin freebsd netbsd openbsd
	```

2. 不同行的tag是与关系,两个都要符合。

	```
	// +build linux darwin
	// +build 386
	```

3. ! 表示不能该操作系统或者CPU架构上运行。

	```
	// +build !linux
	```

4. tag与包头之间要空行。

	```
	// +build !linux
	package mypkg // wrong
	```

## 文件名后缀
以 `_$GOOS.go` 为后缀的文件只在此平台上编译，其它平台上编译时就当此文件不存在。完整的后缀如：

```
_$GOOS_$GOARCH.go
```

如syscall_linux_amd64.go,syscall_windows_386.go,syscall_windows.go等。


# 参考
<https://colobu.com/2015/09/28/go-cross-compiling/>
<https://stackoverflow.com/questions/20728767/all-possible-goos-value>
