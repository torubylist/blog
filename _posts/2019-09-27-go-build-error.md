--
layout:     post
title:      "go build错误 stackcheck redeclared in this block"
subtitle:   " \"Go build\""
date:       2019-09-27 20:00:00
author:     "会飞的蜗牛"
header-img: "img/golang-build-0927.jpg"
tags:
    - Golang
    
---

升级到go1.13之后，一切正常。突然有一天报错。如下。
>  runtime
/usr/local/go/src/runtime/stubs_x86.go:10:6: stackcheck redeclared in this block
	previous declaration at /usr/local/go/src/runtime/stubs_amd64x.go:10:6
/usr/local/go/src/runtime/unaligned1.go:11:6: readUnaligned32 redeclared in this block
	previous declaration at /usr/local/go/src/runtime/alg.go:321:40
/usr/local/go/src/runtime/unaligned1.go:15:6: readUnaligned64 redeclared in this block
	previous declaration at /usr/local/go/src/runtime/alg.go:329:40


查阅文档， 解决方案如下，先删除老的GOROOT文件夹。再解压新的go进去。

```
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf go$VERSION.$OS-$ARCH.tar.gz
#mac
sudo tar -C /usr/local -xzf go1.13.darwin-amd64.tar.gz

```