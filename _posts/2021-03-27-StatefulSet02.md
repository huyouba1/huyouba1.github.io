---
layout: post
title: redis cluster三主三从
date: 2021-03-27
tags: StatefulSet
---
### 一、Redis 介绍
Redis 代表`REmote DIctionary Server`是一种开源的内存中数据存储，通常用作数据库，缓存或消息代理。它可以存储和操作高级数据类型，例如列表，地图，集合和排序集合。由于Redis接受多种格式的密钥，因此可以在服务器上执行操作，从而减少了客户端的工作量。它仅将磁盘用于持久性，而将数据库完全保存在内存中。Redis是一种流行的数据存储解决方案，并被`GitHub，Pinterest，Snapchat，Twitter，StackOverflow，Flickr`等技术巨头所使用。

### 二、为什么使用 Redis
- 它的速度非常快。它是用 ANSI C 编写的，并且可以在 POSIX 系统上运行，例如 Linux，Mac OS X 和 Solaris。
- Redis 通常被排名为最流行的键/值数据库和最流行的与容器一起使用的 NoSQL 数据库。
- 其缓存解决方案减少了对云数据库后端的调用次数。
- 应用程序可以通过其客户端 API 库对其进行访问。
- 所有流行的编程语言都支持 Redis。
- 它是开源且稳定的。

### 三、什么是 Redis 集群
Redis Cluster 是一组 Redis 实例，旨在通过对数据库进行分区来扩展数据库，从而使其更具弹性。群集中的每个成员（无论是主副本还是辅助副本）都管理哈希槽的子集。如果主机无法访问，则其从机将升级为主机。在由三个主节点组成的最小 Redis 群集中，每个主节点都有一个从节点（以实现最小的故障转移），每个主节点都分配有一个介于 0 到 16,383 之间的哈希槽范围。节点 A 包含从 0 到 5000 的哈希槽，节点 B 从 5001 到 10000，节点 C 从 10001 到 16383。群集内部的通信是通过内部总线进行的，使用协议传播有关群集的信息或发现新节点。

### 四、在 Kubernetes 中部署 Redis 集群

> 在Kubernetes中部署Redis集群面临挑战，因为每个 Redis 实例都依赖于一个配置文件，该文件可以跟踪其他集群实例及其角色。为此，我们需要结合使用Kubernetes StatefulSets和PersistentVolumes。

**创建 statefulset 类型资源**

```
[root@k8s-master redis-sts]# cat redis-sts.yaml 
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-cluster
data:
  update-node.sh: |
    #!/bin/sh
    REDIS_NODES="/data/nodes.conf"
    sed -i -e "/myself/ s/[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}/${POD_IP}/" ${REDIS_NODES}
    exec "$@"
  redis.conf: |+
    cluster-enabled yes
    cluster-require-full-coverage no
    cluster-node-timeout 15000
    cluster-config-file /data/nodes.conf
    cluster-migration-barrier 1
    appendonly yes
    protected-mode no
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-cluster
spec:
  serviceName: redis-cluster
  replicas: 6
  selector:
    matchLabels:
      app: redis-cluster
  template:
    metadata:
      labels:
        app: redis-cluster
    spec:
      containers:
      - name: redis
        image: redis:5.0.1-alpine
        ports:
        - containerPort: 6379
          name: client
        - containerPort: 16379
          name: gossip
        command: ["/conf/update-node.sh", "redis-server", "/conf/redis.conf"]
        env:
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        volumeMounts:
        - name: conf
          mountPath: /conf
          readOnly: false
        - name: data
          mountPath: /data
          readOnly: false
      volumes:
      - name: conf
        configMap:
          name: redis-cluster
          defaultMode: 0755
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
      storageClassName: managed-nfs-storage    ##准备好strageclass
      
[root@k8s-master redis-sts]# kubectl apply -f redis-sts.yaml 
configmap/redis-cluster created
statefulset.apps/redis-cluster created

[root@k8s-master redis-sts]# kubectl get pods -l app=redis-cluster
NAME              READY   STATUS    RESTARTS   AGE
redis-cluster-0   1/1     Running   0          79m
redis-cluster-1   1/1     Running   0          78m
redis-cluster-2   1/1     Running   0          78m
redis-cluster-3   1/1     Running   0          78m
redis-cluster-4   1/1     Running   0          77m
redis-cluster-5   1/1     Running   0          77m
```
**创建 service**
```
[root@k8s-master redis-sts]# cat redis-svc.yaml 
apiVersion: v1
kind: Service
metadata:
  name: redis-cluster
spec:
  type: ClusterIP
  ports:
  - port: 6379
    targetPort: 6379
    name: client
  - port: 16379
    targetPort: 16379
    name: gossip
  selector:
    app: redis-cluster

[root@k8s-master redis-sts]# kubectl apply -f redis-svc.yaml 
service/redis-cluster created

[root@k8s-master redis-sts]# kubectl get svc redis-cluster 
NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)              AGE
redis-cluster   ClusterIP   10.108.58.105   <none>        6379/TCP,16379/TCP   86m
```

