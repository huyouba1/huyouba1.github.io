---
layout: post
title: prometheus alert rules 导入失败问题
date: 2021-10-10 
tags: 常见问题处理
---

### 问题现象：

> 部署监控平台需要导入 alert-rule.yaml 中的告警规则，通过 kubectl apply -f  alert-rule.yaml 可以正常导入进去，但是在prometheus alert界面看不到对应规则

### 原因：

> prometheus 环境部署是通过 prometheus-operator 进行的部署，operator 会通过 rules 的标签进行匹配是否将规则注入到 prometheus 里面。alert-rules.yaml 的标签不符合 operator 要匹配的标签，所以注入不进去

```
$ kubectl get prometheus prometheus-operator-operat-prometheus -oyaml   # 具体名字根据现场环境填写，格式 kubectl get prometheus <PROMETHEUS_NAME> -oyaml
 
···
spec:
  ruleNamespaceSelector: {}
  ruleSelector:
    matchLabels:      ## rules 规则需要匹配以下两个标签
      app: prometheus-operator
      release: prometheus-operator-operator
···
```
```
alert-rule.yaml
---
 
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  labels:   #与此标签不匹配，所以注入不进去
    app: prometheus-operator
    release: prometheus-operator 
  name: alert-node-rules
  namespace: default
spec:
  groups:
  - name: node-rules
    rules:
    - alert: 主机内存使用率超过80%告警
      expr: sum(100- ((node_memory_MemAvailable_bytes * 100 ) / node_memory_MemTotal_bytes )) by (instance,job) > 80
      for: 5m
      labels:
        severity: "warning"
      annotations:
        summary: "告警主机：{{ $labels.instance }}"
        address: "告警地址：{{ $labels.instance }}"
        description: 主机 {{ $labels.instance }} 内存使用率超过80%，当前使用率为 {{ $value | printf "%.2f" }}%。
 
---
 
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  labels:     #与此标签不匹配，所以注入不进去
    app: prometheus-operator
    release: prometheus-operator
  name: alert-k8s-rules
  namespace: default
spec:
  groups:
  - name: k8s-rules
    rules:
    - alert: KubeApiserver存活监控告警
      expr: absent(up{job="apiserver"} == 1)
      for: 1m
      labels:
        severity: "critical"
      annotations:
        description: KubeApiserver实例 {{ $labels.instance }} 从prometheus监控目标中失联
```
### 解决：

修改 alert-rule.yaml 的标签，使之匹配