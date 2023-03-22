---
layout: post
title: Pod 网络异常
date: 2023-03-22
tags: 常见问题处理
---

### Pod 网络异常
网络异常大概分为如下几类：
- **网络不可达**,主要现象为ping不通，其可能原因为：
  - 源端和目的端防火墙(iptables,selinux)限制
  - 网络路由配置不正确
  - 源端和目的端的系统负载过高，网络连接数满，网卡队列满
  - 网络链路故障
- **端口不可达**,主要现象为可以ping通，但是telnet端口不通，其可能原因为：
  - 源端和目的端防火墙限制
  - 源端和目的端的系统负载过高，网络连接数满，网卡队列满，端口耗尽
  - 目的端应用未正常监听导致（应用未启动，或监听为127.0.0.1等）
- **DNS解析异常**,主要现象为基础网络可以连通，访问域名报错无法解析，访问IP可以正常连通。其可能原因为：
  - Pod的DNS配置不正确
  - DNS服务异常
  - Pod与DNS服务通讯异常
- **大数据包丢包**,主要现象为基础网络和端口均可连通，小数据包收发无异常，大数据包丢包，可能原因为：
  - 可使用`ping -s`指定数据包大小进行测试
  - 数据包的大小超过了Docker、CNI插件、或者宿主机网卡的MTU值
- **CNI异常**,主要现象为Node可以通，但Pod无法访问集群地址，可能原因有：
  - kube-proxy 服务异常，没有生成iptables策略或者ipvs规则导致无法访问
  - CIDR耗尽，无法为Node注入PodCIDR导致CNI插件异常
  - 其他CNI插件问题


整个pod网络异常分类可以如下图所示

![](/images/posts/media/troublenetwork.png)

总结一下，Pod 最常见的网络故障有，网络不可达（ping 不通）；端口不可达（telnet 不通）；DNS 解析异常（域名不通）与大数据包丢失（大包不通）。


### 常用的网络排查工具
#### tcpdump
tcpdump 网络嗅探器，将强大和简单结合到一个单一的命令行界面中，能够将网络中的报文抓取，输出到屏幕或者记录到文件中。

各系统下的安装

Ubuntu/Debian: tcpdump  ；apt-get install -y tcpdump

Centos/Fedora: tcpdump ；yum install -y tcpdump

Apline：tcpdump ；apk add tcpdump --no-cache

查看指定接口上的所有通讯,语法

参数|说明
---|---
-i |[interface]	
-w |[flle]	第一个 n 表示将地址解析为数字格式而不是主机名，第二个 N 表示将端口解析为数字格式而不是服务名
-n	|不显示 IP 地址
-X	|hex and ASCII
-A	|ASCII（实际上是以人类可读懂的包进行显示）
-XX	|
-v	|详细信息
-r	|读取文件而不是实时抓包
||关键字
type	|host（主机名，域名，IP 地址）, net, port, portrange
direction	|src, dst, src or dst , src and ds
protocol	|ether, ip，arp, tcp, udp, wlan


**捕获所有网络接口**
```
tcpdump -D
```
**按 IP 查找流量**
最常见的查询之一 host，可以看到来往于 1.1.1.1 的流量。
```
tcpdump host 1.1.1.1
```
**按源 / 目的 地址过滤**
如果只想查看来自 / 向某方向流量，可以使用 src 和 dst。
```
tcpdump src|dst 1.1.1.1
```
**通过网络查找数据包**
使用 net 选项，来要查找出 / 入某个网络或子网的数据包。
```
tcpdump net 1.2.3.0/24
```
**使用十六进制输出数据包内容**
hex 可以以 16 进制输出包的内容
```
tcpdump -c 1 -X icmp
```
**查看特定端口的流量**
使用 port 选项来查找特定的端口流量。
```
tcpdump port 3389
tcpdump src port 1025
```
**查找端口范围的流量**
```
tcpdump portrange 21-23
```
**过滤包的大小**
如果需要查找特定大小的数据包，可以使用以下选项。你可以使用 less，greater。
```
tcpdump less 32
tcpdump greater 64
tcpdump <= 128
```
**捕获流量输出为文件**
`-w`可以将数据包捕获保存到一个文件中以便将来进行分析。这些文件称为 PCAP（PEE-cap）文件，它们可以由不同的工具处理，包括 Wireshark 。
```
tcpdump port 80 -w capture_file
```

