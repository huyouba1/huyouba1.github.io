---
layout: post
title: Client-go Informer
date: 2023-12-04
tags: Go
---

### informer 简介

Informer负责与Kubernetes APiserver进行Watch 操作，Watch的资源，可以是Kubernetes内置资源对象，也可以是CRD。

Informer是一个带有本地缓存以及索引机制的核心工具包，当请求为查询操作的时候，会有限从本地缓存内查找数据，而创建、更新、删除这类操作，则会根据事件通知写入到队列DeltaFIFO中，同时对应的事件处理过后，更新本地缓存，使本地缓存与ETCD的数据保持一致。

Informer抽象出来的这个缓存层，将查询相关操作的压力接收了下来，这样就不必每次都去调用APiserver的接口，减轻了APiserver的数据交互压力。

从下图可以看出来，Informer是有多个组件组成的。

- Reflector：使用List-Watch来保证本地缓存数据的，准确性、顺序性和一致性，List对应资源的全量数据列表，Watch负责变化部分的数据，Watch指定的Kubernetes资源，当watch的资源发生变化时，触发变更的事件，比如Added，Updated和Deleted事件，并将资源对象的变化事件存放到本地队列DeltaFIFO中。
- DeltalFIFO：是一个增量队列，记录了资源变化的过程，Reflector就相当于队列的生产者。这个组件可以拆分为两部分来理解，FIFO就是一个队列，拥有队列基本方法，例如ADD、UPDATE、DELETE、LIST、POP、CLOSE等。Delta是一个资源对象存储，保存存储对象的消费类型，比如Added，Updated，Deleted等。
- Indexer：用来存储资源对象并自带索引功能的本地存储，Reflector从DeltaFIFO中将消费出来的资源对象存储到Indexer，Indexer与ETCD中的数据完全保持一致。从而client-go可以本地读取，减少Kubernetes Apiserver的数据交互压力。

![](/images/posts/media/informer01.png)


### List-Watch
List-Watch机制是Kubernetes中的异步消息通知机制，通过它能有效的确保了消息的实时性、顺序性和可靠性。

List-Watch，顾名思义，他是分为两部分的：

- List负责调用资源对应的Kubernetes APiserver的RESTful API获取全局数据列表的，并同步到本地缓存中。
- Watch负责监听资源的变化，并调用相应事件的处理函数进行处理，同时更新本地缓存，使本地缓存与ETCD中数据保持一致。

还有就是List是基于HTTP中的短链接实现。Watch则是基于HTTP场长链接实现，Watch使用长链接的方式，极大的减轻了Kubernetes APiserver的访问压力。


**Watch示例**

```go
package main

import (
	"context"
	"fmt"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"
)

func main() {
	config, err := clientcmd.BuildConfigFromFlags("", "../config")
	if err != nil {
		panic(err)
	}

	clientSet, _ := kubernetes.NewForConfig(config)

	// 调用监听方法
	w, _ := clientSet.AppsV1().Deployments("default").Watch(context.TODO(), metav1.ListOptions{})

	fmt.Println("start.........")
	for {
		select {
		case e, _ := <-w.ResultChan():
			fmt.Println(e.Type, e.Object)
			// e.Type: 表示事件变化的类型，ADD DELETE
			// e.Object: 表示变化后的数据
		}
	}
}
```
> 此时可以对default名称空间的Deployment资源进行修改，可获取到监听数据的更新。


Watch返回的事件实例，存在了两个属性，分别是Type和Object，Type的返回值是这个事件的类型，例如ADDED、MODIFIED等，表示增加或者变更之类的操作。Object则是最新的资源信息数据。



### Reflector
Reflector是Client-Go中，用来监听指定资源的组件，当资源发生变化的时候，例如增加、更新、删除等操作的时候，会以事件的形式存入本地队列，然后有对应的方法处理。

在Reflector中，核心的部分就是List-Watch，其他功能基本上也是围绕着它来搞的。

在实例化Reflector的过程中，有一个`ListerWatch`的接口对象，这个结构对象有两个方法，分别是`List`和`Watch`，这两个方法实现了List-Watch功能。


Reflector的核心逻辑可以简单归为三个部分。

- List：调用List方法获取资源全部列表数据，转换为资源对象列表，然后保存到本地缓存中。
- 定时同步：定时器定时触发同步机制，定时更新缓存数据，在Reflector的结构体对象中，是可以配置定时同步的周期时间的。
- Watch：监听资源的变化，并且调用对应的事件处理函数来进行处理。

Reflector组件对于数据更新同步，都是基于`ResourceVersion`来进行的，每个资源对象都会有`ResourceVersion`这个属性，当数据发生变化的时候,`ResourceVersion`也将会以递增的形式更新，这样就确保了变更事件的顺序性。

