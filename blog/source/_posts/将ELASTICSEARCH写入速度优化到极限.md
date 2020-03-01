---
title: 将ELASTICSEARCH写入速度优化到极限
date: 2020-02-20 16:15:48
tags:
categories:
	- Elasticsearch
---




### 转载自

https://www.easyice.cn/archives/207


综合来说,提升写入速度从以下几方面入手:

* 加大 translog flush ,目的是降低 iops,writeblock
* 加大 index refresh间隔, 目的除了降低 io, 更重要的降低了 segment merge 频率
* 调整 bulk 线程池和队列
* 优化磁盘间的任务均匀情况,将 shard 尽量均匀分布到物理主机的各磁盘
* 优化节点间的任务分布,将任务尽量均匀的发到各节点
* 优化 lucene 层建立索引的过程,目的是降低 CPU 占用率及 IO


### translog flush 间隔调整

从 es 2.x 开始, 默认设置下,translog 的持久化策略为:每个请求都flush.对应配置项为:

```
index.translog.durability: request
```

**这是影响 es 写入速度的最大因素**.但是只有这样,写操作才有可能是可靠的,原因参考写入流程.



如果系统可以接受一定几率的数据丢失,调整 translog 持久化策略为周期性和一定大小的时候 flush:

```
index.translog.durability: async
index.translog.sync_interval: 120s
index.translog.flush_threshold_size: 1024mb
index.translog.flush_threshold_period: 120m
```

### 索引刷新间隔调整: refresh_interval

#### refresh_interval
默认情况下索引的refresh_interval为1秒,这意味着数据写1秒后就可以被搜索到,每次索引的 refresh 会产生一个新的 lucene 段,这会导致频繁的 segment merge 行为,如果你不需要这么高的搜索实时性,应该降低索引refresh 周期,如:

```
index.refresh_interval: 120s
```

#### segment merge

segment merge 操作对系统 CPU 和 IO 占用都比较高,从es 2.0开始,merge 行为不再由 es 控制,而是转由 lucene 控制,因此以下配置已被删除:

```
indices.store.throttle.type
indices.store.throttle.max_bytes_per_sec
index.store.throttle.type
index.store.throttle.max_bytes_per_sec
```
改为以下调整开关:

```
index.merge.scheduler.max_thread_count
index.merge.policy.*
```
最大线程数的默认值为:
```
Math.max(1, Math.min(4, Runtime.getRuntime().availableProcessors() / 2))
```

是一个比较理想的值,如果你只有一块硬盘并且非 SSD, 应该把他设置为1,因为在旋转存储介质上并发写,由于寻址的原因,不会提升,只会降低写入速度.

merge 策略有三种:

* tiered
* log_byete_size
* log_doc

默认情况下:

```
index.merge.polcy.type: tiered
```

索引创建时合并策略就已确定,不能更改,但是可以动态更新策略参数,一般情况下,不需要调整.如果堆栈经常有很多 merge, 可以尝试调整以下配置:

```
index.merge.policy.floor_segment
```
该属性用于阻止segment 的频繁flush, 小于此值将考虑优先合并,默认为2M,可考虑适当降低此值

index.merge.policy.segments_per_tier
 
该属性指定了每层分段的数量,取值约小最终segment 越少,因此需要 merge 的操作更多,可以考虑适当增加此值.默认为10,他应该大于等于 index.merge.policy.max_merge_at_once 

```
index.merge.policy.max_merged_segment
```
指定了单个 segment 的最大容量,默认为5GB,可以考虑适当降低此值

#### Indexing Buffer
indexing buffer在为 doc 建立索引时使用,当缓冲满时会刷入磁盘,生成一个新的 segment, 这是除refresh_interval外另外一个刷新索引,生成新 segment 的机会. 每个 shard 有自己的 indexing buffer,下面的关于这个 buffer 大小的配置需要除以这个节点上所有的 shard 数量

```
indices.memory.index_buffer_size
```

默认为整个堆空间的10%

```
indices.memory.min_index_buffer_size
```

默认48mb

```
indices.memory.max_index_buffer_size
```
默认无限制

在大量的索引操作时,indices.memory.index_buffer_size默认设置可能不够,这和可用堆内存,单节点上的 shard 数量相关,可以考虑适当增大.

#### bulk 线程池和队列大小