**组合条件**
tcpdump 也可以结合逻辑运算符进行组合条件查询

ANDand or &&
ORor or ||
EXCEPTnot or !
```
tcpdump -i eth0 -nn host 220.181.57.216 and 10.0.0.1  # 主机之间的通讯
tcpdump -i eth0 -nn host 220.181.57.216 or 10.0.0.1
# 获取10.0.0.1与 10.0.0.9或 10.0.0.1 与10.0.0.3之间的通讯
tcpdump -i eth0 -nn host 10.0.0.1 and \(10.0.0.9 or 10.0.0.3\)
```
**原始输出**
并显示人类可读的内容进行输出包（不包含内容）。
```
tcpdump -ttnnvvS -i eth0
tcpdump -ttnnvvS -i eth0
```
**IP 到端口**
让我们查找从某个 IP 到端口任何主机的某个端口所有流量。
```
tcpdump -nnvvS src 10.5.2.3 and dst port 3389
```
**去除特定流量**
可以将指定的流量排除，如这显示所有到 192.168.0.2 的 非 ICMP 的流量。
```
tcpdump dst 192.168.0.2 and src net and not icmp
```
来自非指定端口的流量，如，显示来自不是 SSH 流量的主机的所有流量。
```
tcpdump -vv src mars and not dst port 22
```
**选项分组**
在构建复杂查询时，必须使用单引号 '。单引号用于忽略特殊符号 () ，以便于使用其他表达式（如 host, port, net 等）进行分组。
```
tcpdump 'src 10.0.2.4 and (dst port 3389 or 22)'
```
**过滤 TCP 标记位**
TCP RST

The filters below find these various packets because tcp[13] looks at offset 13 in the TCP header, the number represents the location within  the byte, and the !=0 means that the flag in question is set to 1, i.e.  it’s on.
```
tcpdump 'tcp[13] & 4!=0'
tcpdump 'tcp[tcpflags] == tcp-rst'
```
TCP SYN
```
tcpdump 'tcp[13] & 2!=0'
tcpdump 'tcp[tcpflags] == tcp-syn'
```
同时忽略 SYN 和 ACK 标志的数据包
```
tcpdump 'tcp[13]=18'
```
TCP URG
```
tcpdump 'tcp[13] & 32!=0'
tcpdump 'tcp[tcpflags] == tcp-urg'
```
TCP ACK
```
tcpdump 'tcp[13] & 16!=0'
tcpdump 'tcp[tcpflags] == tcp-ack'
```

TCP PSH
```
tcpdump 'tcp[13] & 8!=0'
tcpdump 'tcp[tcpflags] == tcp-push'
```
TCP FIN
```
tcpdump 'tcp[13] & 1!=0'
tcpdump 'tcp[tcpflags] == tcp-fin'
```
**查找 http 包**
查找 user-agent 信息
```
tcpdump -vvAls0 | grep 'User-Agent:'
```
查找只是 GET 请求的流量
```
tcpdump -vvAls0 | grep 'GET'
```
查找 http 客户端 IP
```
tcpdump -vvAls0 | grep 'Host:'
```
查询客户端 cookie
```
tcpdump -vvAls0 | grep 'Set-Cookie|Host:|Cookie:'
```
**查找 DNS 流量**
tcpdump -vvAs0 port 53
**查找对应流量的明文密码**
```
tcpdump port http or port ftp or port smtp or port imap or port pop3 or port telnet -lA | egrep -i -B5 'pass=|pwd=|log=|login=|user=|username=|pw=|passw=|passwd= |password=|pass:|user:|username:|password:|login:|pass |user '
```
**wireshark 追踪流**
wireshare 追踪流可以很好的了解出在一次交互过程中都发生了那些问题。