Watch资源对象的时候,`ResourceVersion`可以根据我们的需求，进行配置。

- ResourceVersion未设置的时候，从最新版本开始监听。
    - 为了建立初始状态，Watch从起始资源版本中存在的所有资源实例的合成“添加”事件开始。一下所有的监视时间都针对在Watch开始的资源版本之后发生的所有更改。

- ResourceVersion设置为“0”的时候，则表示从任意版本开始监听。
    - 以这种方式初始化的监视可能会返回任意陈旧的数据。首选可用的最新资源版版本，但不是必须的。允许任何起始资源版本。由于分区或过时的缓存，Watch可能从客户端之前观察到的更旧的资源版本开始，特别是在高可用性配置中。不能容忍这种明显倒带的客户不应该用这种语义启动Watch。
    - 为了建立初始状态，Watch从起始资源版本中存在的所有资源实例的合成“添加”事件开始。以下所有监视事件都针对在watch开始的资源版本之后发生的所有更改。

- ResourceVersion从指定版本开始监听。
    -   监视事件适用于提供的资源版本之后的所有更改。Watch不会以所提供资源版本的合成“添加”事件启动。由于客户端提供了资源版本，因此假定客户端已经具有起始资源版本的初始状态

ResourceVersion不仅是在Reflector中有重要的应用，Update机制或Patch机制也是会基于ResourceVersion来比较两个资源对象，确定是否有变化的。

当Watch资源断开的时候，Reflector会重新List-Watch以确保数据的可靠性。

同时Watch使用HTTP的长链接的形式进行资源的监听，保证了数据实时性的同时，还减轻Kubernetes APiserver的访问压力。


### DeltaFIFO

DeltaFIFO 是一个增量的本地队列，记录了资源对象的变化过程。

它的生产者就是 Reflector 组件。将监听的对象，同步到 DeltaFIFO 中。

之前有过介绍 DeltaFIFO 是分为两部分的。分别 FIFO 和 Delta。

FIFO 就是一个先入先出的本地队列，Delta 则是资源对象的变化，例如增加、删除、修改等。

FIFO 负责接收 Reflector传递过来的事件，并将其按照顺序存储，然后等待事件的处理函数进行处理，同时若出现多个相同的事件，则只会被处理一次。

FIFO 既然是一个队列那么就肯定有队列相关的操作方法，在这里就是通过 Queue 这个接口对象，来实现队列所需的方法的，同时还根据需求拓展了一些其他的方法。例如: Pop、AddifNotPresent 等。

Delta 有两个属性，分别是 Type 和 Object。

- Type 就表示这个事件的类型，就比如说 Added 表示增加，Updated 表示更新等等- - Object 是一个interface 类型的，它就表示一个具体 Kubernetes 资源对象，例如: Pod、Service 等。


### Indexer
Indexer通过字面意思我们可以看出它叫做“索引器”。它其实就是 Informer 中 LocalStore 的部分。

Indexer 本身是一个存储，同时在存储的基础上扩展了索引的功能。Reflector 通过 DeltaFIFO 一系列的操作后，然后更新存储到 Indexer 中。

Indexer 中的数据、与E1cd 中的数提是完全一致的，当 CIe-Go 需要获数接的时候，无须每次都从 APISEVr 中获取，而减了 APIServer 的请求压力。

在更深入了解 Indexer 之前，我们还需要知道 Indexer 中几个非常重要的概念。

- IndexFunc:索引器函数，用于计算一个资源对象的索值列表，可以根据需求定义其他的，比如根据 Label 标签、Annotation 等属性来生成索引值列表.
- Index:存储改据，要查找某个命名空间下面的 Pod，那就要让 Pod 按照其命名空间进行索引，对应的 ndex 类型就是 map/namespace sets,.pod。
- Indexers: 存储索引器，key 为索引器名称，value 为索引器的实现函数，例如: map"namespace"]MetaNamespacelndexFunc。
- Indices: 存储缓存器，key为素引器名称，value 为缓存的数据，例如: mapamespace"mapnamespaceisets.pod.

最容易混滑的是 indexers 和 ndices 这两个概念，我们可以这样理释: dexers 是存储索引(生成索引键)的，ndices 里面是存储的真正的数据(对象键)

**ThreadSafeMap**

```
k8s.io/client-go/tools/cache/thread_safe_store.go
```
这是一个并发安全的存储，Indexer就是在ThreadSafeMap基础上进行封装的，式训练了索引相关的功能。

**Cache**
```
k8s.io/client-go/tools/cache/store.go
```
Cache是对ThreadSafeStore的一个再次封装，很多操作都是直接调用的ThreadSafeStore的操作实现的。