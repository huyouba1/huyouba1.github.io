---
layout: post
title: Kubernetes 简介
date: 2020-11-25
tags: kubernetes
--- 

## 一、Kubernetes简介

### 1.1基础概念

> Kubernetes（通常写成“k8s”）Kubernetes是Google开源的容器集群管理系统。其设计目标是在主机集群之间提供一个能够自动化部署、可拓展、应用容器可运营的平台。Kubernetes通常结合docker容器工具工作，并且整合多个运行着docker容器的主机集群，Kubernetes不仅仅支持Docker，还支持Rocket，这是另一种容器技术。

**功能特性：**

•	自动化容器部署与复制

•	随时扩展或收缩容器规模

•	组织容器成组，提供容器间的负载均衡

•	快速更新及回滚容器版本

•	提供弹性伸缩，如果某个容器失效就进行替换
### 1.2 架构图

![](/images/posts/06_k8s/describe/1.png)

### 1.3 组件

#### 1.3.1 Master
> Master节点上面主要由四个模块组成：APIServer、scheduler、controller manager、etcd

•	APIServer:APIServer负责对外提供RESTful的Kubernetes API服务，它是系统管理指令的统一入口，任何对资源进行增删改查的操作都要交给APIServer处理后再提交给etcd。如架构图中所示，kubectl（Kubernetes提供的客户端工具，该工具内部就是对Kubernetes API的调用）是直接和APIServer交互的。

•	schedule:scheduler的职责很明确，就是负责调度pod到合适的Node上。如果把scheduler看成一个黑匣子，那么它的输入是pod和由多个Node组成的列表，输出是Pod和一个Node的绑定，即将这个pod部署到这个Node上。Kubernetes目前提供了调度算法，但是同样也保留了接口，用户可以根据自己的需求定义自己的调度算法。

•	controller manager:如果说APIServer做的是“前台”的工作的话，那controller manager就是负责“后台”的。每个资源一般都对应有一个控制器，而controller manager就是负责管理这些控制器的。比如我们通过APIServer创建一个pod，当这个pod创建成功后，APIServer的任务就算完成了。而后面保证Pod的状态始终和我们预期的一样的重任就由controller manager去保证了。
 
 ![](/images/posts/06_k8s/describe/2.png)
 
•	etcd:etcd是一个高可用的键值存储系统，Kubernetes使用它来存储各个资源的状态，从而实现了Restful的API。

#### 1.3.2 Node
> 每个Node节点主要由三个模块组成：kubelet、kube-proxy、runtime。
> 
> runtime指的是容器运行环境，目前Kubernetes支持docker和rkt两种容器。

•	kube-proxy:该模块实现了Kubernetes中的服务发现和反向代理功能。反向代理方面：kube-proxy支持TCP和UDP连接转发，默认基于Round Robin算法将客户端流量转发到与service对应的一组后端pod。服务发现方面，kube-proxy使用etcd的watch机制，监控集群中service和endpoint对象数据的动态变化，并且维护一个service到endpoint的映射关系，从而保证了后端pod的IP变化不会对访问者造成影响。另外kube-proxy还支持session affinity。

•	kubelet:Kubelet是Master在每个Node节点上面的agent，是Node节点上面最重要的模块，它负责维护和管理该Node上面的所有容器，但是如果容器不是通过Kubernetes创建的，它并不会管理。本质上，它负责使Pod得运行状态与期望的状态一致。

#### 1.3.3 Pod
> Pod是k8s进行资源调度的最小单位，每个Pod中运行着一个或多个密切相关的业务容器，这些业务容器共享这个Pause容器的IP和Volume，我们以这个不易死亡的Pause容器作为Pod的根容器，以它的状态表示整个容器组的状态。一个Pod一旦被创建就会放到Etcd中存储，然后由Master调度到一个Node绑定，由这个Node上的Kubelet进行实例化。
>
> 每个Pod会被分配一个单独的Pod IP，Pod IP + ContainerPort 组成了一个Endpoint。

#### 1.3.4 Service
 
 ![](/images/posts/06_k8s/describe/3.png)
 
> Service其功能使应用暴露，Pods 是有生命周期的，也有独立的 IP 地址，随着 Pods 的创建与销毁，一个必不可少的工作就是保证各个应用能够感知这种变化。这就要提到 Service 了，Service 是 YAML 或 JSON 定义的由 Pods 通过某种策略的逻辑组合。更重要的是，Pods 的独立 IP 需要通过 Service 暴露到网络中。

