---
layout: post
title: ipmi-exporter一直处于ContainerCreating
date: 2022-04-12 
tags: kubernetes报错
---
### 问题现象
![](/images/posts/error/ipmi.png)

### 原因
> 检查发现，该pod在启动的时候，需要挂载宿主机的 /dev/ipmi0 这个设备，由于对应机器上面没有这个路径，导致pod挂载不成功，无法启动

ipmi是一个针对物理机的硬件暴露指标数据的模块，虚机没有。如果是物理机执行以下命令加载下模块，就可以挂载成功

### 解决方法：
```
$ lsmod  | grep ipmi
# 此时的执行结果为空（说明没有/dev/ipmi 设备）

$ modprobe ipmi_devintf
$ modprobe ipmi_watchdog

$ ls /dev/ipmi0  #检查有没有成功加载
```
参考：http://blog.sina.com.cn/s/blog_413d250e0102y3z2.html