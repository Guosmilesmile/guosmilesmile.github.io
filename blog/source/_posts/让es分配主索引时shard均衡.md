---
title: 让es分配主索引时shard均衡
date: 2020-03-01 21:33:07
tags:
categories:
	- Elasticsearch
---




### 现象

由于es在分配shard的时候，会根据磁盘容量分配shard，导致会出现shard会集中在某一台机器上，出现很严重的倾斜

### 初始解决方法

采用
```
index.routing.allocation.total_shards_per_node

控制单个索引在一个节点上的最大分片数，默认值是不限制。
```

#### 问题

采用上面的方法，如果没有副本，会先填充磁盘使用低的机器，再填充其他机器。如果shard数量和机器数量没有调整好，会出现问题。

如果有副本，由于es是先创建主shard，在生成副本，会出现磁盘使用率低的机器全是主shard，在高峰期写入会扛不住。

### 解决方法


先观察一下下面的参数

####  cluster.routing.allocation.balance.shard

控制各个节点分片数一致的权重，默认值是 0.45f。增大该值，分配 shard 时，Elasticsearch 在不违反 Allocation Decider 的情况下，尽量保证集群各个节点上的分片数是相近的。

#### cluster.routing.allocation.balance.index

控制单个索引在集群内的平衡权重，默认值是 0.55f。增大该值，分配 shard 时，Elasticsearch 在不违反 Allocation Decider 的情况下，尽量将该索引的分片平均的分布到集群内的节点。



其实对我们来说，只用将主shard均衡分配到每台机器上，副本也相对均衡。


```
cluster.routing.allocation.balance.shard = 0.01
cluster.routing.allocation.balance.index = 0.99
```

### Reference

[高可用 Elasticsearch 集群的分片管理 （Shard）](https://www.jianshu.com/p/210465322e18)