---
title: 流计算中的Exactly Once特性
date: 2021-08-15 21:10:12
tags:
categories:
	- Flink
---


## 背景

流处理可以简单地描述为是对无界数据或事件的连续处理。流或事件处理应用程序可以或多或少地被描述为有向图，并且通常被描述为有向无环图（DAG）。sources读取外部数据/事件到应用程序中，而 sinks 通常会收集应用程序生成的结果。下图是流式应用程序的示例。

![image](https://note.youdao.com/yws/api/personal/file/WEB7aff0947790566eb226f32cde5f95687?method=download&shareKey=97559b5f2f84b59f1353e99d486f4ada)

流处理引擎通常允许用户指定可靠性模式或处理语义，以指示它将为整个应用程序中的数据处理提供哪些保证。

## 处理语义

### 最多一次（At-most-once）

这本质上是一『尽力而为』的方法。保证数据或事件最多由应用程序中的所有算子处理一次。 这意味着如果数据在被流应用程序完全处理之前发生丢失，则不会进行其他重试或者重新发送。

![image](https://note.youdao.com/yws/api/personal/file/WEB4ebc056efb6ca1f5ba29461fbd878df4?method=download&shareKey=621b8768dd3390c1b2d8e0663e113aac)


例如kafka刚消费下来，还没处理完提交到下游，作业就挂了，这个时候consumer已经提交了offset。

### 至少一次（At-least-once） 

应用程序中的所有算子都保证数据或事件至少被处理一次。这通常意味着如果事件在流应用程序完全处理之前丢失，则将从源头重放或重新传输事件。

![image](https://note.youdao.com/yws/api/personal/file/WEBe24223c3f69dad21b0ed8b7de145af81?method=download&shareKey=f5814776c30c2931a6abae226f47adaa)


### 精确一次（Exactly-once）

即使是在各种故障的情况下，流应用程序中的所有算子都保证事件只会被『精确一次』的处理。

通常使用两种流行的机制来实现『精确一次』处理语义。

* 分布式快照 / 状态检查点
* 至少一次事件传递和对重复数据去重

实现『精确一次』的分布式快照/状态检查点方法受到 Chandy-Lamport 分布式快照算法的启发。

> Chandy, K. Mani and Leslie Lamport.Distributed snapshots: Determining global states of distributed systems. ACMTransactions on Computer Systems (TOCS) 3.1 (1985): 63-75.

通过这种机制，流应用程序中每个算子的所有状态都会定期做 checkpoint。如果是在系统中的任何地方发生失败，每个算子的所有状态都回滚到最新的全局一致 checkpoint 点。在回滚期间，将暂停所有处理。源也会重置为与最近 checkpoint 相对应的正确偏移量。整个流应用程序基本上是回到最近一次的一致状态，然后程序可以从该状态重新启动。

![image](https://note.youdao.com/yws/api/personal/file/WEB60cd695b48054fddfec9faa0fcfa2f00?method=download&shareKey=5e235ba749daecd5665bde0875487aa8)

在上图中，流应用程序在 T1 时间处正常工作，并且做了checkpoint。然而，在时间 T2，算子未能处理输入的数据。此时，S=4 的状态值已保存到持久存储器中，而状态值 S=12 保存在算子的内存中。为了修复这种差异，在时间 T3，处理程序将状态回滚到 S=4 并“重放”流中的每个连续状态直到最近，并处理每个数据。最终结果是有些数据已被处理了多次，但这没关系，因为无论执行了多少次回滚，结果状态都是相同的。

另一种实现『精确一次』的方法是：在每个算子上实现至少一次事件传递和对重复数据去重来。使用此方法的流处理引擎将重放失败事件，以便在事件进入算子中的用户定义逻辑之前，进一步尝试处理并移除每个算子的重复事件。此机制要求为每个算子维护一个事务日志，以跟踪它已处理的事件。利用这种机制的引擎有 Google 的 MillWheel[2] 和 Apache Kafka Streams。


![image](https://note.youdao.com/yws/api/personal/file/WEBf03b4e5eddf5a218522bb193ba03c7bd?method=download&shareKey=96dd4747d6ea5e626fe3d4807575c482)


### Exactly-once本质

当引擎声明『精确一次』处理语义时，它们实际上是在说，它们可以保证引擎管理的状态更新只提交一次到持久的后端存储。**事件的处理可以发生多次，但是该处理的效果只在持久后端状态存储中反映一次。**

上面描述的两种机制都使用持久的后端存储作为真实性的来源，可以保存每个算子的状态并自动向其提交更新。对于机制 1 (分布式快照 / 状态检查点)，此持久后端状态用于保存流应用程序的全局一致状态检查点(每个算子的检查点状态)。对于机制 2 (至少一次事件传递加上重复数据删除)，持久后端状态用于存储每个算子的状态以及每个算子的事务日志，该日志跟踪它已经完全处理的所有事件。

### 事务与exectly-once

事务与exectly-once是两个容易混淆的概念，前者保证数据能够“打包”成整体处理，后者保证数据的精准投递，两者概念不同但相辅相成。

支持事务，只能保证事务单元整理处理，无法解决事务单元处理多次问题；
    
    
### 两种Exactly-once的比较

从语义的角度来看，分布式快照和至少一次事件传递以及重复数据删除机制都提供了相同的保证。然而，由于两种机制之间的实现差异，存在显着的性能差异。


#### 分布式快照 / 状态检查点

性能开销是最小，因为引擎实际上是往流应用程序中的所有算子一起发送常规事件和特殊事件，而状态检查点可以在后台异步执行。（事件和数据本质其实都是event）

但是，对于大型流应用程序，故障可能会更频繁地发生，导致引擎需要暂停应用程序并回滚所有算子的状态，这反过来又会影响性能。流式应用程序越大，故障发生的可能性就越大，因此也越频繁，反过来，流式应用程序的性能受到的影响也就越大。然而，这种机制是非侵入性的，运行时需要的额外资源影响很小。


#### 至少一次事件传递加重复数据删除

需要更多资源，尤其是存储后端。使用此机制，引擎需要能够跟踪每个算子实例已完全处理的每个元组，以执行重复数据删除，以及为每个事件执行重复数据删除本身。这意味着需要跟踪大量的数据，尤其是在流应用程序很大或者有许多应用程序在运行的情况下。执行重复数据删除的每个算子上的每个事件都会产生性能开销。但是，使用这种机制，流应用程序的性能不太可能受到应用程序大小的影响。

#### 总体比较

分布式快照 / 状态检查点的优缺点：

优点：
* 较小的性能和资源开销
缺点：
* 对性能的影响较大
* 拓扑越大，对性能的潜在影响越大

至少一次事件传递以及重复数据删除机制的优缺点：

优点：
* 故障对性能的影响是局部的
* 故障的影响不一定会随着拓扑的大小而增加

缺点：
* 可能需要大量的存储和基础设施来支持
* 每个算子的每个事件的性能开销

虽然从理论上讲，分布式快照和至少一次事件传递加重复数据删除机制之间存在差异，但两者都可以简化为至少一次处理加幂等性。对于这两种机制，当发生故障时(至少实现一次)，事件将被重放/重传，并且通过状态回滚或事件重复数据删除，算子在更新内部管理状态时本质上是幂等的。


## 端到端Exactly-Once

上面的机制实现了集群内计算任务的 Exactly Once 语义，但是仍然实现不了在输入和输出两端数据不丢不重，集群的计算任务可以回放，下游没办法回放，这就要求下游等系统也得支持Exactly Once或者相对的Exactly Once（支持回滚）

所以要求，上游支持回放数据，计算保证 Exactly Once，下游支持回滚。

## Flink的一致性



### Flink的CheckPoint功能简介

Flink CheckPoint 的存在就是为了解决flink任务failover掉之后，能够正常恢复任务。


CheckPoint是通过给程序快照的方式使得将历史某些时刻的状态保存下来，当任务挂掉之后，默认从最近一次保存的完整快照处进行恢复任务。

![image](https://note.youdao.com/yws/api/personal/file/DACDA0D36E65409599BBB080EA633019?method=download&shareKey=9adc9bb6af2b7599ff382a0408b0c26f)

### 案例

app的pv，flink该怎么统计呢？

从Kafka读取到一条条的日志，从日志中解析出app_id，然后将统计的结果放到内存中一个Map集合，app_id做为key，对应的pv做为value，每次只需要将相应app_id 的pv值+1后put到Map中即可

![image](https://note.youdao.com/yws/api/personal/file/27D1AD836DC04280A2E3D7C7FACE35F9?method=download&shareKey=4ef1e56e0b3924ad270a9b45c756fa6f)


flink的Source task记录了当前消费到kafka test topic的所有partition的offset，为了方便理解CheckPoint的作用，这里先用一个partition进行讲解，假设名为 “test”的 topic只有一个partition0

```
例：（0，1000）
表示0号partition目前消费到offset为1000的数据
```

flink的pv task记录了当前计算的各app的pv值

```
例：（app1，50000）（app2，10000）
表示app1当前pv值为50000
表示app2当前pv值为10000
```
每来一条数据，只需要确定相应app_id，将相应的value值+1后put到map中即可

#### checkPoint的作用

checkPoint记录了第n次CheckPoint消费的offset信息和各app的pv值信息，记录一下发生CheckPoint当前的状态信息，并将该状态信息保存到相应的状态后端

```
chk-100
offset：（0，1000）
pv：（app1，50000）（app2，10000）
```

该状态信息表示第100次CheckPoint的时候， partition 0 offset消费到了1000，pv统计结果为（app1，50000）（app2，10000）

#### 任务挂了如何恢复

如果任务挂了flink只需要从最近一次成功的CheckPoint保存的offset（0，1000）处接着消费即可，当然pv值也要按照状态里的pv值（app1，50000）（app2，10000）进行累加。


### 原理

![image](https://note.youdao.com/yws/api/personal/file/AFC5592903304F76BC9C3632FBDCCD02?method=download&shareKey=b16fc1c90a635ee142d5d2fbe1422679)

* barrier从Source Task处生成，一直流到Sink Task，期间所有的Task只要碰到barrier，就会触发自身进行快照
* * CheckPoint barrier n-1处做的快照就是指Job从开始处理到 barrier n-1所有的状态数据
* * barrier n 处做的快照就是指从Job开始到处理到 barrier n所有的状态数据



### 多并行度、多Operator情况下，CheckPoint过程

所有的Operator运行过程中遇到barrier后，都对自身的状态进行一次快照，保存到相应状态后端


![image](https://note.youdao.com/yws/api/personal/file/CF040D5C0C084F4CA96BBDEDEC7642DA?method=download&shareKey=82ed7ec2664971e9522dd892c443b0ed)


多Operator状态恢复

![image](https://note.youdao.com/yws/api/personal/file/5AB07951DFFD49BCAB1894A7B8C18541?method=download&shareKey=2f7f878f15c086fb4f7d6d1a00201b73)

JobManager向Source Task发送CheckPointTrigger，Source Task会在数据流中安插CheckPoint barrier

![image](https://note.youdao.com/yws/api/personal/file/AB184338C3094C9C98D897D864B1F865?method=download&shareKey=f6858815be435fd9706076899e26f39a)

Source Task自身做快照，并保存到状态后端，Source Task将barrier跟数据流一块往下游发送。当下游的Operator实例接收到CheckPoint barrier后，对自身做快照。

![image](https://note.youdao.com/yws/api/personal/file/25DED52D106A49E89E0F7D2229919F6D?method=download&shareKey=6943719a8084298b1f6fd4db28326802)
上述图中，有4个带状态的Operator实例，相应的状态后端就可以想象成填4个格子。整个CheckPoint 的过程可以当做Operator实例填自己格子的过程，Operator实例将自身的状态写到状态后端中相应的格子，当所有的格子填满可以简单的认为一次完整的CheckPoint做完了


#### 整个CheckPoint执行过程如下

1. JobManager端的 CheckPointCoordinator向所有SourceTask发送CheckPointTrigger，Source Task会在数据流中安插CheckPoint barrier
2.  当task收到所有的barrier后，向自己的下游继续传递barrier，然后自身执行快照，并将自己的状态异步写入到持久化存储中
增量CheckPoint只是把最新的一部分更新写入到 外部存储
为了下游尽快做CheckPoint，所以会先发送barrier到下游，自身再同步进行快照
* 增量CheckPoint只是把最新的一部分更新写入到 外部存储(例如时间窗口内等所有数据汇聚完在做计算的算子，增量比全量好)
* 为了下游尽快做CheckPoint，所以会先发送barrier到下游，自身再同步进行快照
3. 当task完成备份后，会将备份数据的地址（state handle）通知给JobManager的CheckPointCoordinator
* 如果CheckPoint的持续时长超过 了CheckPoint设定的超时时间，CheckPointCoordinator 还没有收集完所有的 State Handle，CheckPointCoordinator就会认为本次CheckPoint失败，会把这次CheckPoint产生的所有 状态数据全部删除
4.  最后 CheckPoint Coordinator 会把整个 StateHandle 封装成 completed CheckPoint Meta，写入到hdfs


### barrier对齐

![image](https://note.youdao.com/yws/api/personal/file/4F2FD952BD414B81A43DF0E0821BD34D?method=download&shareKey=d6df6aa9ad7390bf00dc872b8f3a6a41)


* 一旦Operator从输入流接收到CheckPoint barrier n，它就不能处理来自该流的任何数据记录，直到它从其他所有输入接收到barrier n为止。否则，它会混合属于快照n的记录和属于快照n + 1的记录
* 接收到barrier n的流暂时被搁置。从这些流接收的记录不会被处理，而是放入输入缓冲区。
* * 上图中第2个图，虽然数字流对应的barrier已经到达了，但是barrier之后的1、2、3这些数据只能放到buffer中，等待字母流的barrier到达
* 一旦最后所有输入流都接收到barrier n，Operator就会把缓冲区中pending 的输出数据发出去，然后把CheckPoint barrier n接着往下游发送
* 之后，Operator将继续处理来自所有输入流的记录，在处理来自流的记录之前先处理来自输入缓冲区的记录


##### 什么是barrier不对齐？

barrier不对齐就是指当还有其他流的barrier还没到达时，为了不影响性能，也不用理会，直接处理barrier之后的数据。等到所有流的barrier的都到达后，就可以对该Operator做CheckPoint了

Exactly Once时必须barrier对齐，如果barrier不对齐就变成了At Least Once



### Flink端到端的Exactly-Once

要使数据输出端提供Exactly-Once保证，它必须将所有数据通过一个事务提交给外部系统，在这种情况下，为了提供Exactly-Once保证，外部系统必须支持事务，这样才能和两阶段提交协议集成。

![image](https://note.youdao.com/yws/api/personal/file/49500B5399F342E5BA4BFA5FBB487B4C?method=download&shareKey=63b01ba174583757b860b379e9a6800b)

当checkpoint barrier在所有operator都传递了一遍，并且触发的checkpoint回调成功完成时，预提交阶段就结束了。所有触发的状态快照都被视为该checkpoint的一部分。checkpoint是整个应用程序状态的快照，包括预先提交的外部状态。如果发生故障，我们可以回滚到上次成功完成快照的时间点。

下一步是通知所有operator，checkpoint已经成功了。这是两阶段提交协议的提交阶段，JobManager为应用程序中的每个operator发出checkpoint已完成的回调。

数据源和widnow operator没有外部状态，因此在提交阶段，这些operator不必执行任何操作。但是，数据输出端（Data Sink）拥有外部状态，此时应该提交外部事务。

![image](https://note.youdao.com/yws/api/personal/file/F4FBF9FDB7E644CCB270891DF8AD9415?method=download&shareKey=14ef4a48b8fcfcf6dbd5a521ffbbc343)

![image](https://note.youdao.com/yws/api/personal/file/3DF290A566CB458687E51539F0C07061?method=download&shareKey=7ba23b905d82733fdce6f5dcc6678f25)









### Reference

https://segmentfault.com/a/1190000019353382


http://www.zdingke.com/2018/09/27/flume%E7%9A%84%E4%BA%8B%E5%8A%A1%E6%9C%BA%E5%88%B6%E5%92%8C%E5%8F%AF%E9%9D%A0%E6%80%A7/


https://www.cnblogs.com/tuowang/p/9022198.html

https://www.jdon.com/48558


https://cloud.tencent.com/developer/article/1591349


https://cloud.tencent.com/developer/article/1438832?from=article.detail.1591349


https://www.jianshu.com/p/8d1681f05d88


https://ververica.cn/developers/flink-kafka-end-to-end-exactly-once-analysis/