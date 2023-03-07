---
layout: post
title: Envoy
date: 2023-03-05
tags: ServiceMesh
---

### 1.微服务治理面临的挑战
> kubernetes 提供了强大的应用部署、升级和弹性伸缩等管理机制，并借助于Service等实现了服务注册、发现、负载均衡；
>
> 但是,在强分布式环境中，网络的网络的不可靠性无法忽略，这显然带来了新的挑战

- 服务注册和服务发现
- 客户端重试
- 可配置的超时机制
- 负载均衡
- 限速
- 熔断
- 服务间路由
- 异常节点检测
- 健康状态检查
- 流量整型
- 流量镜像
- 边缘路由
- 按需路由
- A/B 测试
- 内部发布
- 故障注入
- 统计和度量
- 分布式追踪


### 2.服务网格

概念源于Buoyant公司的CEO Willian Morgan的文章“What's a service mesh? And do I need one?”；


是指庄主与处理服务间通信的基础设施，他负责在现代云原生应用组成的复杂拓扑中可靠的传递请求；

治理模式： 除了处理业务逻辑的相关功能之外，每个微服务还必须实现此前单体应用模型中用于网络间通信的基础功能，甚至还包括分布式应用程序之间的通信环境中应该实现的其他网络功能，例如熔断、限流、应用跟踪、指标采集、服务发现和负载均衡等。

> 实现模型经过了三代演进：内嵌于应用程序、SDK和sidecar


![](/images/posts/media/moxing.png)



### 3. 服务网格的基本功能
- 控制服务间通信：熔断、重试、超时、故障注入、负载均衡和故障转移等；
- 服务发现：通过专用的服务总线发现服务端点
- 可观测：指标数据采集、金控、分布式日志记录和分布式追踪
- 安全性：TLS、SSl通信和秘钥管理
- 身份认证和授权检查：身份认证，以及基于黑白名单或RBAC的访问控制功能
- 部署：对容器技术的原生支持，例如Docker和kubernetes等
- 服务间的通信协议：HTTP1.1、HTTP2.0和gRPC等
- 健康状态检测：检测上游服务的健康状态
- 。。。


ServiceMesh 解决方案极大降低了业务路基与网络功能之间的耦合度，能够快捷、方便地集成到现有的业务当中，并提供了多语言、多协议支持，运维和管理成本被大大压缩，且开发人员能够将精力集中于业务逻辑本身，而无需再关注业务代码以外的其他功能

一旦启用ServiceMesh，服务间的通信将遵循以下通信逻辑
- 微服务彼此间不会直接进行通信，而是由各服务前端的称为ServiceMesh的代理程序进行
- ServiceMesh内置支持服务发现、熔断、负载均衡等网络相关的用于控制服务间通信的各种高级功能
- ServiceMesh与编程语言无关，开发人员可以使用任何编程语言编写微服务的业务逻辑，各服务之间也可以使用不通的编程语言开发
- 服务间的通信的局部故障可以由ServiceMesh自动处理
- ServiceMesh中的各服务的代理程序由控制平面（Control plan）集中管理；各代理程序之间通信网络也称为数据平面（Data Plan）
- 部署于容器编排平台时，各代理程序会以微服务容器的Sidecar模式运行

### Sidecar
新一代的解决方案：让服务集中解决业务逻辑的问题，网络相关的功能则与业务逻辑剥离，并封装为独立的运行单元作为服务的反向透明代理，从而不再与业务紧密关联

换句话说，微服务的业务程序独立运行，而网络功能则以独立的代理层工作与客户端与服务之间，专门为代理的服务提供熔断、限流、追踪、指标采集和服务发现等功能；而这些由各服务的专用代理层联合组成的服务通信网络则称之为服务网格(Service Mesh)

![](/images/posts/media/sidecar.png)

**Out-of-process architecture**
![](/images/posts/media/sidecar2.png)