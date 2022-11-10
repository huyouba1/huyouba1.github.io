---
layout: post
title: Zabbix监控k8s集群
date: 2021-01-15
tags: kubernetes
---

### 一、监控策略
其实在容器云环境中，zabbix并不是最有效的监控方案，prometheus+Grafana监控比zabbix更加精细化。此处zabbix只是充当一个监控k8s集群健康状态及异常声音告警的一个工具。生产环境中，Prometheus+grafana和zabbix同时使用效果更佳。

因k8s是 Restful APi 风格的接口设置，此处原理也较简单，如下图所示，可直接通过接口管理、查看数据。后面的python脚本将通过此原理来实现监控。Centos7系统自带的python v2版本即可。

![](/images/posts/06_k8s/19/1.png)

### 二、集群环境
```
[root@k8s-master ~]# kubectl get nodes -o wide
NAME         STATUS   ROLES    AGE   VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION          CONTAINER-RUNTIME
k8s-master   Ready    master   52d   v1.15.1   192.168.160.70   <none>        CentOS Linux 7 (Core)   3.10.0-957.el7.x86_64   docker://19.3.13
k8s-node01   Ready    <none>   52d   v1.15.1   192.168.160.66   <none>        CentOS Linux 7 (Core)   3.10.0-957.el7.x86_64   docker://19.3.13
k8s-node02   Ready    <none>   52d   v1.15.1   192.168.160.67   <none>        CentOS Linux 7 (Core)   3.10.0-957.el7.x86_64   docker://19.3.13
```

zabbix可以通过k8s来部署，也可集群外二进制部署，此处为二进制部署。安装过程不再概述。

### 三、k8s做授权


```
[root@k8s-master ~]# cat zabbix-user.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: zabbix-user
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: zabbix-user
rules:
- apiGroups:
  - ""
  resources:
  - pods
  - services
  - nodes
  - namespaces
  - apiservices
  - componentstatuses
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - "apiregistration.k8s.io"
  resources:
  - apiservices
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - "apps"
  resources:
  - deployments
  verbs:
  - get
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: zabbix-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: zabbix-user
subjects:
- kind: ServiceAccount
  name: zabbix-user
  namespace: default

[root@k8s-master ~]# kubectl apply -f zabbix-user.yaml
```
> 授权后，zabbix将可正常通过脚本抓取数据，否则返回503报错



### 四、在master01上安装Zabbix-agent
过程不赘述，安装完成后修改/etc/zabbix/conf/Zabbix-agentd.conf文件中的server IP地址，并到web页面添加这台机器

![](/images/posts/06_k8s/19/2.png)

### 五、导入并应用模板
[脚本及配置文件下载位置](http://huyouba1.github.io:81/zabbix-kubernetes-monitoring-master.zip)  

```
[root@k8s-master ~]# mkdir -p /etc/zabbix/scripts   //新增脚本目录

[root@k8s-master scripts]# ls
k8s-stats.py      //此为py脚本

[root@k8s-master zabbix]# cd /etc/zabbix/zabbix_agentd.d/

[root@k8s-master zabbix_agentd.d]# ls
k8s.conf  userparameter_mysql.conf       //将k8s.conf上传到此目录
```

> 修改python脚本中以下两个位置：

- api_server = 'https://192.168.160.70:6443' # 这里改成自己的apiServer地址
- token 地址：“token获取方法如下”

```
[root@k8s-master zabbix_agentd.d]# kubectl get secret
NAME                       TYPE                                  DATA   AGE
admin-token-z78rj          kubernetes.io/service-account-token   3      28d
cm-adapter-serving-certs   Opaque                                2      12d
def-ns-admin-token-x472f   kubernetes.io/service-account-token   3      26d
default-token-xg4xv        kubernetes.io/service-account-token   3      52d
mysql-root-password        Opaque                                1      31d
tomcat-ingress-secret      kubernetes.io/tls                     2      34d
zabbix-user-token-jwkz2    kubernetes.io/service-account-token   3      3d2h

[root@k8s-master zabbix_agentd.d]# kubectl get secret zabbix-user-token-jwkz2 -o jsonpath={.data.token} |base64  -d

eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6InphYmJpeC11c2VyLXRva2VuLWp3a3oyIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6InphYmJpeC11c2VyIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiMGI4ZjMxZmUtYTJiOC00Zjc5LWIyZTgtMDIzNmJiN2ZhZTgxIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OmRlZmF1bHQ6emFiYml4LXVzZXIifQ.xnKNNhhbvH0cu1tB-5rFDRYPs2TENQu4YsmfncZ2sSIoeEmR0n4fcYa3yFjuY_rg0sdkoAhw5PHl9x7GlV0nYEemJzugVlr9TSG-yG-dv1nDB4V1ej0rheRIkwc9xmQWvg7YhoCBbKIT2urIZAveyFRr6r7zNnvxLoQe7jw_NrCQ9m7QO-NH4xwPoHsZgA9a77EEO7Uz-SU7gHTFuTzzGqjubL4uUUCaEsdtpSpH7jFogtfNpW5lLcMdaRtgCFmbU8VgVox4IuQrL_Af5vHMfIRaCuR4-tKp1N-W9KVN8ngqNlf164xuxigMKf5UiYRDoskS8WuTDnbrWnNJNCgvfQ
```
> 把编码后的字符作为k8s-stats.py中的token地址
> 还有一点要注意的是把Zabbix-agent安装的目录设为Zabbix用户所有，过程不赘述

### 六、导入模板并将主机链接到模板上，等待发现即可

![](/images/posts/06_k8s/19/3.png)

![](/images/posts/06_k8s/19/4.png)


![](/images/posts/06_k8s/19/5.png)

![](/images/posts/06_k8s/19/6.png)
