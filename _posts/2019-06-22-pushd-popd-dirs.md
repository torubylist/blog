---
layout:     post
title:      "如何在Linux的多个目录中自由切换"
subtitle:   " \"pushd popd dir\""
date:       2019-06-22 12:00:00
author:     "会飞的蜗牛"
header-img: "img/pushd-popd-dirs.jpg"
tags:
    - pushd
    - popd
    - dir
    
---

## cd
我们刚使用linux的时候，最常用的命令之一就是`cd`， 该命令的是`change directory`的简称。顾名思义，就是用来切换文件夹。

`cd -`可以用来退回到上一个工作目录，这个命令非常常用，而且也非常实用。我们的工作当中几乎每天都会用到。

`cd -`本质上是引用环境变量`$OLDPWD`。我们可以来看看二者的区别。

```
root@dev:~# cd -
/vagrant_data
root@dev:/vagrant_data# cd -
/root
root@dev:~# cd $OLDPWD
root@dev:/vagrant_data# cd $OLDPWD
root@dev:~#
```
二者仅有细微的区别，这两个命令仅仅适合在两个目录上切换。如何在多个目录上切换呢？接下来要介绍的是三个大杀器，pushd, popd, dirs。

## pushd/popd/dirs
切换到作为参数的目录，并把原目录和当前目录压入到一个虚拟的环形链表中
如果不指定参数，则会回到前一个目录，并把堆栈中最近的两个目录作交换

1. usage

	```
pushd: usage: pushd [-n] [+N | -N | dir]
popd: usage: popd [-n] [+N | -N]
dirs: usage: dirs [-clpv] [+N] [-N]
```
2. `pushd`和`popd`不带参数,`pushd`不带参数则在链表头部的两个目录中来回切换。`popd`则表示删除链表表头节点。

	```
root@dev:/vagrant_data# dirs -v
 0  /vagrant_data
 1  /home/ubuntu
root@dev:/vagrant_data# pushd	
/home/ubuntu /vagrant_data
root@dev:/home/ubuntu# dirs -v
 0  /home/ubuntu
 1  /vagrant_data
root@dev:/home/ubuntu# pushd
/vagrant_data /home/ubuntu
root@dev:/vagrant_data# dirs -v
 0  /vagrant_data
 1  /home/ubuntu
root@dev:/vagrant_data# popd
/home/ubuntu
root@dev:/home/ubuntu# dirs -v
 0  /home/ubuntu

	```

3. `pushd /dest/dir` 讲该目录与原目录一起压入到虚拟堆栈中。并将当前目录置于栈顶。 `popd ` 将顶目录从目录堆栈中删除。工作目录依然在剩下的栈顶中。`dirs`展示当前堆栈中的目录。

	```
root@dev:~# dirs -v
 0  ~
root@dev:~# pushd /vagrant_data/
/vagrant_data ~
root@dev:/vagrant_data# dirs -v
 0  /vagrant_data
 1  ~
 root@dev:/vagrant_data# pushd /home 
/home /vagrant_data ~
root@dev:/home# dirs -v
 0  /home
 1  /vagrant_data
 2  ~
 root@dev:/home# popd +2 #删除2号目录
 0  /home
 1  /vagrant_data
 2  ~
 root@dev:/home# dirs -v
 0  /home
 1  /vagrant_data
root@dev:/home# pushd ~
~ /home /vagrant_data
root@dev:~#
root@dev:~# pushd ~ #可以多次将同一目录压入堆栈
~ ~ /home /vagrant_data
root@dev:~# pushd ~
~ ~ ~ /home /vagrant_data
root@dev:~# dirs -v
 0  ~
 1  ~
 2  ~
 3  /home
 4  /vagrant_data
 
	```

4. `pushd [+N | -N]` 旋转目录链表。$PWD 始终在链表头部。
+N: 则表示从头部开始第N号目录及后面的目录往上移。整个操作像一个环形链表一样。
-N：则表示从尾部开始的第N号目录及后面的目录往上移。

	```
root@dev:/vagrant_data# dirs -v
 0  /vagrant_data
 1  ~
 2  /home
 3  /home/ubuntu
root@dev:/vagrant_data# pushd +2
/home /home/ubuntu /vagrant_data ~
root@dev:/home# dirs -v
 0  /home
 1  /home/ubuntu
 2  /vagrant_data
 3  ~
root@dev:/home# pushd -1
/vagrant_data ~ /home /home/ubuntu
root@dev:/vagrant_data# dirs -v
 0  /vagrant_data
 1  ~
 2  /home
 3  /home/ubuntu
	```

5. `popd [+N | -N]`删除目标目录。$PWD 始终在链表头部。
+N: 则表示删除从头部开始序号为N的目录。
-N：则表示删除从尾部开始的序号为N目录。

	```
root@dev:/vagrant_data# dirs -v
 0  /vagrant_data
 1  ~
 2  /home
 3  /home/ubuntu
root@dev:/vagrant_data# popd +1 #删除从头部开始的序号为1的目录
/vagrant_data /home /home/ubuntu
root@dev:/vagrant_data# dirs -v
 0  /vagrant_data
 1  /home
 2  /home/ubuntu
root@dev:/vagrant_data# popd -1 #删除从尾部开始序号为1的目录
/vagrant_data /home/ubuntu
	```


## 参考
<https://blog.csdn.net/muzilanlan/article/details/45564163>
<https://unix.stackexchange.com/questions/77077/how-do-i-use-pushd-and-popd-commands>