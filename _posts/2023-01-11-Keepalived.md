---
layout: post
title: keepalived
date: 2023-01-11
tags: 集群
---
关于HA目前有很多的解决方案，比如Heartbeat、keepalived等，各有优缺点。本文以keepalived为例做说明

### 1.keepalived 概述
keepalived 的作用是检测后端TCP服务的状态，如果有一台提供TCP服务的后端节点司机，或工作出现故障，keepalived及时检测到，并将有故障的节点从系统中剔除，当提供TCP服务的节点恢复并且正常提供服务后keepalived自动将提供TCP服务的节点加入到激荡中，这些工作全部由keepalived自动完成，不需要人工干涉，需要人工做的只是修复故障的服务器。

keepalived可以工作在TCP/IP协议栈的IP层、TCP层及应用层:

- IP 层
    
    keepalived 使用的IP 的方式工作时，会定期向服务器集群中的服务器发送一个ICMP的数据包，如果发现某台服务的IP地址没有激活，keepalived便报告这台服务器异常，并将其从集群中剔除。常见的场景为某台机器网卡损坏或服务器被非法关机。IP层的工作方式是以服务器的IP地址是否有效作为服务器工作正常与否的标准。
- TCP 层

    这种工作模式主要是以 TCP 后台服务的状态来确定后端服务器是否工作正常。如 MYSQL默认端口一般为3306，如果keepalived检测到3306无法登录或拒绝连接，则认为后端服务异常，则keepalived将把这台服务器从集群中剔除。
- 应用层

    如keepalived工作在应用层了，此时keepalived将根据用户的设定检查服务器程序的运行是否正常，如果与用户的设定不相符，则keepalived将把服务器从集群中剔除。

以上几种方式可以通过keepalived的配置文件实现。
### 2.keepalived 安装与配置
本节实现的功能为访问172.32.10.18 的web服务时，自动代理到后端的真实服务器 192.168.10.114 和 192.168.10.121，keepalived主机为172.32.10.14，备机为192.168.10.68。

最新的版本可以在https://keepalived.org 获取，本示例采用的版本为2.2.2。

【两个节点都需操作】
```
[root@vulhub keepalived]# tar xf keepalived-2.2.2.tar.gz

[root@vulhub keepalived]# cd keepalived-2.2.2/

# 安装编译环境的依赖
[root@vulhub keepalived-2.2.2]# yum -y install gcc openssl-devel libnl3-devel net-snmp-devel

[root@vulhub keepalived-2.2.2]# ./configure --prefix=/usr/local/keepalived

[root@vulhub keepalived-2.2.2]# make

[root@vulhub keepalived-2.2.2]# make install

[root@vulhub keepalived-2.2.2]# ln -s /usr/local/keepalived/etc/keepalived/ /etc/
```
经过上面的步骤，keepalived已经安装完成，安装路径为`/usr/local/keepalived`，备节点操作步骤同主节点。


```
# 主节点配置文件
[root@vulhub keepalived-2.2.2]# cat -n /etc/keepalived/keepalived.conf
     1	! Configuration File for keepalived
     2
     3	global_defs {
     4	   router_id vulhub  # 节点名称  
     5	}
     6
     7	vrrp_instance VI_1 {
     8      # 指定该节点为主节点 备用节点上需要设置为 BACKUP
     9 	    state MASTER
    10      # 绑定虚拟 IP 的网络接口
    11	    interface ens192
    12      # VRRP组名，两个节点需要设置一样，以指明各个节点属于同一VRRP组
    13	    virtual_router_id 51
    14      # 主节点的优先级，数值在1~254，注意从节点必须比主节点优先级低
    15	    priority 50
    16      # 组播信息发送间隔，两个节点需要设置一样
    17	    advert_int 1
    18      # 设置验证信息，两个节点需要一致
    19	    authentication {
    20	        auth_type PASS
    21	        auth_pass 1111
    22	    }
    23      # 指定虚拟IP，两个节点需要设置一样
    24	    virtual_ipaddress {
    25	        172.32.10.18
    26	    }
    27	}
    28  # 虚拟IP服务
    29	virtual_server 172.32.10.18 80 {
    30      # 设定检查实际服务器的间隔
    31	    delay_loop 6
    32      # 指定LVS算法
    32	    lb_algo rr
    33      # 指定LVS模式
    33	    lb_kind NAT
    34      # 持久连接设置，会话保持时间
    35	    persistence_timeout 50
    36      # 转发协议为TCP
    37	    protocol TCP
    38
    39      # 后端实际TCP服务配置
    40	    real_server 192.168.10.114 80 {
    41	        weight 1
    42	    }
    43      # 后端实际TCP服务配置
    44	    real_server 192.168.10.121 80 {
    45	        weight 1
    46	    }
    47	}

```
备节点配置大部分配置同主节点，不同处如下所示

```

```

```
$ ./configure  --prefix=/data/keepalived
$ make && make install 
$ cp /data/keepalived/sbin/keepalived /usr/sbin
$ mkdir /etc/keepalived
$ cp /data/keepalived/etc/keepalived/keepalived.conf   /etc/keepalived/
$ vim /etc/keepalived/keepalived.conf

! Configuration File for keepalived
global_defs {
   router_id k8s-master01  # 节点名称
}

vrrp_script chk_haproxy {
    script "/bin/bash -c 'if [[ $(netstat -nlp | grep 9443) ]]; then exit 0; else exit 1; fi'"
    interval 2
    fall 2
    weight 50
}

vrrp_instance VI_11 {
    state MASTER
    interface em1  # 网卡名
    virtual_router_id 22
    priority 100  # 优先级
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.20.111  # 虚拟地址
    }
    track_script {
        chk_haproxy
    }
}

$ systemctl enable keepalived.service && systemctl start keepalived.service

```
### 3.keepalived 启动与测试
### 4.keepalived 抢占模式
`keepalived`配置抢占模式就是：当`keepalived`的`master`节点服务器挂了之后vip漂移到了备节点，当主节点恢复后主动将vip再次抢回来。

`keepalived`默认就是抢占模式。


可以看的出来，抢占模式实现了两次vip的漂移，如果在业务场景中我们觉得第二次的vip漂移会主节点是多余的。
> 在数据库的双主模式情况下，如果使用的默认抢占模式配置的HA，当数据库`master`节点宕机之后，由`backup`节点顶上来，此时，`master`节点由于是宕机状态，数据肯定是不会同步的，当`master`节点启动之后，数据不是最新的，两个节点数据不会一致。这时候业务方连接到数据库，就会出现数据不一致现象。为了避免这种现象出现，就必须要配置非抢占模式。

我们可以将keepalived配置非抢占模式
```
非抢占式：
master故障--->backup顶上--->master恢复不抢占vip--->backup拥有vip继续工作
```

非抢占模式配置：
```
1、两个节点的state都必须配置为BACKUP(官方建议)
2、两个节点都在vrrp_instance中添加nopreempt参数（其实优先级高的配置nopreempt参数即可）
3、其中一个节点的优先级必须要高于另外一个节点的优先级。

两台服务器都启用nopreempt后，必须修改角色状态统一为BACKUP，唯一的区分就是优先级。

官方对nopreempt参数解释：高优先级VRRP实例通常会抢占低优先级VRRP实例，
"nopreempt"参数将停止优先级更高的机器在抢占vip，并允许较低优先级的机器保持为master。
注意:要使nopreempt参数起作用，初始状态不能是MASTER。
```