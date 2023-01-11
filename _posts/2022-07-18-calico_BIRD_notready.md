---
layout: post
title: calico is not ready
date: 2022-07-18
tags: kubernetes报错
---
### 问题现象
1. calico 一直 Crash
2. 报错信息如下

```
Health endpoint failed, trying to restart it... error=listen tcp: lookup localhost on 8.8.8.8
Readiness probe failed: calico/node is not ready: BIRD is not ready: Error querying BIRD: unable to connect to BIRDv4 socket: dial unix /var/run/calico/bird.ctl: connect: connection refused
enp61s0f0:
```
### 原因
> 检查发现，该节点上面的 `/etc/hosts` 文件中，本地回环地址被禁用，calico 无法通过localhost 与宿主机进行交互

### 解决方法：
取消注释，重启pod