wireshare 选中包，右键选择 “追踪流“ 如果该包是允许的协议是可以打开该选项的

**关于抓包节点和抓包设备**
如何抓取有用的包，以及如何找到对应的接口，有以下建议

抓包节点：

通常情况下会在源端和目的端两端同时抓包，观察数据包是否从源端正常发出，目的端是否接收到数据包并给源端回包，以及源端是否正常接收到回包。如果有丢包现象，则沿网络链路上各节点抓包排查。例如，A 节点经过 c 节点到 B 节点，先在 AB 两端同时抓包，如果 B 节点未收到 A 节点的包，则在 c 节点同时抓包。

抓包设备：

对于 Kubernetes 集群中的 Pod，由于容器内不便于抓包，通常视情况在 Pod 数据包经过的 veth 设备，docker0 网桥，CNI 插件设备（如 cni0，flannel.1 etc..）及 Pod 所在节点的网卡设备上指定 Pod IP 进行抓包。选取的设备根据怀疑导致网络问题的原因而定，比如范围由大缩小，从源端逐渐靠近目的端，比如怀疑是 CNI 插件导致，则在 CNI 插件设备上抓包。从 pod 发出的包逐一经过 veth 设备，cni0 设备，flannel0，宿主机网卡，到达对端，抓包时可按顺序逐一抓包，定位问题节点。

需要注意在不同设备上抓包时指定的源目 IP 地址需要转换，如抓取某 Pod 时，ping {host} 的包，在 veth 和 cni0 上可以指定 Pod IP 抓包，而在宿主机网卡上如果仍然指定 Pod IP 会发现抓不到包，因为此时 Pod IP 已被转换为宿主机网卡 IP。

下图是一个使用 VxLAN 模式的 flannel 的跨界点通讯的网络模型，在抓包时需要注意对应的网络接口

![](/images/posts/media/vxlan.png)

#### nsenter
nsenter 是一款可以进入进程的名称空间中。例如，如果一个容器以非 root 用户身份运行，而使用 docker exec 进入其中后，但该容器没有安装 sudo 或未 netstat ，并且您想查看其当前的网络属性，如开放端口，这种场景下将如何做到这一点？nsenter 就是用来解决这个问题的。

nsenter (namespace enter) 可以在容器的宿主机上使用 nsenter 命令进入容器的命名空间，以容器视角使用宿主机上的相应网络命令进行操作。当然需要拥有 root 权限


Ubuntu/Debian: util-linux  ；apt-get install -y util-linux

Centos/Fedora: util-linux ；yum install -y util-linux

Apline：util-linux ；apk add util-linux --no-cache

nsenter 的 c 使用语法为，nsenter -t pid -n <commond>，-t 接 进程 ID 号，-n 表示进入名称空间内，<commond> 为执行的命令。

