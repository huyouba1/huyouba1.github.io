---
layout: post
title: Goroutine
date: 2023-12-16
tags: Go
---

## 并发介绍

并发编程是将一个过程按照并行算法拆分为多个可以独立执行的代码块，从而充分利用多核和多处理器提高系统吞吐率。


**进程和线程**

> 进程是程序在操作系统中的一次执行过程，系统进行资源分配和调度的一个独立单位。
> 线程是进程的一个执行实体，是CPU调度和分派的基本单位，它是比进程更小的能独立运行的基本单位。
> 一个进程可以创建和撤销多个线程；同一个进程中的多个线程之间可以并发执行。



**并发和并行**

> 多线程程序在一个核的CPU上运行就是并发。
> 多线程程序在多个核的CPU上运行就是并行。




**并发**

![](/images/posts/media/bingfa01.png)



**并行**

![](/images/posts/media/bingfa02.png)


**协程和线程**

> 协程: 独立的栈空间，共享堆空间，调度由用户自己控制，本质上有点类似于用户级线程，这些用户级线程的调度也是自己实现的。
> 线程: 一个线程上可以跑多个协程，协程是轻量级的线程。



**Goroutine** 

Go 语言中每个并发执行的单元叫 Goroutine，使用 go 关键字后接函数调用来创建一个 Goroutine

goroutine 奉行通过通信来共享内存，而不是共享内存来通信。

### Goroutine
在java/c++中我们要实现并发编程的时候，我们通常需要自己维护一个线程池，并且需要自己去包装一个又一个的任务，同时需要自己去调度线程执行任务并维护上下文切换，这一切通常会耗费程序员大量的心智。那么能不能有一种机制，程序员只需要定义很多个任务，让系统去帮助我们把这些任务分配到CPU上实现并发执行呢？

Go语言中的goroutine就是这样一种机制，goroutine的概念类似于线程，但 goroutine是由Go的运行时（runtime）调度和管理的。Go程序会智能地将 goroutine 中的任务合理地分配给每个CPU。Go语言之所以被称为现代化的编程语言，就是因为它在语言层面已经内置了调度和上下文切换的机制。

在Go语言编程中你不需要去自己写进程、线程、协程，你的技能包里只有一个技能–goroutine，当你需要让某个任务并发执行的时候，你只需要把这个任务包装成一个函数，开启一个goroutine去执行这个函数就可以了，就是这么简单粗暴。

**使用goroutine**

Go语言中使用goroutine非常简单，只需要在调用函数的时候在前面加上`go`关键字，就可以为一个函数创建一个goroutine。

一个goroutine必定对应一个函数，可以创建多个goroutine去执行相同的函数。


```go
package main

import (
	"fmt"
	"runtime"
)

func printStr(prefix string) {
	for ch := 'A'; ch <= 'Z'; ch++ {
		fmt.Printf("%s:%c\n", prefix, ch)
	}
	runtime.Gosched()
}

func main() {
	go printStr("go1")
	go printStr("go2")
	printStr("go3")
}
```