建立索引的过程偏计算密集型任务,应该使用固定大小的线程池配置,来不及处理的放入队列,线程数量配置为 CPU 核心数+1,避免过多的上下文切换.队列大小可以适当增加

### 磁盘间的任务均衡


如果你的部署方案是为path.data 配置多个路径来使用多块磁盘, es 在分配 shard 时,落到各磁盘上的 shard 可能并不均匀,这种不均匀可能会导致某些磁盘繁忙,利用率达到100%,这种不均匀达到一定程度可能会对写入性能产生负面影响.


es 在处理多路径时,优先将 shard 分配到可用空间百分比最多的磁盘,因此短时间内创建的 shard 可能被集中分配到这个磁盘,即使可用空间是99%和98%的差别.后来 es 在2.x 版本中开始解决这个问题的方式是:预估一下这个 shard 会使用的空间,从磁盘可用空间中减去这部分,直到现在6.x beta 版也是这种处理方式.但是实现也存在一些问题:

```
从可用空间减去预估大小
```

这种机制只存在于一次索引创建的过程中,下一次的索引创建,磁盘可用空间并不是上次做完减法以后的结果,这也可以理解,毕竟预估是不准的,一直减下去很快就减没了.

但是最终的效果是,这种机制并没有从根本上解决问题,即使没有完美的解决方案,这种机制的效果也不够好.

如果单一的机制不能解决所有的场景,至少应该为不同场景准备多种选择.

### 节点间的任务均衡


为了在节点间任务尽量均衡,数据写入客户端应该把 bulk 请求轮询发送到各个节点.

当使用 java api ,或者 rest api 的 bulk 接口发送数据时,客户端将会轮询的发送的集群节点,节点列表取决于:

当client.transport.sniff为 true,(默认为 false),列表为所有数据节点
否则,列表为初始化客户端对象时添加进去的节点.


java api 的 TransportClient 和 rest api 的 RestClient 都是线程安全的,当写入程序自己创建线程池控制并发,应该使用同一个 Client 对象.在此建议使用 rest api,兼容性好,只有吞吐量非常大才值得考虑序列化的开销,显然搜索并不是高吞吐量的业务.

观察bulk 请求在不同节点上的处理情况,通过cat 接口观察 bulk 线程池和队列情况,是否存在不均:


```
GET _cat/thread_pool?v
```

### 索引过程调整和优化


#### 自动生成 doc ID


分析 es 写入流程可以看到,写入 doc 时如果是外部指定了 id,es 会先尝试读取原来doc的版本号, 判断是否需要更新,使用自动生成 doc id 可以避免这个环节.

#### 调整字段 Mappings
**字段的 index 属性设置为: not_analyzed,或者 no**

对字段不分词,或者不索引,可以节省很多运算,降低 CPU 占用.尤其是 binary 类型,默认情况下占用 CPU 非常高,而这种类型根本不需要进行分词做索引.

单个 doc 在建立索引时的运算复杂度,最大的因素 不在于 doc 的字节数或者说某个字段 value 的长度,而是字段的数量. 例如在满负载的写入压力测试中,mapping 相同的情况下,一个有10个字段,200字节的 doc, 通过增加某些字段 value 的长度到500字节,写入 es 时速度下降很少,而如果字段数增加到20,即使整个 doc 字节数没增加多少,写入速度也会降低一倍.

**使用不同的分析器:analyzer**

不同的分析器在索引过程中运算复杂度也有较大的差异


#### 调整_source 字段

_source 字段用于存储 doc 原始数据,对于部分不需要存储的字段,可以通过 includes excludes 来过滤,或者将 _source 禁用,一般用于索引和数据分离

这样可以降低 io 的压力,不过实际场景大多数情况不会禁用 _source ,而即使过滤掉某些字段,对于写入速度的提示效果也不大,满负荷写入情况下,基本是CPU 先跑满了,瓶颈在于 CPU.

#### 对于 Analyzed 的字段禁用 Norms
Norms 用于在搜索时计算 doc 的评分,如果不需要评分,可以禁用他:

```
"title": {"type": "string","norms": {"enabled": false}}
```

#### index_options 设置
index_options 用于控制在建立倒排索引过程中,哪些内容会被添加到倒排,例如 doc数量,词频,positions,offsets等信息,优化这些设置可以一定程度降低索引过程中运算任务,节省 CPU 占用率

不过实际场景中,通常很难确定业务将来会不会用到这些信息,除非一开始方案就明确这样设计的




