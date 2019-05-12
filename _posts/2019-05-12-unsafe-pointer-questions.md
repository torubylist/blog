---
layout:     post
title:      "[译]关于unsafe.Pointer的几个问题"
subtitle:   "Some questions about unsafe.Pointer"
date:       2019-05-12 21:00:00
author:     "会飞的蜗牛"
header-img: "img/unsafepointer-some-questions.jpg"
tags:
    - unsafe.Pointer
---


1. 如果一个对象只被unsafe.Pointer引用，那么这个对象会被回收么？
	
	如果unsafe.Pointer指向Go对象，则垃圾收集器将知道该对象，并且不会释放属于该对象的内存。

	如果unsafe.Pointer指向的内存在Go之外分配（例如C.malloc），则上述操作不适用。

2. 如果对象的元素也包含某些指针，那么这些指针会被回收么？
	
	如果垃圾回收器发现了unsafe.Pointer，并且对象是由Go分配的，那么垃圾回收器将会遍历存储在该对象上的所有指针。
	
3. 如果以上答案都是NO，那怎么办？ unsafe.Pointer是否在内部保存类型信息？
	
	如果对象是由Go分配的：没有类型信息直接存储在unsafe.Pointer本身中。如果unsafe.Pointer指向由Go分配的内存块，则通常会有与指针关联的类型信息。如果存在类型信息，则将其存储在属于Go的内存分配器的位置。如果类型信息不存在，垃圾收集器将采用保守策略（这意味着：对象将不会被释放，垃圾收集器将找到存储在对象中的所有指针）。


4. 如果其中一个答案是YES，这意味Pointer是非法指针，对吗？
	
	如果某个特定的unsafe.Pointer在Go程序执行期间的任何时间点是有效的，并且它的值没有被改变，那么Go运行时保证它将一直保持有效直到程序结束。从一个unsafe.Pointer到另一个unsafe.Pointer的直接分配总是安全的。 某些unsafe.Pointers之间的间接分配会是编程错误，因此程序员在这种情况下需要小心。将unsafe.Pointer转换为uintptr类型的变量可能就是编程错误。

	Go运行时会忽略由C.malloc等函数分配的任何内存。如果C内存包含指向Go内存的指针，程序需要确保指针也存储在Go变量，结构或数组中。
	
>非法指针一般包括悬空指针和野指针，没有指向有效对象的指针。在C/C++等语言中，悬空指针（Dangling Pointer）指的是：一个指针的指向对象已被删除，那么就成了悬空指针。野指针是那些未初始化的指针。
>野指针通常是因为指针变量中保存的值不是一个合法的内存地址而造成的。
>合法的内存地址：
>1.在堆空间动态申请的；
>2.局部变量所在的栈。	
	
	
	## 参考
	<https://groups.google.com/forum/#!msg/golang-nuts/yNis7bQG_rY/yaJFoSx1hgIJ>
	<https://www.cnblogs.com/idorax/p/6475941.html>