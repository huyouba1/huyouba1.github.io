---
layout: post
title: Client-go 介绍
date: 2023-08-28
tags: Go
---


## GVR/GVK
### 简介
在围绕Kubernetes的功能开发中，经常会调用到Kubernetes提供的API，本章就来介绍一下Kubernetes的API接口规则

Kubernetes是通过HTTP协议咦RESTFUL的形式提供的，同时支持JSON和Protobuf的数据格式，Protobuf是方便集群内部调用而支持的，我们自己平时调用Kubernetes接口，一般都是使用JSON数据格式的。

在Kubernetes API中，我们一般使用GVR或GVK来区分特定的资源，即根据不同的分组、版本以及资源，进行URL定义，有了分组和多版本的支持，即便是后续的版本中，需要去掉资源对象的某些字段或者重构API资源，也可以保证版本之间的兼容性。

同时，因为历史原因，Kubernetes的API分组是分`吴无组名资源组`和`有组名资源组`的，`无组名资源组`也被称之为核心资源组，Core Group。

**有组名资源组**
![](/images/posts/media/clientgo1.png)

**无组名资源组**
![](/images/posts/media/clientgo2.png)


### GVR/GVK 含义介绍
- G(Group)： 资源组，包含一组资源操作的集合
- V(Version)：资源版本，用于区分不同API的稳定程度及兼容性
- R(Resource): 资源信息，用于区分不同的资源API
- K(Kind)：资源对象的类型，每个资源类型都需要Kind来区分他自身代表的资源类型。

一般在接口调用的时候，我们只需要知道GVR即可。通过GVR操作对应的资源对象。

通过GVR组成RESTFUL API请求路径。例如针对apps/v1下面Deployment的RESTFUL API请求路径如下所示:

```
GET /apis/apps/v1/namespaces/{namespace}/deployments/{name}
```

GVK，相反，通过GVK信息则可以获取要读取得资源对象的GVR，进而构建RESTFUL API请求获取对应的资源。这种GVK与GVR的映射叫做RESTMapper。

RESTMapper其主要作用是在ListerWatcher时，根据schema定义的类型GVK解析出GVR，向APIserver发起HTTP请求获取资源，然后watch





## Client-Go 简介
Client-go 是负责与Kubernetes Apiserver服务进行交互的客户端库，利用Client-Go与Kubernetes进行的交互访问，以此来对Kubernetes中的各类资源对象进行管理操作，包括内置的资源对象以及CRD。

Client-go不仅被Kubernetes项目本身使用，其他围绕Kubernetes的生态，也被大量的使用，例如：kubectl，ETCD-operator等

### Client-go客户端对象
Client-go提供了4种与KubernetesApiserver交互的客户端对象，分表是RESTClient、DiscoveryClient、ClientSet、DynamicClient。

- RESTClient：最基础的客户端，主要是对HTTP请求进行封装，支持Json和Protobuf格式的数据。
- DiscoveryClient：发现客户端，负责发现APIServer支持的资源组、资源版本和资源信息
- ClientSet：负责操作Kubernetes内置的资源对象，例如Pod、Service等
- DynamicClient：动态客户端，可以对任意的Kubernetes资源对象进行通用操作，包括CRD


![](/images/posts/media/clientgo3.png)