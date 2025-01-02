---
layout: post
title: Containerlab
date: 2024-05-30
tags: Network
---

# Containerlab
## 什么是Containerlab
随着容器化网络操作系统数量的不断增加，在用户定义的多功能实验室拓扑中轻松运行它们的需求也不断增长。

不幸的是，像 docker-compose 这样的容器编排工具不太适合这个目的，因为它们不允许用户在定义拓扑的容器之间轻松创建连接。

Containerlab 提供了一个 CLI，用于编排和管理基于容器的网络实验室。它启动容器，在容器之间构建虚拟线路，以创建用户选择的实验室拓扑并管理实验室生命周期。


## 快速开始
### base env
```
# 我们采用的环境为ubuntu20.04桌面版本：
# 安装必要工具：
$ apt install -y net-tools tcpdump chrony bridge-utils tree wget iftop ethtool curl
# 关闭swap：
$ sed -ri 's/.*swap.*/#&/' /etc/fstab
$ swapoff -a

$ vim /etc/sysctl.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-arptables = 1

# 添加Docker repo：
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
$ sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
$ sudo apt update
$ sudo apt install docker-ce docker-ce-cli containerd.io
$ mkdir -p /etc/docker
$ cat <<EOF > /etc/docker/daemon.json
{
  "registry-mirrors": ["https://docker.cloudimages.asia"]
}
EOF
$ systemctl daemon-reload
$ systemctl restart docker
$ systemctl enable docker


# 添加阿里的kubernetes repo：
$ apt-get update && apt-get install -y apt-transport-https
$ apt-get install -y apt-transport-https
$ apt upgrade -y
$curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add - 
$ tee /etc/apt/sources.list.d/kubernetes.list <<-'EOF'
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
$ apt-get update



# 安装kubelet kubectl kubeadm:
$ apt-get install -y kubelet=1.23.4-00 kubeadm=1.23.4-00 kubectl=1.23.4-00 --allow-unauthenticated

$ systemctl enable kubelet && systemctl restart kubelet

# 配置 kubelet 的 cgroup 驱动  
$  vi /etc/docker/daemon.json 
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "registry-mirrors": ["https://docker.cloudimages.asia"]
}

# [master node]
$ cat > /var/lib/kubelet/config.yaml <<EOF      
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
EOF

$ systemctl restart docker
$ systemctl daemon-reload
$ systemctl enable kubelet && systemctl restart kubelet
$ systemctl daemon-reload
$ systemctl restart docker
$ systemctl enable docker

$ docker info|grep "Cgroup Driver"
 Cgroup Driver: systemd



# 初始化Kubernetes cluster：[master]
$ kubeadm config images pull --image-repository=registry.aliyuncs.com/google_containers
$ kubeadm init --kubernetes-version=v1.23.5 --image-repository registry.aliyuncs.com/google_containers --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12 --skip-phases=addon/kube-proxy --ignore-preflight-errors=Swap

$ kubeadm init --kubernetes-version=v1.23.5 --image-repository registry.aliyuncs.com/google_containers --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12 --ignore-preflight-errors=Swap
# 添加其他的节点：
$ kubeadm config images pull --image-repository=registry.aliyuncs.com/google_containers
$ kubeadm join 192.168.2.61:6443 --token ac4k64.e3i6j13sryj1twzt \
        --discovery-token-ca-cert-hash sha256:7feb5f701bbad147116daddda3e74e720738e61938eedccc7bfaa3d24aed23bf 
        

# 配置env:
$ kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
$ kubectl taint nodes --all node-role.kubernetes.io/master-

$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
### 安装
