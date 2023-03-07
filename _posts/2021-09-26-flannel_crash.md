---
layout: post
title: kube-flannel CrashLoopBackOff
date: 2021-09-26 
tags: 常见问题处理
---
### 背景
三个节点的集群，其中一个节点的 flannal 异常

> 日志信息：Failed to find any valid interface to use: failed to get default interface: Unable to find default route

### 处置思路
```
# 登录到对应机器，发现该机器没有网关，找不到路由
$ ip r |grep default
 
$ cat /etc/sysconfig/network-scripts/ifcfg-eth0 # 添加网关
DEVICE=eth0
BOOTPROTO=static
IPADDR=192.168.1.14
NETMASK=255.255.255.0
ONBOOT=yes
GATEWAY=192.168.1.1 
 
 
$ systemctl restart network 重启解决
```