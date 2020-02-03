---
title: flink指标对接prometheus和grafana
date: 2020-01-22 13:20:13
tags:
categories:
	- Flink
---
先上一个架构图
![image](https://note.youdao.com/yws/api/personal/file/C534CC37D156467A8C07D0F80A62A431?method=download&shareKey=16abc8aa9e968b4b57234f1197ef10a4)

Flink App ： 通过report 将数据发出去

Pushgateway :  Prometheus 生态中一个重要工具

Prometheus :  一套开源的系统监控报警框架 （Prometheus 入门与实践）

Grafana： 一个跨平台的开源的度量分析和可视化工具，可以通过将采集的数据查询然后可视化的展示，并及时通知（可视化工具Grafana：简介及安装）

Node_exporter : 跟Pushgateway一样是Prometheus 的组件，采集到主机的运行指标如CPU, 内存，磁盘等信息

### prometheus搭建

参考 [prometheus安装教程](https://guosmilesmile.github.io/2020/01/22/prometheus%E5%AE%89%E8%A3%85%E6%95%99%E7%A8%8B/)

### grafana搭建

参考 [grafana安装教程](https://guosmilesmile.github.io/2020/01/22/grafana%E5%AE%89%E8%A3%85%E6%95%99%E7%A8%8B/)

### Pushgateway 安装
pushgateway-deployment.yaml
```
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  namespace: prometheus
  name:  pushgateway-ttt
  labels:
    app:  pushgateway-ttt
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
spec:
  replicas: 1
  revisionHistoryLimit: 0
  selector:
    matchLabels:
      app:  pushgateway-ttt
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: "25%"
      maxUnavailable: "25%"
  template:
    metadata:
      name:  pushgateway-ttt
      labels:
        app:  pushgateway-ttt
    spec:
      containers:
        - name:  pushgateway-ttt
          image: prom/pushgateway:v0.7.0
          imagePullPolicy: IfNotPresent
          livenessProbe:
            initialDelaySeconds: 600
            periodSeconds: 10
            successThreshold: 1
            failureThreshold: 10
            httpGet:
              path: /
              port: 9091
          ports:
            - name: "app-port"
              containerPort: 9091
```

pushgateway-svc.yaml

```
apiVersion: v1
kind: Service
metadata:
  name: pushgateway-ttt
  namespace: prometheus
  labels:
    app: pushgateway-ttt
spec:
  selector:
    app: pushgateway-ttt
  type: NodePort
  ports:
    - name: pushgateway-ttt
      port: 9091
      targetPort: 9091
      nodePort: 30002

```

通过任意机器IP：30002 可以访问ui
![image](https://note.youdao.com/yws/api/personal/file/57787DF2C30D40AABA531F356C2E6121?method=download&shareKey=4db1da3f7b7e2816220e804cd2a2a742)

### 配置Flink report ：


在Flink 配置文件 flink-conf.yml 中添加如下内容：

```
metrics.reporter.promgateway.class: org.apache.flink.metrics.prometheus.PrometheusPushGatewayReporter
metrics.reporter.promgateway.host: pushgateway配置的ip或者域名
metrics.reporter.promgateway.port: 9091
metrics.reporter.promgateway.jobName: myJob
metrics.reporter.promgateway.randomJobNameSuffix: true
metrics.reporter.promgateway.deleteOnShutdown: false

```
jobName配置成对应的名称，不然会被覆盖


### Grafana 中配置Flink监控

由于上面一句配置好Flink report、 pushgateway、prometheus，并且在Grafana中已经添加了prometheus 数据源，所以Grafana中会自动获取到 flink job的metrics 。

 Grafana 首页，点击New dashboard，创建一个新的dashboard
![image](https://note.youdao.com/yws/api/personal/file/01EA8A30DD304E34829B8768A342F983?method=download&shareKey=f5348df553277bff17f8997aa03c7e50)
选中之后，即会出现对应的监控指标

![image](https://note.youdao.com/yws/api/personal/file/6D599FBCACF141698E986BBFBF1DF830?method=download&shareKey=952fc58041313023dcbf2a8999e050b9)
至此，Flink 的metrics 的指标展示在Grafana 中了

flink 指标对应的指标名比较长，可以在Legend 中配置显示内容，在{{key}} 将key换成对应需要展示的字段即可，如： {{job_name}},{{operator_name}}

对应显示如下：
![image](https://note.youdao.com/yws/api/personal/file/97FE97D07161493C8C3ADA0D5403B342?method=download&shareKey=1701b1cfef746a32f73cc97e8f8ddb44)

### 优化

#### 仪表盘的配置
仪表盘的配置可以到官网找别人配好的现成的

[Dashboards](https://grafana.com/grafana/dashboards)
下载json文件然后导入即可。

#### 原生指标太多

当TM太多的时候，仪表盘会太多信息。

例如TaskManagers Garbage Collection Time 的统计，只需要TOP10就好

原先语句

```
delta(flink_taskmanager_Status_JVM_GarbageCollector_G1_Young_Generation_Time[30s])
```

改为

```
topk(10,delta(flink_taskmanager_Status_JVM_GarbageCollector_G1_Young_Generation_Time[30s]))
```

统计每个算子每个TM的吞吐，这种维度太细，其实只要知道每种算子的吞吐即可

```
sum (flink_taskmanager_job_task_operator_numRecordsInPerSecond{job_name=~"$job_name"})  by (operator_name) 
```

关于配置，学习一下PromQL

https://www.jianshu.com/p/f77047fd8dca

### 关于Pushgateway单点瓶颈

在架构部分我们简单介绍过，通过Pushgateway组件，我们可以通过Pushgatway作为中间代理，实现跨网络环境的数据采集，通过一些周期性的任务定时向Pushgateway上报数据，Prometheus依然通过Pull模式定时获取数据。


在官方文档中介绍了三种使用Pushgateway的问题：

* 当采用Pushgateway获取监控数据时，Pushway即会成为单点以及潜在的性能瓶颈
* 丧失了Prometheus的实例健康检查功能
* 除非手动删除Pushgateway的数据，否则Pushgateway会一直保留所有采集到的数据并且提供给Prometheus。


目前我这边想到的方法是pushgateway是部署在kubernetes上，直接通过扩展副本数量来增加实例数，通过kubernetes的service来做负载均衡。

缺点就是pushgateway的ui就没办法指定查看某个节点的UI。

可以通过在master上curl特定节点来获取整个html查看。

```
kubectl get pods -n prometheus -o wide

prometheus-6bb658bb79-wpsd5       1/1     Running   0          24h   10.44.0.3         node1   <none>           <none>
pushgateway-ttt-6b6755dd8-2vtnz   1/1     Running   0          83m   10.44.0.5         node1   <none>           <none>
pushgateway-ttt-6b6755dd8-fd6mq   1/1     Running   0          19h   10.44.0.4         node1   <none>           <none>

```

```
curl -k '10.44.0.4:9091'
```

### 样例

![image](https://note.youdao.com/yws/api/personal/file/60589B0686F6458CB4E9C0CFE77CE388?method=download&shareKey=5c1deae4e8b0de01eeefa278ba569d26)


### Reference

https://www.cnblogs.com/Springmoon-venn/p/11445023.html

https://segmentfault.com/a/1190000017833186?utm_source=tag-newest
