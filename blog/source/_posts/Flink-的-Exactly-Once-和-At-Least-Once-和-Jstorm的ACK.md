---
title: Flink 的 Exactly Once 和 At Least Once  和 Jstorm的ACK
date: 2019-08-10 23:21:26
tags:
categories:
	- Flink
---


### 有状态与无状态

* 无状态：数据的计算与上一次的计算结果无关。例如map,flatMap
* 有状态: 数据的计算与上一次的计算结果有关，例如时间窗口内的sum，需要累加求和。


无状态计算的例子

* 比如：我们只是进行一个字符串拼接，输入 a，输出 a_666,输入b，输出 b_666
输出的结果跟之前的状态没关系，符合幂等性。
* * 幂等性：就是用户对于同一操作发起的一次请求或者多次请求的结果是一致的，不会因为多次点击而产生了副作用

有状态计算的例子
* 计算pv、uv
* 输出的结果跟之前的状态有关系，不符合幂等性，访问多次，pv会增加


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




### Jstorm 消息可靠性

#### 消息树
![image](https://note.youdao.com/yws/api/personal/file/50CED120A8D848FBA99FDEDEB7038546?method=download&shareKey=86148fc758742fb66dc035c5f81f6e37)

#### 一个消息被完整处理是什么意思？


指一个从Spout发出的元组（tuple）所触发的消息树中所有的消息都被Storm处理了。如果在指定的超时时间里，这个Spout元组触发的消息树中有任何一个消息没有处理完，就认为这个Spout元组处理失败了。这个超时时间是通过每个拓扑的Config.TOPOLOGY_MESSAGE_TIMEOUT_SECS配置项来进行配置的，默认是30秒

在前面消息树的例子里，只有消息树中所有的消息(包含一条Spout消息，六条split Bolt消息,六条count Bolt消息)都被Storm处理完了，才算是这条Spout消息被完整处理了。


在Storm消息树(元组树)中添加一个子结点的操作叫做锚定(anchoring)。在应用程序发送一个新元组时候，Storm会在幕后做锚定。
```java
//锚定
_collector.emit(tuple, new Values(word));
//不锚定
_collector.emit(new Values(word));
```

如果你不在意某个消息派生出来的子孙消息的可靠性，则此消息派生出来的子消息在发送时不要做锚定，即在emit方法中不指定输入消息。因为这些子孙消息没有被锚定在任何tuple tree中，因此他们的失败不会引起任何spout重新发送消息。


### 元组处理完后通知Storm


锚定的作用就是指定元组树的结构–下一步是当元组树中某个元组已经处理完成时，通知Storm。通知是通过OutputCollector中的ack和fail函数来完成的。例如上面流式计算单词个数例子中的split Bolt的实现SplitSentence类，可以看到句子被切分成单词后，当所有的单词元组都被发射后，会确认(ack)输入的元组处理完成。


可以利用OutputCollector的fail函数来立即通知Storm，当前消息树的根元组处理失败了。例如，应用程序可能捕捉到了数据库客户端的一个异常，就显示地通知Storm输入元组处理失败。通过显示地通知Storm元组处理失败，这个Spout元组就不用等待超时而能更快地被重新处理。


Storm需要占用内存来跟踪每个元组，所以每个被处理的元组都必须被确认。因为如果不对每个元组进行确认，任务最终会耗光可用的内存。

做聚合或者合并操作的Bolt可能会延迟确认一个元组，直到根据一堆元组计算出了一个结果后，才会确认。聚合或者合并操作的Bolt，通常也会对他们的输出元组进行多锚定。


一个Storm拓扑有一组特殊的”acker”任务，它们负责跟踪由每个Spout元组触发的消息的处理状态。当一个”acker”看到一个Spout元组产生的有向无环图中的消息被完全处理，就通知当初创建这个Spout元组的Spout任务，这个元组被成功处理。可以通过拓扑配置项Config.TOPOLOGY_ACKER_EXECUTORS来设置一个拓扑中acker任务executor的数量。


假设”D”元组和”E”元组是基于“C”元组产生的，那么下图描述了确认“C”元组成功处理后，元组树的变化。图中虚线框表示的元组代表已经在消息树上被删除了：

![image](https://note.youdao.com/yws/api/personal/file/4F6F3541D36244C181EB20E170F9AEA9?method=download&shareKey=9b766e0b06cb195588162670274a4d10)

由于在“C”从消息树中删除(通过acker函数确认成功处理)的同时，“D”和“E”也被添加到(通过emit函数来锚定的)元组树中，所以这棵树从来不会被提早处理完。


![image](https://note.youdao.com/yws/api/personal/file/C1FD3C170A274409876E17780D2DCA61?method=download&shareKey=c54b4e51f221381dd3a376dac8c03cf1)


### 对比


![image](https://note.youdao.com/yws/api/personal/file/369928E45EDF4C0AB7500302AE0B2303?method=download&shareKey=e98f93e47bdbb75dbbfd16eec9bd2b55)

左边的图展示的是Storm的Ack机制。Spout每发送一条数据到Bolt，就会产生一条ack的信息给acker，当Bolt处理完这条数据后也会发送ack信息给acker。当acker收到这条数据的所有ack信息时，会回复Spout一条ack信息。也就是说，对于一个只有两级（spout+bolt）的拓扑来说，每发送一条数据，就会传输3条ack信息。这3条ack信息则是为了保证可靠性所需要的开销。


右边的图展示的是Flink的Checkpoint机制。Flink中Checkpoint信息的发起者是JobManager。它不像Storm中那样，每条信息都会有ack信息的开销，而且按时间来计算花销。用户可以设置做checkpoint的频率，比如10秒钟做一次checkpoint。每做一次checkpoint，花销只有从Source发往map的1条checkpoint信息（JobManager发出来的checkpoint信息走的是控制流，与数据流无关）。与storm相比，Flink的可靠性机制开销要低得多。这也就是为什么保证可靠性对Flink的性能影响较小，而storm的影响确很大的原因。



### Reference

https://www.jianshu.com/p/8d6569361999

https://zhoukaibo.com/2019/01/10/flink-kafka-exactly-once/