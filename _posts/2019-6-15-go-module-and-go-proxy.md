---
layout:     post
title:      "go module and go proxy"
subtitle:   " \"使用go module爽，一直用一直爽\""
date:       2019-06-15 16:00:00
author:     "会飞的蜗牛"
header-img: "img/go-module-go-proxy.jpg"
tags:
    - Go Module
    - Go Proxy
    - Package
    
---

# Go Module
## 创建一个module
我们先来创建一个自己的包。叫做helloworld。但是需要注意的是，不要在GOPATH路径下创建文件夹，默认来说在GOPATH路径下，module功能是关闭的。我们go module的目标之一就是消除掉GOPATH。如果你想在GOPATH下新建项目或者修改已有项目，需要打开module功能，export GO111MODULE=on。

```
$ mkdir helloworld
$ cd helloworld
```

```
package helloworld

import "fmt"

//Hi say hi to someone
func Hi(name string) string {
    return fmt.Sprintf("hello world, %s!\n", name)
}
```

包是写完了，但是他还不是一个module，我们需要给他mod初始化。

```
$ go mod init github.com/torubylist/helloworld
go: creating new go.mod: module github.com/torubylist/helloworld
```

在当前包的路径下会多出来一个go.mod文件。

```
module github.com/torubylist/helloworld
```

然后，我们可以把这个包上传到github上。

```
$ git init 
$ git add * 
$ git commit -am "First commit" 
$ git push -u origin master
```

然后大家都可以引入这个包了。

```
$ go get github.com/torubylist/helloworld
```

这样就可以获取最新的master分支的代码了。获取master分支的代码还是比较危险的，因为最新的更新都在这个包里，所以我们很难知道作者是否做了什么不好的更新之类的。而module的目标之一就是要修复这个问题。

## 引入包的某个版本
Go的模块有不同的版本，每个版本都有自己特殊的功能。所以Go语言在查找包的时候会使用版本库标记-tag。不同的版本应该有不同的导入路径，默认情况下，Go将获取存储库里面可用的最新的版本。这点尤其需要注意，因为我们平时可能习惯使用master分支，而这里默认引入的可能是最新的标记版本。所以当我们需要发布一个包的时候，记得给我们的包打上tag，在推送到远端存储。

如果我们的包已经准备好，那么我们就可以给包打上tag，然后推送到存储库里。

```
$ git tag v1.0.0
$ git push --tags
```

这样我们的github库里就多了一个release分支v1.0.0。当我们有bug需要fix时候，记得先新建分支，再修bug，再push到远端存储库。

```
$ git checkout -b v1
$ git push -u origin v1
```

这样我们就可以在master上改代码，而无需担心会打乱我们的release。

## 版本管理


首先，引入我们的包。

```
package main
import (
    "fmt"

    "github.com/torubylist/helloworld"
)

func main() {
    fmt.Println(helloworld.Hi("roberto"))
}
```

到目前为止，还是会通过`go get github.com/torubylist/helloworld`去下载我们的包，但是有module，就没必要这么做了。

```
$ go mod init mod
```

这里就是创建一个mod，跟前文一样，然后我们使用`go build`构建我们的程序。go module会自动去下载我们的引入的包。

```
$ go build
go: finding github.com/torubylist/helloworld v1.0.0
go: downloading github.com/torubylist/helloworld v1.0.0
```

```
cat go.mod
module mod
require github.com/torubylist/helloworld v1.0.0
```

同时，多了一个新的文件，go.sum。包括包的哈希值。这样确保我们使用的是正确的版本和文件。

```
github.com/torubylist/helloworld v1.0.0 h1:9EdH0EArQ/rkpss9Tj8gUnwx3w5p0jkzJrd5tRAhxnA=
github.com/torubylist/helloworld v1.0.0/go.mod h1:UVhi5McON9ZLc5kl5iN2bTXlL6ylcxE9VInV71RrlO8=
```

如果我们在开发的时候，发现了一个bug，那么显然我们需要创建一个bug-fix release。那么我们可以在v1分支上做这件事情，因为这跟v2分支无关。当然现实当中，我们也可以在master上作出修改，然后将它移至到v1分之上。

```
// Hi returns a friendly greeting
func Hi(name string) string {
-       return fmt.Sprintf("Hi, %s", name)
+       return fmt.Sprintf("Hi, %s!", name)
}
```


```
$ git commit -m "Emphasize our friendliness" testmod.go
$ git tag v1.0.1
$ git push --tags origin v1
```

这里既然引用的包做出了修改，那么如果我们引入方也需要更新我们的引用。

```
$ go get -u #使用最新的小版本或者patch分支，例如v1.0.0更新到v1.0.1或者v1.1.0
$ go get -u=patch #使用最新的patch分支，v1.0.1，但是不能是v1.1.0
$ go get github.com/torubylist/helloworld@v1.0.1  #更新到指定版本
```

