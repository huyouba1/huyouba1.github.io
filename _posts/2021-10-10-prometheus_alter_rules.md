---
layout: post
title: prometheus 告警含义
date: 2021-10-10 
tags: 常见问题处理
---
告警名称|告警含义
---|---
NodeClockNotSynchronising|是说节点的ntp时钟不同步，是时钟还没有同步报的错。
KubeStatefulSetUpdate |这个告警，是因为sts的升级策略是OnDelete，也就是说如果更新了yaml，需要手动逐个删除Pod以生效，这个历史上是为了避免频繁重启。解决办法就是人工逐个删除下sts的pod然后重建。
AggregatedAPIDown|这个就是警告有些api规则下线了，这个应该只在k8s >=1.18开启;参考https://github.com/prometheus-community/helm-charts/issues/122