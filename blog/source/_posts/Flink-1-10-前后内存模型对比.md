---
title: Flink 1.10 前后内存模型对比
date: 2020-02-15 22:47:17
tags:
categories:
	- Flink
---

## 新旧内存模型

1.10 之前的内存模式

![image](https://note.youdao.com/yws/api/personal/file/6AEE4C670C9E48F09E31E92F1A1EE71E?method=download&shareKey=0b4e60db900be929da570ba01ba4afbe)

Flink 内存主要指 TaskManager 运行时提供的内存资源。

TaskManager 主要由几个内部组件构成: 
* 负责和 JobManager 等进程通信的 actor 系统
* 负责在内存不足时将数据溢写到磁盘和读回的 IOManager
* 负责内存管理的 MemoryManager。

其中 actor 系统和 MemoryManager 会要求大量的内存。相应地，Flink 将 TaskManager 的运行时 JVM heap 分为 Network Buffers、MemoryManager 和 Free 三个区域（在 streaming 模式下只存在 Network Buffers 和 Free 两个区域，因为算子不需要缓存一次读入的大量数据）。



1.10 开始的内存模型

![image](https://note.youdao.com/yws/api/personal/file/74E93C992E78417FAC24C8E603D2DF72?method=download&shareKey=5f4c2cd2feeec20b8de565f23d0a81e7)






### 调整原因 

TaskExecutor 在不同部署模式下具体负责作业执行的进程，可以简单视为 TaskManager。目前 TaskManager 的内存配置存在不一致以及不够直观的问题，具体有以下几点:
* 流批作业内容配置不一致。Managed Memory 只覆盖 DataSet API，而 DataStream API 的则主要使用 JVM 的 heap 内存，相比前者需要更多的调优参数且内存消耗更难把控。
* RocksDB 占用的 native 内存并不在内存管理里，导致使用 RocksDB 时内存需要很多手动调优。
* 不同部署模式下，Flink 内存计算算法不同，并且令人难以理解。


针对这些问题，FLIP-49[4] 提议通过将 Managed Memory 的用途拓展至 DataStream 以解决这个问题。DataStream 中主要占用内存的是 StateBackend，它可以从管理 Managed Memory 的 MemoryManager 预留部分内存或分配内存。通过这种方式同一个 Flink 配置可以运行 Batch 作业和 Streaming 作业，有利于流批统一。

下图是两个模型的对比


![image](https://note.youdao.com/yws/api/personal/file/180E7F756B5E4F7B976C65CF960D3788?method=download&shareKey=77bb768e9f739f1e4594b6ad4c7a241d)


很明显的差别是Flink的 Managed Memory 从 heap转移到了off-heap



分区 | 内存类型 |  描述 | 配置项 |	默认值
---|---|---|---|---|---
Framework Heap Memory | heap |	Flink 框架消耗的 heap 内存 |taskmanager.memory.framework.heap |	128mb
Task Heap Memory | heap | 用户代码使用的 heap 内存| taskmanager.memory.
task.heap | 无
Task Off-Heap Memory  | off-heap | 用户代码使用的 off-heap 内存 | taskmanager.memory.
task.offheap | 0b
Shuffle Memory | off-heap | 网络传输/suffle 使用的内存 | taskmanager.memory.shuffle.[min/max/fraction]	| min=64mb, max=1gb, fraction=0.1 
Managed Heap Memory | heap | Managed Memory 使用的 heap 内存| taskmanager.memory.managed.[size/fraction]| fraction=0.5
Managed Off-heap Memory |   off-heap | Managed Memory 使用的 off-heap 内存 | taskmanager.memory.managed.offheap-fraction | 0.0
JVM Metaspace |	off-heap |	JVM metaspace 使用的 off-heap 内存 |taskmanager.memory.jvm-metaspace	| 192mb
JVM Overhead | off-heap | JVM 本身使用的内存| taskmanager.memory.jvm-overhead.[min/max/fraction] | min=128mb, max=1gb, fraction=0.1)
Total Flink Memory |	heap & off-heap	Flink |框架使用的总内存，是以上除 JVM Metaspace 和 JVM Overhead 以外所有分区的总和| taskmanager.memory.total-flink.size	| 无
otal Process Memory |	heap & off-heap |	进程使用的总内存，是所有分区的总和，包括 JVM Metaspace 和 JVM Overhead	|taskmanager.memory.total-process.size|	无


值得注意的是有 3 个分区是没有默认值的，包括 Framework Heap Memory、Total Flink Memory 和 Total Process Memory，它们是决定总内存的最关键参数，三者分别满足不同部署模式的需要。比如在 Standalone 默认下，用户可以配置 Framework Heap Memory 来限制用户代码使用的 heap 内存；而在 YARN 部署模式下，用户可以通过配置 YARN container 的资源来间接设置 Total Process Memory，如果是docker之类的，需要设置这个参数。


在旧的模型中，如果配置了taskmanager.heap.size的内存为10G，并且限制了容器的内存为10G，那么堆外内存(network buffer、RocksDB 占用的 native 内存里)和jvm本身需要的内存(cut-off)将无内存可分配，直接导致容器被kill掉，不够直观。

在新的内存模型中，直接以Total Process Memory开发出来作为内存配置，直接等价于容器的内存，就可以比较直观的进行调整。



```
jobmanager.rpc.address: flink-jobmanager
    taskmanager.numberOfTaskSlots: 15
    blob.server.port: 6124
    jobmanager.rpc.port: 6123
    taskmanager.rpc.port: 6122
    taskmanager.memory.process.size: 30000m
    jobmanager.heap.size: 1024m
    jobstore.expiration-time: 172800
    taskmanager.memory.managed.fraction: 0.2
    taskmanager.network.memory.min: 2gb
    taskmanager.network.memory.max: 3gb
    taskmanager.memory.task.off-heap.size: 1024m
    taskmanager.network.memory.floating-buffers-per-gate: 16
    taskmanager.network.memory.buffers-per-channel: 4
    akka.ask.timeout: 30s 
    akka.framesize: 104857600b
    restart-strategy: failure-rate

```
### Referenece

https://www.whitewood.me/2019/10/17/Flink-1-10-%E7%BB%86%E7%B2%92%E5%BA%A6%E8%B5%84%E6%BA%90%E7%AE%A1%E7%90%86%E8%A7%A3%E6%9E%90/