**初始化 redis cluster**
> 下一步是形成Redis集群。为此，我们运行以下命令并键入yes以接受配置。前三个节点成为主节点，后三个节点成为从节点。

```
正常情况下，执行下面命令就可初始化集群，如该命令无法执行成功，参考后面即可。
[root@k8s-master redis-sts]# kubectl exec -it redis-cluster-0 -- redis-cli --cluster create --cluster-replicas 1 $(kubectl get pods -l app=redis-cluster -o jsonpath='{range.items[*]}{.status.podIP}:6379 ')
Could not connect to Redis at :6379: Name does not resolve
command terminated with exit code 1
```
```
[root@k8s-master redis-sts]# kubectl get pod -o wide |grep redis   //获取各个pod的ip
redis-cluster-0                           1/1     Running   0          85m   10.244.58.215   k8s-node02   <none>           <none>
redis-cluster-1                           1/1     Running   0          85m   10.244.85.204   k8s-node01   <none>           <none>
redis-cluster-2                           1/1     Running   0          84m   10.244.58.214   k8s-node02   <none>           <none>
redis-cluster-3                           1/1     Running   0          84m   10.244.85.203   k8s-node01   <none>           <none>
redis-cluster-4                           1/1     Running   0          84m   10.244.58.223   k8s-node02   <none>           <none>
redis-cluster-5                           1/1     Running   0          84m   10.244.85.206   k8s-node01   <none>           <none>

[root@k8s-master redis-sts]# kubectl exec -it redis-cluster-0 -- redis-cli --cluster create --cluster-replicas 1  10.244.58.215:6379 10.244.85.204:6379 10.244.58.214:6379 10.244.85.203:6379 10.244.58.223:6379 10.244.85.206:6379
>>> Performing hash slots allocation on 6 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
Adding replica 10.244.85.203:6379 to 10.244.58.215:6379
Adding replica 10.244.58.223:6379 to 10.244.85.204:6379
Adding replica 10.244.85.206:6379 to 10.244.58.214:6379
M: 43c1cc57340ed703852bc4bbc4bc906d33c080df 10.244.58.215:6379
   slots:[0-5460] (5461 slots) master
M: d0b7d0d893ab6bdb3435f633e656cf8b9b5eecc4 10.244.85.204:6379
   slots:[5461-10922] (5462 slots) master
M: 29da99ee264d9041a5ad08d42ce5be418f1a781b 10.244.58.214:6379
   slots:[10923-16383] (5461 slots) master
S: b8515ea3fff4b8bdeb930a5d1afb11aab638456c 10.244.85.203:6379
   replicates 43c1cc57340ed703852bc4bbc4bc906d33c080df
S: e969c6723e111f352fbf628b64db912ebb0385d7 10.244.58.223:6379
   replicates d0b7d0d893ab6bdb3435f633e656cf8b9b5eecc4
S: a0aa35eda57bd2e92f5ff76909568e8ad588a37a 10.244.85.206:6379
   replicates 29da99ee264d9041a5ad08d42ce5be418f1a781b
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join
........
>>> Performing Cluster Check (using node 10.244.58.215:6379)
M: 43c1cc57340ed703852bc4bbc4bc906d33c080df 10.244.58.215:6379
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
S: e969c6723e111f352fbf628b64db912ebb0385d7 10.244.58.223:6379
   slots: (0 slots) slave
   replicates d0b7d0d893ab6bdb3435f633e656cf8b9b5eecc4
S: a0aa35eda57bd2e92f5ff76909568e8ad588a37a 10.244.85.206:6379
   slots: (0 slots) slave
   replicates 29da99ee264d9041a5ad08d42ce5be418f1a781b
S: b8515ea3fff4b8bdeb930a5d1afb11aab638456c 10.244.85.203:6379
   slots: (0 slots) slave
   replicates 43c1cc57340ed703852bc4bbc4bc906d33c080df
M: d0b7d0d893ab6bdb3435f633e656cf8b9b5eecc4 10.244.85.204:6379
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
M: 29da99ee264d9041a5ad08d42ce5be418f1a781b 10.244.58.214:6379
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```
**验证集群**
```
[root@k8s-master redis-sts]# kubectl exec -it redis-cluster-0 -- redis-cli cluster info
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:6
cluster_size:3
cluster_current_epoch:6
cluster_my_epoch:1
cluster_stats_messages_ping_sent:103
cluster_stats_messages_pong_sent:105
cluster_stats_messages_sent:208
cluster_stats_messages_ping_received:100
cluster_stats_messages_pong_received:103
cluster_stats_messages_meet_received:5
cluster_stats_messages_received:208

[root@k8s-master redis-sts]#  for x in $(seq 0 5); do echo "redis-cluster-$x"; kubectl exec redis-cluster-$x -- redis-cli role; echo; done
redis-cluster-0
master
154
10.244.85.203
6379
154

redis-cluster-1
master
154
10.244.58.223
6379
154

redis-cluster-2
master
154
10.244.85.206
6379
154

redis-cluster-3
slave
10.244.58.215
6379
connected
154

redis-cluster-4
slave
10.244.85.204
6379
connected
154

redis-cluster-5
slave
10.244.58.214
6379
connected
154

```
### 五、测试集群
我们想使用集群，然后模拟节点的故障。对于前一项任务，我们将部署一个简单的 Python 应用程序，而对于后者，我们将删除一个节点并观察集群行为。

