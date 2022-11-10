---
layout: post
title: PrometheusRuleFailures
date: 2021-10-10 
tags: kubernetes报错
---
### 问题现象
> 出现此类告警需查看 prometheus 的日志，错误信息 err="query processing would load too many samples into memory in query execution"

### 原因
> 在 prometheus 中执行大量查询导致

### 解决方法：
使用--query.max-samples在普罗米修斯配置文件，增加内存负载的数量。默认值为50000000，增加此值取决于您的机器能力

参考：https://stackoverflow.com/questions/60160973/grafana-prometheus-query-processing-would-load-too-many-samples-into-memory-in/60161953

参考：https://github.com/prometheus-operator/prometheus-operator/blob/master/Documentation/api.md