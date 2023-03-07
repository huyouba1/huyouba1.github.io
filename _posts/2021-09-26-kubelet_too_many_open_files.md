---
layout: post
title: kubelet too many open files 
date: 2021-09-26 
tags: 常见问题处理
---

### 问题现象
![](/images/posts/error/WechatIMG14791.png)

### 处置思路
```
看到too many open files可能想到fs.file-max参数，其实还受下面参数影响：
fs.inotify.max_queued_events：表示调用inotify_init时分配给inotify instance中可排队的event的数目的最大值，超出这个值的事件被丢弃，但会触发IN_Q_OVERFLOW事件。
fs.inotify.max_user_instances：表示每一个real user ID可创建的inotify instatnces的数量上限，默认128.
fs.inotify.max_user_watches：表示同一用户同时可以添加的watch数目（watch一般是针对目录，决定了同时同一用户可以监控的目录数量）
```
修改参数：
```
因为系统默认的  fs.inotify.max_user_instances=128 太小，在查看日志的pod所在节点重新设置此值：
临时设置
 
sudo sysctl fs.inotify.max_user_instances=8192
 
永久保存
 
echo fs.inotify.max_user_instances=8192| tee -a /etc/sysctl.conf && sudo sysctl -p
```
重启kubelet即可
