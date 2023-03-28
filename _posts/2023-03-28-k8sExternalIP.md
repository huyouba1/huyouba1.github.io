---
layout: post
title: Service ExternalIP引起的问题
date: 2023-03-22
tags: 常见问题处理
---


### 问题现象
![](/images/posts/media/16799878731403.jpg)

> rancher上面所有的服务均异常，甚至连接不到apiserver


### 排查记录

主机名|IP|角色
---|---|---
cdev01|192.168.10.114|master
cdev02|192.168.10.115|node
cdev03|192.168.10.116|node


- 登录到master节点发现，节点状态为NotReady，NotReady的原因一般就是：
    1. apiserver -> kubelet 网络联通问题
    2. kubelet relist 超时问题
    3. kubelet 异常
- 尝试ping机器返回信息为`Destination Host Unreachable`，字面意思就是路由不可达，开始围绕网络相关排查

![](/images/posts/media/16799879216022.jpg)
![](/images/posts/media/16799880688972.jpg)

- 登录到115/116机器ping 114机器，发现可以联通，同时113机器除115、116节点不能访问，其他节点均能访问现象如下图

![](/images/posts/media/16799915700362.jpg)

- 尝试抓包，分析数据包，中间发送icmp
    1. 114 --> 115   无任何返回包
    2. 115 --> 114   正常
    3. 114 --> 116   无任何返回包
    4. 116 --> 114   正常

- 根据抓包现象，怀疑是路由走向问题，尝试在114机器添加两条静态路由，再次尝试，仍不通

![](/images/posts/media/16799910096770.jpg)

- 尝试删除默认路由，再次添加两个节点的静态路由，均未果
- 中间检查iptables规则、检查selinux、检查firewalld均正常，重启服务器，删除calico、calicController均未果，重试reset cni，均未果
- 中间忘了省略很多很多。。。。。大概是检查calico集群。。。。
    `calicoctl node status`
- 检查正常集群的路由表，也是三个节点，发现路由表中会有tunl1隧道网卡，而故障集群的节点中是没有的

![](/images/posts/media/16799916975486.jpg)

- 故障期间，尝试过重启calico-node pod，但是由于网络不通，已经无法接受请求。登录到对应节点直接`docker restart`相应容器，再观察115、116节点路由表，发现没有tunl0 隧道网卡，114也没有。（此问题是由于calico集群没有正常创建导致的）
- 问题很离奇，但问题基本锁定在k8s集群之中
- 多次重启114机器网卡，发现有时网卡未能启动成功，报错信息为地址被占用

![](/images/posts/media/16799921980649.jpg)

- 到这，问题基本定位了，`192.168.10.114`是master节点IP，但是由于IP地址冲突，node节点去连接`192.168.10.114`时，有可能连接的是非master节点，导致启动异常
- 停止master(114)节点网卡，通过115节点远程连接114依然可以联通，登录上去之后，发现节点为(`192.168.10.115`) 没截图
- 在115 节点 ip a 发现，在`kube-ipvs`网卡中，存在`192.168.10.114`临时ip。
    1. 删除`kube-ipvs`网卡上的临时ip
    2. `ip a del 192.168.10.114/32 dev kube-ipvs0`
    3. 发现过一阵子还会出现，直接循环删除，方便继续排查
    4. `while  true ; do ip a del 192.168.10.114/32 dev kube-ipvs0 && sleep  1 ; done`
    5. 116节点上面也会出现该临时ip，同上处理
- 那么`192.168.10.114`IP为何会在`kube-ipvs`网卡上？
- 通过上面循环删除临时ip后，集群状态恢复，但是没有根本解决，`kube-ipvs`网卡，是和kubeproxy、svc做关联的，检查proxy 状态正常
- 最后突然想起来，上午手贱，测了一个tomcat，给暴露的svc类型是ClusterIP，但是给配置了`externalIPs`，地址就是114

![](/images/posts/media/16799929082891.jpg)


### 故障定位
由于使用tomcat的svc使用了externalIP，地址为192.168.10.114导致。
正常，该IP应填写为集群之外的IP地址，这样才会通过映射来进行转发到真正的endpoint里面

![](/images/posts/media/16799929969549.jpg)

该external IP 会在每个集群的节点上面，均会增加一个虚拟IP，就是`192.168.10.114`，该地址是只能在集群节点之内进行访问的，也就是为啥，我通过终端ssh连接`192.168.10.114`的时候，不会连接到集群中的node节点，而是连接到真实的`192.168.10.114`master节点。
同时，由于每个节点都会增加一个`192.168.10.114`IP，kubelet在和apiserver进行交互的时候，真实交互的是节点自身，并没有和master节点的进行交互，也就是为啥，master节点看到所有节点都是NotReady的原因。


### 问题解决
删除此段
```bash
kubectl edit svc tomcat
```

![](/images/posts/media/16799930485643.jpg)

再次查看，ip只有一个

![](/images/posts/media/16799931068732.jpg)

隧道正常建立

![](/images/posts/media/16799931866954.jpg)
![](/images/posts/media/16799935136768.jpg)