实例：如我们有一个 Pod 进程 ID 为 30858，进入该 Pod 名称空间内执行 ifconfig ，如下列所示
```
$ ps -ef|grep tail
root      17636  62887  0 20:19 pts/2    00:00:00 grep --color=auto tail
root      30858  30838  0 15:55 ?        00:00:01 tail -f

$ nsenter -t 30858 -n ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1480
        inet 192.168.1.213  netmask 255.255.255.0  broadcast 192.168.1.255
        ether 5e:d5:98:af:dc:6b  txqueuelen 0  (Ethernet)
        RX packets 92  bytes 9100 (8.8 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 92  bytes 8422 (8.2 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 5  bytes 448 (448.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 5  bytes 448 (448.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

net1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.1.0.201  netmask 255.255.255.0  broadcast 10.1.0.255
        ether b2:79:f9:dd:2a:10  txqueuelen 0  (Ethernet)
        RX packets 228  bytes 21272 (20.7 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 216  bytes 20272 (19.7 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

**如何定位 Pod 名称空间**

首先需要确定 Pod 所在的节点名称
```
$ kubectl get pods -owide |awk '{print $1,$7}'
NAME NODE
netbox-85865d5556-hfg6v master-machine
netbox-85865d5556-vlgr4 node01
```
如果 Pod 不在当前节点还需要用 IP 登录则还需要查看 IP（可选）
```
$ kubectl get pods -owide |awk '{print $1,$6,$7}'
NAME IP NODE
netbox-85865d5556-hfg6v 192.168.1.213 master-machine
netbox-85865d5556-vlgr4 192.168.0.4 node01
```
接下来，登录节点，获取容器 lD，如下列所示，每个 pod 默认有一个 pause 容器，其他为用户 yaml 文件中定义的容器，理论上所有容器共享相同的网络命名空间，排查时可任选一个容器。
```
$ docker ps |grep netbox-85865d5556-hfg6v
6f8c58377aae   f78dd05f11ff                                                    "tail -f"                45 hours ago   Up 45 hours             k8s_netbox_netbox-85865d5556-hfg6v_default_4a8e2da8-05d1-4c81-97a7-3d76343a323a_0
b9c732ee457e   registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.1   "/pause"                 45 hours ago   Up 45 hours             k8s_POD_netbox-85865d5556-hfg6v_default_4a8e2da8-05d1-4c81-97a7-3d76343a323a_0
```
接下来获得获取容器在节点系统中对应的进程号，如下所示
```
$ docker inspect --format "{{ .State.Pid }}" 6f8c58377aae
30858
```
最后就可以通过 nsenter 进入容器网络空间执行命令了

### Pod网络排查流程
Pod 网络异常时排查思路，可以按照下图所示

![](/images/posts/media/podtrouble.png)

### 案例
#### 扩容节点访问 service 地址不通
测试环境 k8s 节点扩容后无法访问集群 clusterlP 类型的 registry 服务

环境信息：

IP	|Hostname|	role
---|---|---
10.153.204.15	|yq01-aip-aikefu12	|worknode 节点（本次扩容的问题节点）
10.153.203.14	|yq01-aip-aikefu31	|master 节点
10.61.187.42	|yq01-aip-aikefu2746f8e9	|master 节点
10.61.187.48	|yq01-aip-aikefu30b61e25	|master 节点（本次 registry 服务 pod 所在 节点）

- cni 插件：flannel vxlan
- kube-proxy 工作模式为 iptables
- registry 服务
  - 单实例部署在 10.61.187.48:5000
  - Pod IP：10.233.65.46，
  - Cluster IP：10.233.0.100

现象：

- 所有节点之间的 pod 通信正常
- 任意节点和 Pod curl registry 的 Pod 的 IP:5000 均可以连通
- 新扩容节点 10.153.204.15 curl registry 服务的 Cluster lP 10.233.0.100:5000 不通，其他节点 curl 均可以连通

分析思路：

- 根据现象 1 可以初步判断 CNI 插件无异常
- 根据现象 2 可以判断 registry 的 Pod 无异常
- 根据现象 3 可以判断 registry 的 service 异常的可能性不大，可能是新扩容节点访问 registry 的 service 存在异常

怀疑方向：

- 问题节点的 kube-proxy 存在异常
- 问题节点的 iptables 规则存在异常
- 问题节点到 service 的网络层面存在异常

排查过程：

- 排查问题节点的 `kube-proxy`
- 执行 `kubectl get pod -owide -nkube-system l grep kube-proxy` 查看 kube-proxy Pod 的状态，问题节点上的 kube-proxy Pod 为 running 状态
- 执行 `kubecti logs <nodename> <kube-proxy pod name> -nkube-system `查看问题节点 kube-proxy 的 Pod 日志，没有异常报错
- 在问题节点操作系统上执行 iptables -S -t nat 查看 iptables 规则

排查过程：

确认存在到 registry 服务的 Cluster lP 10.233.0.100 的 KUBE-SERVICES 链，跳转至 KUBE-SVC-* 链做负载均衡，再跳转至 KUBE-SEP-* 链通过 DNAT 替换为服务后端 Pod 的 IP 10.233.65.46。因此判断 iptables 规则无异常执行 route-n 查看问题节点存在访问 10.233.65.46 所在网段的路由，如图所示

![](/images/posts/media/route1.png)

查看对端的回程路由

![](/images/posts/media/route2.png)


以上排查证明问题原因不是 cni 插件或者 kube-proxy 异常导致，因此需要在访问链路上抓包，判断问题原因、问题节点执行 curl 10.233.0.100:5000，在问题节点和后端 pod 所在节点的 flannel.1 上同时抓包发包节点一直在重传，Cluster lP 已 DNAT 转换为后端 Pod IP，如图所示(抓包过程，发送端)

![](/images/posts/media/tcpdump1.png)

后端 Pod（ registry 服务）所在节点的 flannel.1 上未抓到任何数据包，如图所示(抓包过程，服务端)

![](/images/posts/media/tcpdump2.png)

请求 service 的 ClusterlP 时，在两端物理机网卡抓包，发包端如图所示，封装的源端节点 IP 是 `10.153.204.15`，但一直在重传(包传送过程，发送端)

![](/images/posts/media/tcpdump3.png)

收包端收到了包，但未回包，如图所示(包传送过程，服务端)

![](/images/posts/media/tcpdump4.png)

由此可以知道，NAT 的动作已经完成，而只是后端 Pod（ registry 服务）没有回包，接下来在问题节点执行 `curl 10.233.65.46:5000`，在问题节点和后端（ registry 服务）Pod 所在节点的 flannel.1 上同时抓包，两节点收发正常，发包如图所示(正常包发送端)

![](/images/posts/media/tcpdump5.png)


正常包接收端

![](/images/posts/media/tcpdump6.png)

接下来在两端物理机网卡接口抓包，因为数据包通过物理机网卡会进行 vxlan 封装，需要抓 vxlan 设备的 8472 端口，发包端如图所示

发现网络链路连通，但封装的 IP 不对，封装的源端节点 IP 是 10.153.204.228，但是存在问题节点的 IP 是 10.153.204.15（问题节点物理机网卡接口抓包）

![](/images/posts/media/tcpdump7.png)

后端 Pod 所在节点的物理网卡上抓包，注意需要过滤其他正常节点的请求包，如图所示；发现收到的数据包，源地址是 10.153.204.228，但是问题节点的 IP 是 10.153.204.15。(对端节点物理机网卡接口抓包)

![](/images/posts/media/tcpdump8.png)

此时问题以及清楚了，是一个 Pod 存在两个 IP，导致发包和回包时无法通过隧道设备找到对端的接口，所以发可以收到，但不能回。

问题节点执行 ip addr，发现网卡 enp26s0f0 上配置了两个 IP，如图所示(问题节点 IP)

![](/images/posts/media/ipaddr.png)

进一步查看网卡配置文件，发现网卡既配置了静态 IP，又配置了 dhcp 动态获取 IP。如图所示(问题节点网卡配置)

![](/images/posts/media/networkconfiguration.png)

最终定位原因为问题节点既配置了 dhcp 获取 IP，又配置了静态 IP，导致 IP 冲突，引发网络异常

解决方法：修改网卡配置文件 `/etc/sysconfig/network-scripts/ifcfg-enp26s0f0`里 `BOOTPROTO="dhcp"`为 `BOOTPROTO="none"`；重启 docker 和 kubelet 问题解决。

