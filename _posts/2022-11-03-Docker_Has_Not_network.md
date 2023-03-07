---
layout: post
title: Docker 启动后，没有生成Iptables规则表
date: 2022-11-03 
tags: 常见问题处理
---
### 问题现象
> 启动容器后，无法正常跨主机出网

### 问题解决
**临时解决方案**
手动创建Iptables规则表
```
iptables -t nat -N DOCKER
iptables -t nat -A PREROUTING -m addrtype --dst-type LOCAL -j DOCKER
iptables -t nat -A OUTPUT ! -d 127.0.0.0/8 -m addrtype --dst-type LOCAL -j DOCKER
iptables -t nat -A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
iptables -t nat -A POSTROUTING -s 173.20.1.0/24 ! -o docker0 -j MASQUERADE
iptables -t nat -A DOCKER -i docker0 -j RETURN
```
**永久解决方案**
待补充。。。