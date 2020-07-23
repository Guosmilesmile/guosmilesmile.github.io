---
title: Flink on kubernetes 宕机恢复
date: 2020-07-04 16:48:34
tags:
categories:
	- Kubernetes
	- Flink
---


## 前言


笔者的kubernetes版本是1.17，有一次凌晨机器宕机了，运行在kubernetes上的flink集群因为pod没有重建，服务影响了将近20分钟后人工介入，发现落在宕机机器上的pod没有重生 。而且JobManager也刚好落在这台机器上，后续进行了一次宕机测试 。


## node宕机后的pod驱逐

在 1.5 版本之前的 Kubernetes 上，节点控制器会将不能访问的 Pods 从 apiserver 中 强制删除。 但在 1.5 或更高的版本里，在节点控制器确认这些 Pods 在集群中已经停止运行前，不会强制删除它们。 你可以看到这些可能在无法访问的节点上运行的 Pods 处于 Terminating 或者 Unknown 状态。 如果 kubernetes 不能基于下层基础设施推断出某节点是否已经永久离开了集群，集群管理员可能需要手动删除该节点对象。 从 Kubernetes 删除节点对象将导致 apiserver 删除节点上所有运行的 Pod 对象并释放它们的名字。


默认为5分钟后驱逐

```shell
kubectl describe pod  nginx-deployment|grep -i toleration -A 2
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:          <none>
--
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:          <none>
```


在Deployment增加

```yaml
tolerations:
      - key: "node.kubernetes.io/unreachable"
        operator: "Exists"
        effect: "NoExecute"
        tolerationSeconds: 60
      - key: "node.kubernetes.io/not-ready"
        operator: "Exists"
        effect: "NoExecute"
        tolerationSeconds: 60
```

## flink注册失败重启

JM宕机重生后，旧的TM还认着之前的JM，需要等待TM认为注册resource失败后shutdow后被kubernetes重启拉起才可以重新注册，flink默认注册失败重启需要5min。

修改如下配置可以缩短为1min，taskmanager.registration.timeout: 1 min


## 总结

修改两个参数，可以在机器宕机后，并且还是jobmanager在上面的情况下较快恢复。k8s默认驱逐pod需要5min，flink默认注册失败重启需要5min。一共需要10min以上。所以昨晚需要人工接入，人工介入的时间比自动短