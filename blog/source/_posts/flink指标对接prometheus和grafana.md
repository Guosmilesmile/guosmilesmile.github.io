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

### grafana

```yaml
{
  "annotations": {
    "list": [
      {
        "builtIn": 1,
        "datasource": "-- Grafana --",
        "enable": true,
        "hide": true,
        "iconColor": "rgba(0, 211, 255, 1)",
        "name": "Annotations & Alerts",
        "type": "dashboard"
      }
    ]
  },
  "description": "flink with prometheus",
  "editable": true,
  "gnetId": 11049,
  "graphTooltip": 0,
  "id": 2,
  "iteration": 1584192110931,
  "links": [],
  "panels": [
    {
      "collapsed": false,
      "datasource": null,
      "gridPos": {
        "h": 1,
        "w": 24,
        "x": 0,
        "y": 0
      },
      "id": 16,
      "panels": [],
      "title": "Slots",
      "type": "row"
    },
    {
      "cacheTimeout": null,
      "colorBackground": false,
      "colorValue": true,
      "colors": [
        "rgba(245, 54, 54, 0.9)",
        "rgba(255, 255, 255, 0.89)",
        "rgba(50, 172, 45, 0.97)"
      ],
      "datasource": "$datasource",
      "format": "none",
      "gauge": {
        "maxValue": null,
        "minValue": 0,
        "show": false,
        "thresholdLabels": false,
        "thresholdMarkers": true
      },
      "gridPos": {
        "h": 6,
        "w": 4,
        "x": 0,
        "y": 1
      },
      "hideTimeOverride": false,
      "id": 7,
      "interval": null,
      "links": [],
      "mappingType": 1,
      "mappingTypes": [
        {
          "name": "value to text",
          "value": 1
        },
        {
          "name": "range to text",
          "value": 2
        }
      ],
      "maxDataPoints": 100,
      "nullPointMode": "connected",
      "nullText": null,
      "options": {},
      "pluginVersion": "6.5.3",
      "postfix": "",
      "postfixFontSize": "50%",
      "prefix": "",
      "prefixFontSize": "50%",
      "rangeMaps": [
        {
          "from": "null",
          "text": "N/A",
          "to": "null"
        }
      ],
      "sparkline": {
        "fillColor": "rgba(251, 129, 76, 0.18)",
        "full": false,
        "lineColor": "rgb(193, 31, 31)",
        "show": true
      },
      "tableColumn": "",
      "targets": [
        {
          "expr": "sum(flink_jobmanager_numRegisteredTaskManagers)",
          "format": "time_series",
          "hide": false,
          "intervalFactor": 2,
          "legendFormat": "{{instance}}",
          "metric": "flink_jobmanager_numRegisteredTaskManagers",
          "refId": "A",
          "step": 20
        }
      ],
      "thresholds": "",
      "timeFrom": null,
      "timeShift": null,
      "title": "# of TaskManagers",
      "type": "singlestat",
      "valueFontSize": "80%",
      "valueMaps": [
        {
          "op": "=",
          "text": "N/A",
          "value": "null"
        }
      ],
      "valueName": "current"
    },
    {
      "cacheTimeout": null,
      "colorBackground": false,
      "colorValue": true,
      "colors": [
        "rgba(245, 54, 54, 0.9)",
        "rgba(255, 255, 255, 0.89)",
        "rgba(50, 172, 45, 0.97)"
      ],
      "datasource": "$datasource",
      "format": "none",
      "gauge": {
        "maxValue": null,
        "minValue": 0,
        "show": false,
        "thresholdLabels": false,
        "thresholdMarkers": true
      },
      "gridPos": {
        "h": 6,
        "w": 4,
        "x": 4,
        "y": 1
      },
      "hideTimeOverride": false,
      "id": 3,
      "interval": null,
      "links": [],
      "mappingType": 1,
      "mappingTypes": [
        {
          "name": "value to text",
          "value": 1
        },
        {
          "name": "range to text",
          "value": 2
        }
      ],
      "maxDataPoints": 100,
      "nullPointMode": "connected",
      "nullText": null,
      "options": {},
      "postfix": "",
      "postfixFontSize": "50%",
      "prefix": "",
      "prefixFontSize": "50%",
      "rangeMaps": [
        {
          "from": "null",
          "text": "N/A",
          "to": "null"
        }
      ],
      "sparkline": {
        "fillColor": "rgba(31, 118, 189, 0.18)",
        "full": false,
        "lineColor": "rgb(31, 120, 193)",
        "show": true
      },
      "tableColumn": "",
      "targets": [
        {
          "expr": "sum(flink_jobmanager_taskSlotsAvailable)",
          "format": "time_series",
          "hide": false,
          "intervalFactor": 2,
          "legendFormat": "",
          "metric": "flink_jobmanager_taskSlotsAvailable",
          "refId": "A",
          "step": 20
        }
      ],
      "thresholds": "",
      "title": "Taskslots available",
      "type": "singlestat",
      "valueFontSize": "80%",
      "valueMaps": [
        {
          "op": "=",
          "text": "N/A",
          "value": "null"
        }
      ],
      "valueName": "current"
    },
    {
      "cacheTimeout": null,
      "colorBackground": false,
      "colorValue": true,
      "colors": [
        "rgba(245, 54, 54, 0.9)",
        "rgba(255, 255, 255, 0.89)",
        "rgba(50, 172, 45, 0.97)"
      ],
      "datasource": "$datasource",
      "format": "none",
      "gauge": {
        "maxValue": null,
        "minValue": 0,
        "show": false,
        "thresholdLabels": false,
        "thresholdMarkers": true
      },
      "gridPos": {
        "h": 6,
        "w": 4,
        "x": 8,
        "y": 1
      },
      "hideTimeOverride": false,
      "id": 4,
      "interval": null,
      "links": [],
      "mappingType": 1,
      "mappingTypes": [
        {
          "name": "value to text",
          "value": 1
        },
        {
          "name": "range to text",
          "value": 2
        }
      ],
      "maxDataPoints": 100,
      "nullPointMode": "connected",
      "nullText": null,
      "options": {},
      "pluginVersion": "6.5.3",
      "postfix": "",
      "postfixFontSize": "50%",
      "prefix": "",
      "prefixFontSize": "50%",
      "rangeMaps": [
        {
          "from": "null",
          "text": "N/A",
          "to": "null"
        }
      ],
      "sparkline": {
        "fillColor": "rgba(31, 118, 189, 0.18)",
        "full": false,
        "lineColor": "rgb(31, 120, 193)",
        "show": true
      },
      "tableColumn": "",
      "targets": [
        {
          "expr": "sum(flink_jobmanager_taskSlotsTotal)",
          "format": "time_series",
          "hide": false,
          "intervalFactor": 2,
          "legendFormat": "",
          "metric": "flink_jobmanager_taskSlotsTotal",
          "refId": "A",
          "step": 20
        }
      ],
      "thresholds": "",
      "timeFrom": null,
      "timeShift": null,
      "title": "Taskslots total",
      "type": "singlestat",
      "valueFontSize": "80%",
      "valueMaps": [
        {
          "op": "=",
          "text": "N/A",
          "value": "null"
        }
      ],
      "valueName": "current"
    },
    {
      "cacheTimeout": null,
      "colorBackground": false,
      "colorValue": true,
      "colors": [
        "rgba(245, 54, 54, 0.9)",
        "rgba(255, 255, 255, 0.89)",
        "rgba(50, 172, 45, 0.97)"
      ],
      "datasource": "$datasource",
      "format": "none",
      "gauge": {
        "maxValue": null,
        "minValue": 0,
        "show": false,
        "thresholdLabels": false,
        "thresholdMarkers": true
      },
      "gridPos": {
        "h": 6,
        "w": 3,
        "x": 12,
        "y": 1
      },
      "hideTimeOverride": false,
      "id": 8,
      "interval": null,
      "links": [],
      "mappingType": 1,
      "mappingTypes": [
        {
          "name": "value to text",
          "value": 1
        },
        {
          "name": "range to text",
          "value": 2
        }
      ],
      "maxDataPoints": 100,
      "nullPointMode": "connected",
      "nullText": null,
      "options": {},
      "postfix": "",
      "postfixFontSize": "50%",
      "prefix": "",
      "prefixFontSize": "50%",
      "rangeMaps": [
        {
          "from": "null",
          "text": "N/A",
          "to": "null"
        }
      ],
      "sparkline": {
        "fillColor": "rgba(251, 129, 76, 0.18)",
        "full": false,
        "lineColor": "rgb(193, 31, 31)",
        "show": true
      },
      "tableColumn": "",
      "targets": [
        {
          "expr": "sum(flink_jobmanager_numRunningJobs)",
          "format": "time_series",
          "hide": false,
          "intervalFactor": 2,
          "legendFormat": "{{instance}} jobmanager运行的job数量",
          "metric": "flink_jobmanager_numRunningJobs",
          "refId": "A",
          "step": 20
        }
      ],
      "thresholds": "",
      "title": "# of Running Jobs",
      "type": "singlestat",
      "valueFontSize": "80%",
      "valueMaps": [
        {
          "op": "=",
          "text": "N/A",
          "value": "null"
        }
      ],
      "valueName": "current"
    },
    {
      "aliasColors": {},
      "bars": false,
      "cacheTimeout": null,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 6,
        "w": 9,
        "x": 15,
        "y": 1
      },
      "hiddenSeries": false,
      "id": 65,
      "legend": {
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": true,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "null",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pluginVersion": "6.5.3",
      "pointradius": 2,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "flink_jobmanager_job_fullRestarts{job_name=~\"$job_name\"}",
          "intervalFactor": 2,
          "legendFormat": "{{job_name}}",
          "refId": "A"
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "job restart count TOP $topN",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "collapsed": false,
      "datasource": null,
      "gridPos": {
        "h": 1,
        "w": 24,
        "x": 0,
        "y": 7
      },
      "id": 58,
      "panels": [],
      "title": "BackPressure",
      "type": "row"
    },
    {
      "aliasColors": {},
      "bars": false,
      "cacheTimeout": null,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "description": "numRecordsOutPerSecond",
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 9,
        "w": 24,
        "x": 0,
        "y": 8
      },
      "hiddenSeries": false,
      "id": 59,
      "legend": {
        "alignAsTable": true,
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "rightSide": true,
        "show": true,
        "sideWidth": 250,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "connected",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pluginVersion": "6.2.4",
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "sum (flink_taskmanager_job_task_operator_numRecordsOutPerSecond{job_name=~\"$job_name\"})  by (operator_name) ",
          "format": "time_series",
          "hide": false,
          "intervalFactor": 2,
          "legendFormat": "{{operator_name}}",
          "metric": "flink_jobmanager_Status_JVM_Memory_Direct_MemoryUsed",
          "refId": "A",
          "step": 20
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "numRecordsOutPerSecond",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "cumulative"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "format": "decbytes",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "cacheTimeout": null,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "description": "umRecordsInPerSecond",
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 7,
        "w": 24,
        "x": 0,
        "y": 17
      },
      "hiddenSeries": false,
      "id": 61,
      "legend": {
        "alignAsTable": true,
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "rightSide": true,
        "show": true,
        "sideWidth": 250,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "connected",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pluginVersion": "6.2.4",
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "sum (flink_taskmanager_job_task_operator_numRecordsInPerSecond{job_name=~\"$job_name\"})  by (operator_name) ",
          "format": "time_series",
          "hide": false,
          "instant": false,
          "intervalFactor": 2,
          "legendFormat": "{{operator_name}}",
          "metric": "flink_jobmanager_Status_JVM_Memory_Direct_MemoryUsed",
          "refId": "B",
          "step": 20
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "numRecordsInPerSecond",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "cumulative"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "format": "bytes",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 7,
        "w": 24,
        "x": 0,
        "y": 24
      },
      "hiddenSeries": false,
      "id": 69,
      "legend": {
        "alignAsTable": true,
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "rightSide": true,
        "show": true,
        "sideWidth": 250,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "nullPointMode": "null",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 2,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "time() -  ( max(flink_taskmanager_job_task_currentInputWatermark{job_name=~\"$job_name\"}) by (task_name)/1000)",
          "legendFormat": "{{task_name}}",
          "refId": "A"
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "waterMark",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "cacheTimeout": null,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "description": "inPoolUsag",
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 6,
        "w": 24,
        "x": 0,
        "y": 31
      },
      "hiddenSeries": false,
      "id": 56,
      "legend": {
        "alignAsTable": true,
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "rightSide": true,
        "show": true,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "connected",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pluginVersion": "6.2.4",
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "flink_taskmanager_job_task_Shuffle_Netty_Output_Buffers_outPoolUsage{job_name=~\"$job_name\"} *100 >50",
          "format": "time_series",
          "hide": false,
          "intervalFactor": 5,
          "legendFormat": "{{subtask_index}}_{{task_name}}_{{host}}",
          "metric": "flink_jobmanager_Status_JVM_Memory_Direct_MemoryUsed",
          "refId": "A",
          "step": 20
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "inPoolUsag",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "cumulative"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "percent",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "format": "decbytes",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "cacheTimeout": null,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "description": "outPoolUsage",
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 6,
        "w": 24,
        "x": 0,
        "y": 37
      },
      "hiddenSeries": false,
      "id": 62,
      "legend": {
        "alignAsTable": true,
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "rightSide": true,
        "show": true,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "connected",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pluginVersion": "6.2.4",
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "flink_taskmanager_job_task_Shuffle_Netty_Input_Buffers_inPoolUsage{job_name=~\"$job_name\"} *100  > 50",
          "format": "time_series",
          "hide": false,
          "intervalFactor": 4,
          "legendFormat": "{{subtask_index}}_{{task_name}}_{{host}}",
          "metric": "flink_jobmanager_Status_JVM_Memory_Direct_MemoryUsed",
          "refId": "A",
          "step": 20
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "outPoolUsage",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "cumulative"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "percent",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "format": "decbytes",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "cacheTimeout": null,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "description": "inputQueueLength",
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 7,
        "w": 11,
        "x": 0,
        "y": 43
      },
      "hiddenSeries": false,
      "id": 63,
      "legend": {
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": false,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "connected",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pluginVersion": "6.2.4",
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "sum(flink_taskmanager_job_task_Shuffle_Netty_Input_Buffers_inputQueueLength{job_name=~\"$job_name\"}) by (host)",
          "format": "time_series",
          "hide": false,
          "intervalFactor": 2,
          "legendFormat": "{{host}}_{{subtask_index}}",
          "metric": "flink_jobmanager_Status_JVM_Memory_Direct_MemoryUsed",
          "refId": "A",
          "step": 20
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "inputQueueLength ",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "cumulative"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "format": "decbytes",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "cacheTimeout": null,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "description": "outputQueueLength",
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 7,
        "w": 13,
        "x": 11,
        "y": 43
      },
      "hiddenSeries": false,
      "id": 60,
      "legend": {
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": false,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "connected",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pluginVersion": "6.2.4",
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "sum(flink_taskmanager_job_task_Shuffle_Netty_Input_Buffers_inputQueueLength{job_name=~\"$job_name\"}) by (host)",
          "format": "time_series",
          "hide": false,
          "intervalFactor": 4,
          "legendFormat": "{{host}}",
          "metric": "flink_jobmanager_Status_JVM_Memory_Direct_MemoryUsed",
          "refId": "A",
          "step": 20
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "outputQueueLength",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "cumulative"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "format": "decbytes",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 8,
        "w": 24,
        "x": 0,
        "y": 50
      },
      "hiddenSeries": false,
      "id": 71,
      "legend": {
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": true,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "nullPointMode": "null",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 2,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "max (flink_taskmanager_job_task_operator_fetch_latency_avg{job_name=~\"$job_name\"}) by (task_name)",
          "legendFormat": "fetch_{{task_name}}",
          "refId": "A"
        },
        {
          "expr": "max (flink_taskmanager_job_task_operator_commit_latency_avg{job_name=~\"$job_name\"}) by (task_name)",
          "legendFormat": "commit_{{task_name}}",
          "refId": "B"
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "flink_taskmanager_job_task_operator_fetch_latency_avg",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "collapsed": false,
      "datasource": null,
      "gridPos": {
        "h": 1,
        "w": 24,
        "x": 0,
        "y": 58
      },
      "id": 18,
      "panels": [],
      "repeat": null,
      "title": "JVM",
      "type": "row"
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "fill": 0,
      "fillGradient": 0,
      "gridPos": {
        "h": 6,
        "w": 24,
        "x": 0,
        "y": 59
      },
      "hiddenSeries": false,
      "id": 10,
      "interval": "",
      "legend": {
        "alignAsTable": true,
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "rightSide": true,
        "show": true,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "connected",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 0.5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "delta(flink_taskmanager_Status_JVM_GarbageCollector_G1_Old_Generation_Count[30s]) > 50",
          "format": "time_series",
          "instant": false,
          "interval": "",
          "intervalFactor": 1,
          "legendFormat": "{{host}}_{{tm_id}}_老年代gc count",
          "metric": "flink_jobmanager_Status_JVM_GarbageCollector_Copy_Time",
          "refId": "A",
          "step": 2
        },
        {
          "expr": "delta(flink_taskmanager_Status_JVM_GarbageCollector_G1_Young_Generation_Count[30s]) > 50",
          "format": "time_series",
          "instant": false,
          "interval": "",
          "intervalFactor": 1,
          "legendFormat": "{{host}}_{{tm_id}}_年轻代gc count",
          "refId": "B"
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "TaskManager Garbage Collection count ",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "decimals": null,
          "format": "short",
          "label": "",
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "fill": 0,
      "fillGradient": 0,
      "gridPos": {
        "h": 8,
        "w": 24,
        "x": 0,
        "y": 65
      },
      "hiddenSeries": false,
      "id": 9,
      "legend": {
        "alignAsTable": true,
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "rightSide": true,
        "show": true,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "connected",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "delta(flink_taskmanager_Status_JVM_GarbageCollector_G1_Young_Generation_Time[30s]) > 1000",
          "format": "time_series",
          "interval": "",
          "intervalFactor": 1,
          "legendFormat": "{{host}}_{{tm_id}}_年轻代gc时间",
          "metric": "flink_taskmanager_Status_JVM_GarbageCollector_G1_Young_Generation_Count",
          "refId": "B",
          "step": 2
        },
        {
          "expr": "delta(flink_taskmanager_Status_JVM_GarbageCollector_G1_Old_Generation_Time[30s]) > 1000",
          "format": "time_series",
          "hide": false,
          "instant": false,
          "interval": "",
          "intervalFactor": 1,
          "legendFormat": "{{host}}_{{tm_id}}_老年代gc时间",
          "refId": "A"
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "TaskManagers Garbage Collection Time ",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "ms",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 7,
        "w": 24,
        "x": 0,
        "y": 73
      },
      "hiddenSeries": false,
      "id": 26,
      "legend": {
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": false,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "null",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 2,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "sum(delta(flink_taskmanager_job_task_operator_numLateRecordsDropped{job_name=~\"$job_name\"}[30s]))",
          "format": "time_series",
          "intervalFactor": 3,
          "legendFormat": "{{host}}_{{job_name}}_{{tm_id}}",
          "refId": "A"
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "records has dropped due to arriving late.",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "collapsed": false,
      "datasource": null,
      "gridPos": {
        "h": 1,
        "w": 24,
        "x": 0,
        "y": 80
      },
      "id": 24,
      "panels": [],
      "title": "IO",
      "type": "row"
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "description": "TaskManger NonHeap Memory Available",
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 6,
        "w": 12,
        "x": 0,
        "y": 81
      },
      "hiddenSeries": false,
      "id": 41,
      "legend": {
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": false,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "connected",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "flink_taskmanager_Status_JVM_Memory_NonHeap_Committed-flink_taskmanager_Status_JVM_Memory_NonHeap_Used  < 2000000",
          "format": "time_series",
          "intervalFactor": 2,
          "legendFormat": "{{host}}:{{tm_id}}",
          "metric": "flink_jobmanager_Status_JVM_Memory_Direct_MemoryUsed",
          "refId": "A",
          "step": 20
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "TaskManger NonHeap Memory Available < 2m",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "cumulative"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "bytes",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "format": "decbytes",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "description": "TaskManager Memory Available",
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 6,
        "w": 12,
        "x": 12,
        "y": 81
      },
      "hiddenSeries": false,
      "id": 6,
      "legend": {
        "avg": false,
        "current": false,
        "hideEmpty": false,
        "hideZero": false,
        "max": false,
        "min": false,
        "rightSide": false,
        "show": false,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "connected",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 0.5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "flink_taskmanager_Status_JVM_Memory_Heap_Max-flink_taskmanager_Status_JVM_Memory_Heap_Used < 1000000000",
          "format": "time_series",
          "hide": false,
          "intervalFactor": 2,
          "legendFormat": "{{host}}:{{tm_id}}",
          "metric": "flink_taskmanager_Status_JVM_Memory_Direct_MemoryUsed",
          "refId": "A",
          "step": 4
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "TaskManager Memory Available < 1G",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "bytes",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "fill": 0,
      "fillGradient": 0,
      "gridPos": {
        "h": 6,
        "w": 12,
        "x": 0,
        "y": 87
      },
      "hiddenSeries": false,
      "id": 5,
      "legend": {
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": true,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "connected",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "flink_jobmanager_Status_JVM_Memory_Heap_Max-flink_jobmanager_Status_JVM_Memory_Heap_Used",
          "format": "time_series",
          "interval": "",
          "intervalFactor": 1,
          "legendFormat": "{{host}}",
          "metric": "flink_jobmanager_Status_JVM_Memory_Direct_MemoryUsed",
          "refId": "A",
          "step": 20
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "JobManager Memory Available",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "bytes",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "format": "decbytes",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "description": "JobManager NonHeap Memory Available",
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 6,
        "w": 12,
        "x": 12,
        "y": 87
      },
      "hiddenSeries": false,
      "id": 40,
      "legend": {
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": true,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "connected",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "flink_jobmanager_Status_JVM_Memory_NonHeap_Committed-flink_jobmanager_Status_JVM_Memory_NonHeap_Used",
          "format": "time_series",
          "intervalFactor": 2,
          "legendFormat": "{{host}}",
          "metric": "flink_jobmanager_Status_JVM_Memory_Direct_MemoryUsed",
          "refId": "A",
          "step": 20
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "JobManager NonHeap Memory Available",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "bytes",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "collapsed": false,
      "datasource": null,
      "gridPos": {
        "h": 1,
        "w": 24,
        "x": 0,
        "y": 93
      },
      "id": 67,
      "panels": [],
      "repeat": null,
      "title": "Live",
      "type": "row"
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 8,
        "w": 8,
        "x": 0,
        "y": 94
      },
      "hiddenSeries": false,
      "id": 42,
      "legend": {
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": false,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "null",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "flink_jobmanager_Status_JVM_Threads_Count",
          "format": "time_series",
          "intervalFactor": 2,
          "legendFormat": "{{host}} jobManager live Thread counts",
          "metric": "flink_taskmanager_Status_JVM_GarbageCollector_G1_Young_Generation_Count",
          "refId": "B",
          "step": 2
        },
        {
          "expr": "rate(flink_taskmanager_Status_JVM_Threads_Count[5m]) > 1",
          "format": "time_series",
          "intervalFactor": 2,
          "legendFormat": "{{host}} taskManger live Thread counts",
          "metric": "flink_taskmanager_Status_JVM_GarbageCollector_G1_Young_Generation_Count",
          "refId": "A",
          "step": 2
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "live Thread count",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "locale",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "collapsed": false,
      "datasource": null,
      "gridPos": {
        "h": 1,
        "w": 24,
        "x": 0,
        "y": 102
      },
      "id": 37,
      "panels": [],
      "title": "CheckPoint",
      "type": "row"
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "description": "lastCheckpointDuration",
      "fill": 0,
      "fillGradient": 0,
      "gridPos": {
        "h": 6,
        "w": 8,
        "x": 0,
        "y": 103
      },
      "hiddenSeries": false,
      "id": 38,
      "legend": {
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": false,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "null",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 0.5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "flink_jobmanager_job_lastCheckpointDuration{job_name=~\"$job_name\"}",
          "format": "time_series",
          "instant": false,
          "interval": "",
          "intervalFactor": 2,
          "legendFormat": "{{host}}_{{job_name}}",
          "refId": "A"
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "lastCheckpointDuration",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "cumulative"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "ms",
          "label": "",
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "format": "ms",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "Prometheus",
      "description": "Last CheckPoint Size",
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 6,
        "w": 8,
        "x": 8,
        "y": 103
      },
      "hiddenSeries": false,
      "id": 39,
      "legend": {
        "alignAsTable": false,
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": false,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "connected",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 0.5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "rate(flink_jobmanager_job_lastCheckpointSize{job_name=~\"$job_name\"}[2m])",
          "format": "time_series",
          "instant": false,
          "interval": "",
          "intervalFactor": 2,
          "legendFormat": "{{host}}_{{job_name}}",
          "refId": "A"
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "Last CheckPoint Size",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "decimals": null,
          "format": "bytes",
          "label": "",
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "cacheTimeout": null,
      "dashLength": 10,
      "dashes": false,
      "datasource": "Prometheus",
      "description": "numberOfFailedCheckpoints",
      "fill": 0,
      "fillGradient": 0,
      "gridPos": {
        "h": 6,
        "w": 8,
        "x": 16,
        "y": 103
      },
      "hiddenSeries": false,
      "id": 35,
      "legend": {
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": false,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "connected",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pluginVersion": "6.1.3",
      "pointradius": 2,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "delta(flink_jobmanager_job_numberOfFailedCheckpoints{job_name=~\"$job_name\"}[2m])",
          "format": "time_series",
          "interval": "",
          "intervalFactor": 4,
          "legendFormat": "{{host}}_{{job_name}}_FailedCheckpoints",
          "refId": "A"
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "numberOfFailedCheckpoints",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "decimals": null,
          "format": "short",
          "label": "",
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "collapsed": false,
      "datasource": null,
      "gridPos": {
        "h": 1,
        "w": 24,
        "x": 0,
        "y": 109
      },
      "id": 20,
      "panels": [],
      "title": "connector",
      "type": "row"
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "Prometheus",
      "description": "offset commit failures to Kafka",
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 6,
        "w": 8,
        "x": 0,
        "y": 110
      },
      "hiddenSeries": false,
      "id": 22,
      "legend": {
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": false,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "connected",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 2,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "rate(flink_taskmanager_job_task_operator_commitsFailed{job_name=~\"$job_name\"}[2m])",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "{{host}}_{{job_name}}_{{tm_id}}",
          "refId": "A"
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "offset commit failures to Kafka",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "cacheTimeout": null,
      "dashLength": 10,
      "dashes": false,
      "datasource": "Prometheus",
      "description": "job-downtime",
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 6,
        "w": 8,
        "x": 16,
        "y": 110
      },
      "hiddenSeries": false,
      "id": 43,
      "legend": {
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": false,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "connected",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pluginVersion": "6.1.3",
      "pointradius": 1,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "flink_jobmanager_job_downtime{job_name=~\"$job_name\"}",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "{{host}}_{{job_name}}_downtime",
          "refId": "A"
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "job-downtime",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "ms",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": false
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "collapsed": false,
      "datasource": null,
      "gridPos": {
        "h": 1,
        "w": 24,
        "x": 0,
        "y": 116
      },
      "id": 52,
      "panels": [],
      "title": "rocksdb",
      "type": "row"
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "description": "delayed write rate",
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 6,
        "w": 8,
        "x": 0,
        "y": 117
      },
      "hiddenSeries": false,
      "id": 50,
      "legend": {
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": false,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "connected",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "flink_taskmanager_job_task_operator_counter_rocksdb_actual_delayed_write_rate{job_name=~\"$job_name\"}",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "{{host}}_{{job_name}}_{{tm_id}}",
          "metric": "flink_jobmanager_Status_JVM_Memory_Direct_MemoryUsed",
          "refId": "A",
          "step": 20
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "delayed write rate",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "cumulative"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "bytes",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "format": "decbytes",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "description": "entries in the unflushed immutable memtables.",
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 6,
        "w": 8,
        "x": 8,
        "y": 117
      },
      "hiddenSeries": false,
      "id": 53,
      "legend": {
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": false,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "connected",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "flink_taskmanager_job_task_operator_counter_rocksdb_num_deletes_imm_mem_tables{job_name=~\"$job_name\"}",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "{{host}}_{{job_name}}_{{tm_id}} delete entries",
          "metric": "flink_jobmanager_Status_JVM_Memory_Direct_MemoryUsed",
          "refId": "A",
          "step": 20
        },
        {
          "expr": "flink_taskmanager_job_task_operator_counter_rocksdb_num_entries_imm_mem_tables{job_name=~\"$job_name\"}",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "{{host}}_{{job_name}}_{{tm_id}} total entries",
          "metric": "flink_jobmanager_Status_JVM_Memory_Direct_MemoryUsed",
          "refId": "B",
          "step": 20
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": " entries in the unflushed immutable memtables",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "cumulative"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "format": "decbytes",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "description": "background errors in RocksDB",
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 6,
        "w": 8,
        "x": 16,
        "y": 117
      },
      "hiddenSeries": false,
      "id": 54,
      "legend": {
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": false,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "connected",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "flink_taskmanager_job_task_operator_counter_rocksdb_background_errors{job_name=~\"$job_name\"}",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "{{host}}_{{job_name}}_{{tm_id}}",
          "metric": "flink_jobmanager_Status_JVM_Memory_Direct_MemoryUsed",
          "refId": "A",
          "step": 20
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "background errors in RocksDB",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "cumulative"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "format": "decbytes",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "description": "unreleased snapshots",
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 6,
        "w": 8,
        "x": 0,
        "y": 123
      },
      "hiddenSeries": false,
      "id": 55,
      "legend": {
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": false,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "connected",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "flink_taskmanager_job_task_operator_counter_rocksdb_num_snapshots{job_name=~\"$job_name\"}",
          "format": "time_series",
          "intervalFactor": 2,
          "legendFormat": "{{host}}_{{job_name}}_{{tm_id}}",
          "metric": "flink_jobmanager_Status_JVM_Memory_Direct_MemoryUsed",
          "refId": "A",
          "step": 20
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "unreleased snapshots",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "cumulative"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "format": "decbytes",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "collapsed": false,
      "datasource": null,
      "gridPos": {
        "h": 1,
        "w": 24,
        "x": 0,
        "y": 129
      },
      "id": 45,
      "panels": [],
      "title": "System",
      "type": "row"
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "description": "jobmanager network Rate per second",
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 6,
        "w": 8,
        "x": 0,
        "y": 130
      },
      "hiddenSeries": false,
      "id": 31,
      "legend": {
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": true,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "connected",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pointradius": 0.5,
      "points": false,
      "renderer": "flot",
      "repeat": null,
      "repeatDirection": "h",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "flink_jobmanager_System_Network_bond0_ReceiveRate",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "{{host}} ReceiveRate",
          "refId": "A"
        },
        {
          "expr": "flink_jobmanager_System_Network_bond0_SendRate",
          "format": "time_series",
          "intervalFactor": 1,
          "legendFormat": "{{host}} SendRate",
          "refId": "B"
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "jobmanager network  Rate per second",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "decimals": null,
          "format": "bytes",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "cacheTimeout": null,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "description": "TaskManager network Rate per second",
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 6,
        "w": 8,
        "x": 8,
        "y": 130
      },
      "hiddenSeries": false,
      "id": 33,
      "legend": {
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": false,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "connected",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pluginVersion": "6.1.3",
      "pointradius": 0.5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "flink_taskmanager_System_Network_bond0_ReceiveRate",
          "format": "time_series",
          "hide": false,
          "instant": false,
          "interval": "",
          "intervalFactor": 1,
          "legendFormat": "{{host}}_{{tm_id}} ReceiveRate",
          "refId": "A"
        },
        {
          "expr": "flink_taskmanager_System_Network_bond0_SendRate",
          "format": "time_series",
          "hide": false,
          "instant": false,
          "interval": "",
          "intervalFactor": 1,
          "legendFormat": "{{host}}_{{tm_id}} SendRate",
          "refId": "B"
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "TaskManager network Rate per second",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "decbytes",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "cacheTimeout": null,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "description": "TaskManager CPU USE",
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 6,
        "w": 8,
        "x": 16,
        "y": 130
      },
      "hiddenSeries": false,
      "id": 49,
      "legend": {
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": false,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "connected",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pluginVersion": "6.1.3",
      "pointradius": 0.5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "flink_taskmanager_System_CPU_Usage",
          "format": "time_series",
          "hide": false,
          "instant": false,
          "interval": "",
          "intervalFactor": 2,
          "legendFormat": "{{host}}_{{tm_id}}",
          "refId": "A"
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "TaskManager CPU USE",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "percent",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "cacheTimeout": null,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$datasource",
      "description": "JobManager CPU USE",
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 6,
        "w": 8,
        "x": 0,
        "y": 136
      },
      "hiddenSeries": false,
      "id": 48,
      "legend": {
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": false,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "links": [],
      "nullPointMode": "connected",
      "options": {
        "dataLinks": []
      },
      "percentage": false,
      "pluginVersion": "6.1.3",
      "pointradius": 0.5,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "expr": "flink_jobmanager_System_CPU_Usage",
          "format": "time_series",
          "hide": false,
          "instant": false,
          "interval": "",
          "intervalFactor": 2,
          "legendFormat": "{{host}}",
          "refId": "A"
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "JobManager CPU USE",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "format": "percent",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    }
  ],
  "refresh": "",
  "schemaVersion": 21,
  "style": "dark",
  "tags": [
    "flink"
  ],
  "templating": {
    "list": [
      {
        "current": {
          "selected": true,
          "text": "Prometheus",
          "value": "Prometheus"
        },
        "hide": 0,
        "includeAll": false,
        "label": "数据源",
        "multi": false,
        "name": "datasource",
        "options": [],
        "query": "prometheus",
        "refresh": 1,
        "regex": "",
        "skipUrlSync": false,
        "type": "datasource"
      },
      {
        "allValue": null,
        "current": {
          "text": "LiveLinkAnalyseFlink",
          "value": [
            "LiveLinkAnalyseFlink"
          ]
        },
        "datasource": "$datasource",
        "definition": "label_values(job_name)",
        "hide": 0,
        "includeAll": true,
        "label": "作业名称",
        "multi": true,
        "name": "job_name",
        "options": [],
        "query": "label_values(job_name)",
        "refresh": 1,
        "regex": "",
        "skipUrlSync": false,
        "sort": 0,
        "tagValuesQuery": "",
        "tags": [],
        "tagsQuery": "",
        "type": "query",
        "useTags": false
      }
    ]
  },
  "time": {
    "from": "now-30m",
    "to": "now"
  },
  "timepicker": {
    "refresh_intervals": [
      "5s",
      "10s",
      "30s",
      "1m",
      "5m",
      "15m",
      "30m",
      "1h",
      "2h",
      "1d"
    ],
    "time_options": [
      "5m",
      "15m",
      "1h",
      "6h",
      "12h",
      "24h",
      "2d",
      "7d",
      "30d"
    ]
  },
  "timezone": "browser",
  "title": "Flink Dashboard",
  "uid": "-0rFuzoZk",
  "version": 58
}
```
### Reference

https://www.cnblogs.com/Springmoon-venn/p/11445023.html

https://segmentfault.com/a/1190000017833186?utm_source=tag-newest