作为包的开发者，如果我们对已有的package做了较大的修改，那么我们一般会更新我们的主版本，同时修改go.mod文件，将新的包和版本都更新进去，例如`module github/torubylist/helloworld/v2`, 然后再推送到远端。

```
$ git commit helloworld.go -m "Change Hi to allow multilang"
$ git checkout -b v2 # optional but recommended
$ echo "module github.com/torubylist/helloworld/v2" > go.mod
$ git commit go.mod -m "Bump version to v2"
$ git tag v2.0.0
$ git push --tags origin v2 # or master if we don't have a branch
```

而作为包的使用者。这时候就需要引入新的module，这时候，包的路径就是包/版本。例如

```
package main
import (
    "fmt"
    "github.com/torubylist/helloworld"
    testmodML "github.com/torubylist/helloworld/v2"
)
```

垃圾回收。go module并不会主动进行垃圾回收，例如有的包不需要了。go module自己并不会自动回收不需要的包。而是需要手动回收，运行命令。

```
 go mod tidy
```
vendor文件夹，go module的目标之一就是放弃vendor，但是如果你一定想要vendor，那么还是可以做到的。`go mod vendor`,这个命令会在项目的根目录下面创建vendor文件夹，并包含我们需要的依赖包。但是`go build`的时候还是会跳过vendor，所以要依赖vendor，就需要运行`go build -mod vendor`。

目前来看，大家还是会在开发项目的时候带上vendor，因为从CI的角度来说更为方便。因为直接从上游服务器拉取包对很多人来说并不是很方便，所以go module推出了go proxy服务，后文再讲。

## Go module常用命令

```
download    download modules to local cache (下载依赖的module到本地cache))
edit        edit go.mod from tools or scripts (编辑go.mod文件)
graph       print module requirement graph (打印模块依赖图))
init        initialize new module in current directory (再当前文件夹下初始化一个新的module, 创建go.mod文件))
tidy        add missing and remove unused modules (增加丢失的module，去掉未用的module)
vendor      make vendored copy of dependencies (将依赖复制到vendor下)
verify      verify dependencies have expected content (校验依赖)
why         explain why packages or modules are needed (解释为什么需要依赖)
```


# GO Proxy

go proxy的意思就是当你需要依赖一个包，并且本地`$GOPATH/pkg/mod/`下面没有，然后就从网络中去下载（github，golang等），但是如果我们想准确的控制要下载的版本，那么就需要设置一个proxy server，可以从这个proxy里面去下载。proxy server本质上来说是一个http服务。

流程入下图所示：
![](https://roberto.selbach.ca/wp-content/uploads/2018/08/goproxyseq.png)


假设我们依赖的包是`github/torubylist/helloworld/v2 v0.8.0`，记住，`github/torubylist/helloworld/v2`和`github/torubylist/helloworld`是两个不同的包。

1. 获取一个包的版本列表。获取github/torubylist/helloworld/v2/@v/list。这里有这个包的所有版本。
2. module的元数据，例如版本v0.8.0，名字helloworld，短id xxx，以及commit时间等。
3. 然后是获取某个版本的 go.mod 文件，里面有mod信息，`mod github/torubylist/helloworld/v2`
4. 获取module.zip本身，例如github/torubylist/helloworld/v2/@v/0.8.0.zip。这里不会包含该包的测试文件。

我们也可以自己是实现一个简单的proxy server。甚至可以将这些文件放到某个目录下:

```
$ find . -type f
./github.com/torubylist/helloworld/@v/v1.0.0.mod
./github.com/torubylist/helloworld/@v/v1.0.1.mod
./github.com/torubylist/helloworld/@v/v1.0.1.zip
./github.com/torubylist/helloworld/@v/v1.0.0.zip
./github.com/torubylist/helloworld/@v/v1.0.0.info
./github.com/torubylist/helloworld/@v/v1.0.1.info
./github.com/torubylist/helloworld/@v/list

$ pwd
/home/zyp/go-proxy

$ export GOPROXY=/home/zyp/go-proxy

```

现在网上已经有一些比较有名的proxy server，我们没必要自己再造一个。如https://goproxy.io，国内网络是通的，不翻墙也不用担心找不到`golang/xx/xx`包了，设置好之后就不需要vendor了。编译的时候发现依赖在本地缓存中(`$GOPATH/pkg/mod/`)找不到，就直接从这个proxy去找。

```
export GO111MODULE=on
export GOPROXY=https://goproxy.io
```

## 参考
<https://roberto.selbach.ca/intro-to-go-modules/>

<https://roberto.selbach.ca/go-proxies/>

<https://colobu.com/2018/08/27/learn-go-module/>

<https://goproxy.io/>