**部署点击计数器应用**
> 我们将一个简单的应用程序部署到集群中，并在其前面放置一个负载平衡器。此应用程序的目的是在将计数器值作为 HTTP 响应返回之前，增加计数器并将其存储在 Redis 集群中。

```

[root@k8s-master redis-sts]# cat app-deployment-service.yaml 
---
apiVersion: v1
kind: Service
metadata:
  name: hit-counter-lb
spec:
  type: ClusterIP
  ports:
  - port: 80
    protocol: TCP
    targetPort: 5000
  selector:
      app: myapp
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hit-counter-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: calinrus/api-redis-ha:1.0
        ports:
        - containerPort: 5000

[root@k8s-master redis-sts]# kubectl apply -f app-deployment-service.yaml
service/hit-counter-lb created
deployment.apps/hit-counter-app created
```
在此过程中，如果我们继续加载页面，计数器将继续增加，并且在删除Pod之后，我们看到没有数据丢失。

```
[root@k8s-master redis-sts]# curl `kubectl get svc hit-counter-lb -o json|jq -r .spec.clusterIP`
I have been hit 1 times since deployment.

[root@k8s-master redis-sts]# curl `kubectl get svc hit-counter-lb -o json|jq -r .spec.clusterIP`
I have been hit 2 times since deployment.

[root@k8s-master redis-sts]# curl `kubectl get svc hit-counter-lb -o json|jq -r .spec.clusterIP`
I have been hit 3 times since deployment.

[root@k8s-master redis-sts]# curl `kubectl get svc hit-counter-lb -o json|jq -r .spec.clusterIP`
I have been hit 4 times since deployment.

[root@k8s-master redis-sts]# ^C
[root@k8s-master redis-sts]#  kubectl delete pods redis-cluster-0
pod "redis-cluster-0" deleted

[root@k8s-master redis-sts]# kubectl delete pods redis-cluster-1  
pod "redis-cluster-1" deleted

[root@k8s-master redis-sts]# curl `kubectl get svc hit-counter-lb -o json|jq -r .spec.clusterIP`
I have been hit 5 times since deployment.

[root@k8s-master redis-sts]# curl `kubectl get svc hit-counter-lb -o json|jq -r .spec.clusterIP`
I have been hit 6 times since deployment.

```







