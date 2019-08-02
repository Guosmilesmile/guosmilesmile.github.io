---
title: Kafka消息格式的演变
date: 2019-07-10 22:07:06
tags:
categories:
	- Kafka
---

### copy by 
https://mp.weixin.qq.com/s?__biz=MzU0MzQ5MDA0Mw==&mid=2247483983&idx=1&sn=1c2bd11df195f84e5433512f6b2695e8&chksm=fb0be8dbcc7c61cda63c599769b38a78c08e51ab8a2943985040014dcbcc1cbe1547d3f70ee6&scene=21#wechat_redirect

### 写入

Kafka根据topic（主题）对消息进行分类，发布到Kafka集群的每条消息都需要指定一个topic，每个topic将被分为多个partition（分区）。每个partition在存储层面是追加log（日志）文件，任何发布到此partition的消息都会被追加到log文件的尾部，每条消息在文件中的位置称为offset（偏移量），offset为一个long型的数值，它唯一标记一条消息。

![image](https://note.youdao.com/yws/api/personal/file/1BF0C4B8582F4B8C923F48CDF99840CF?method=download&shareKey=bb97192483d1bb0febc683c8b29f055f)

### v0版本

对于Kafka消息格式的第一个版本，我们把它称之为v0，在Kafka 0.10.0版本之前都是采用的这个消息格式。注意如无特殊说明，我们只讨论消息未压缩的情形。


![image](https://note.youdao.com/yws/api/personal/file/58719E4085AB450EAF5A250417BE4C1B?method=download&shareKey=7215663a4582e51e9404c2731c53ee34)


上左图中的“RECORD”部分就是v0版本的消息格式，大多数人会把左图中的整体，即包括offset和message size字段都都看成是消息，因为每个Record（v0和v1版）必定对应一个offset和message size。每条消息都一个offset用来标志它在partition中的偏移量，这个offset是逻辑值，而非实际物理偏移值，message size表示消息的大小，这两者的一起被称之为日志头部（LOG_OVERHEAD），固定为12B。LOG_OVERHEAD和RECORD一起用来描述一条消息。与消息对应的还有消息集的概念，消息集中包含一条或者多条消息，消息集不仅是存储于磁盘以及在网络上传输（Produce & Fetch）的基本形式，而且是kafka中压缩的基本单元，详细结构参考上右图。


下面来具体陈述一下消息（Record）格式中的各个字段，从crc32开始算起，各个字段的解释如下：

* crc32（4B）：crc32校验值。校验范围为magic至value之间。
* magic（1B）：消息格式版本号，此版本的magic值为0。
* attributes（1B）：消息的属性。总共占1个字节，低3位表示压缩类型：0表示NONE、1表示GZIP、2表示SNAPPY、3表示LZ4（LZ4自Kafka 0.9.x引入），其余位保留。
* key length（4B）：表示消息的key的长度。如果为-1，则表示没有设置key，即key=null。
* key：可选，如果没有key则无此字段。
* value length（4B）：实际消息体的长度。如果为-1，则表示消息为空。
* value：消息体。可以为空，比如tomnstone消息。


v0版本中一个消息的最小长度（RECORD_OVERHEAD_V0）为crc32 + magic + attributes + key length + value length = 4B + 1B + 1B + 4B + 4B =14B，也就是说v0版本中一条消息的最小长度为14B，如果小于这个值，那么这就是一条破损的消息而不被接受。


### v1版本


kafka从0.10.0版本开始到0.11.0版本之前所使用的消息格式版本为v1，其比v0版本就多了一个timestamp字段，表示消息的时间戳。v1版本的消息结构图如下所示：


![image](https://note.youdao.com/yws/api/personal/file/D328529302E142B3A9E154F15D8C5006?method=download&shareKey=55527cdcb69fc9c3cf5295e9e36e94a6)


v1版本的magic字段值为1。v1版本的attributes字段中的低3位和v0版本的一样，还是表示压缩类型，而第4个bit也被利用了起来：0表示timestamp类型为CreateTime，而1表示tImestamp类型为LogAppendTime，其他位保留。v1版本的最小消息（RECORD_OVERHEAD_V1）大小要比v0版本的要大8个字节，即22B。如果像v0版本介绍的一样发送一条key="key"，value="value"的消息，那么此条消息在v1版本中会占用42B.


#### 消息压缩

常见的压缩算法是数据量越大压缩效果越好，一条消息通常不会太大，这就导致压缩效果并不太好。而kafka实现的压缩方式是将多条消息一起进行压缩，这样可以保证较好的压缩效果。而且在一般情况下，生产者发送的压缩数据在kafka broker中也是保持压缩状态进行存储，消费者从服务端获取也是压缩的消息，消费者在处理消息之前才会解压消息，这样保持了端到端的压缩


讲解到这里都是针对消息未压缩的情况，而当消息压缩时是将整个消息集进行压缩而作为内层消息（inner message），内层消息整体作为外层（wrapper message）的value，其结构图如下所示

![image](https://note.youdao.com/yws/api/personal/file/165E0753D27F41B197A08388361CF5BB?method=download&shareKey=dcdf6d34368790d90f50da470d4aca96)


压缩后的外层消息（wrapper message）中的key为null，所以图右部分没有画出key这一部分。当生产者创建压缩消息的时候，对内部压缩消息设置的offset是从0开始为每个内部消息分配offset，详细可以参考下图右部：

![image](https://note.youdao.com/yws/api/personal/file/7B7C2CDF9CA44C81A254B34A2A954334?method=download&shareKey=ce64f83c64ebb83d08e1828aa489c2cd)


其实每个从生产者发出的消息集中的消息offset都是从0开始的，当然这个offset不能直接存储在日志文件中，对offset进行转换时在服务端进行的，客户端不需要做这个工作。外层消息保存了内层消息中最后一条消息的绝对位移（absolute offset），绝对位移是指相对于整个partition而言的。参考上图，对于未压缩的情形，图右内层消息最后一条的offset理应是1030，但是被压缩之后就变成了5，而这个1030被赋予给了外层的offset。


### v2版本

kafka从0.11.0版本开始所使用的消息格式版本为v2，这个版本的消息相比于v0和v1的版本而言改动很大，同时还参考了Protocol Buffer而引入了变长整型（Varints）和ZigZag编码。Varints是使用一个或多个字节来序列化整数的一种方法，数值越小，其所占用的字节数就越少。ZigZag编码以一种锯齿形（zig-zags）的方式来回穿梭于正负整数之间，以使得带符号整数映射为无符号整数，这样可以使得绝对值较小的负数仍然享有较小的Varints编码值，比如-1编码为1,1编码为2，-2编码为3。详细可以参考：https://developers.google.com/protocol-buffers/docs/encoding。


回顾一下kafka v0和v1版本的消息格式，如果消息本身没有key，那么key length字段为-1，int类型的需要4个字节来保存，而如果采用Varints来编码则只需要一个字节。根据Varints的规则可以推导出0-63之间的数字占1个字节，64-8191之间的数字占2个字节，8192-1048575之间的数字占3个字节。而kafka broker的配置message.max.bytes的默认大小为1000012（Varints编码占3个字节），如果消息格式中与长度有关的字段采用Varints的编码的话，绝大多数情况下都会节省空间，而v2版本的消息格式也正是这样做的。不过需要注意的是Varints并非一直会省空间，一个int32最长会占用5个字节（大于默认的4字节），一个int64最长会占用10字节（大于默认的8字节）。


v2版本中消息集谓之为Record Batch，而不是先前的Message Set了，其内部也包含了一条或者多条消息，消息的格式参见下图中部和右部。在消息压缩的情形下，Record Batch Header部分（参见下图左部，从first offset到records count字段）是不被压缩的，而被压缩的是records字段中的所有内容。

![image](https://note.youdao.com/yws/api/personal/file/5EB75C8D06C24F768C91C351B748A91E?method=download&shareKey=16e13e5ebd125965005dde3f25692789)

先来讲述一下消息格式Record的关键字段，可以看到内部字段大量采用了Varints，这样Kafka可以根据具体的值来确定需要几个字节来保存。v2版本的消息格式去掉了crc字段，另外增加了length（消息总长度）、timestamp delta（时间戳增量）、offset delta（位移增量）和headers信息，并且attributes被弃用了，笔者对此做如下分析（对于key、key length、value、value length字段和v0以及v1版本的一样，这里不再赘述）：

* length：消息总长度。
* attributes：弃用，但是还是在消息格式中占据1B的大小，以备未来的格式扩展。
* timestamp delta：时间戳增量。通常一个timestamp需要占用8个字节，如果像这里保存与RecordBatch的其实时间戳的差值的话可以进一步的节省占用的字节数。
* offset delta：位移增量。保存与RecordBatch起始位移的差值，可以节省占用的字节数。
* headers：这个字段用来支持应用级别的扩展，而不需要像v0和v1版本一样不得不将一些应用级别的属性值嵌入在消息体里面。Header的格式如上图最有，包含key和value，一个Record里面可以包含0至多个Header。具体可以参考以下KIP-82。


如果对于v1版本的消息，如果用户指定的timestamp类型是LogAppendTime而不是CreateTime，那么消息从发送端（Producer）进入broker端之后timestamp字段会被更新，那么此时消息的crc值将会被重新计算，而此值在Producer端已经被计算过一次；再者，broker端在进行消息格式转换时（比如v1版转成v0版的消息格式）也会重新计算crc的值。在这些类似的情况下，消息从发送端到消费端（Consumer）之间流动时，crc的值是变动的，需要计算两次crc的值，所以这个字段的设计在v0和v1版本中显得比较鸡肋。在v2版本中将crc的字段从Record中转移到了RecordBatch中。


v2版本对于消息集（RecordBatch）做了彻底的修改，参考上图左部，除了刚刚提及的crc字段，还多了如下字段：

* first offset：表示当前RecordBatch的起始位移。
* length：计算partition leader epoch到headers之间的长度。
* partition leader epoch：用来确保数据可靠性，详细可以参考KIP-101
* magic：消息格式的版本号，对于v2版本而言，magic等于2。
* attributes：消息属性，注意这里占用了两个字节。低3位表示压缩格式，可以参考v0和v1；第4位表示时间戳类型；第5位表示此RecordBatch是否处于事务中，0表示非事务，1表示事务。第6位表示是否是Control消息，0表示非Control消息，而1表示是Control消息，Control消息用来支持事务功能。
* last offset delta：RecordBatch中最后一个Record的offset与first offset的差值。主要被broker用来确认RecordBatch中Records的组装正确性。
* first timestamp：RecordBatch中第一条Record的时间戳。
* max timestamp：RecordBatch中最大的时间戳，一般情况下是指最后一个Record的时间戳，和last offset delta的作用一样，用来确保消息组装的正确性。
* producer id：用来支持幂等性，详细可以参考KIP-98。
* producer epoch：和producer id一样，用来支持幂等性。
* first sequence：和producer id、producer epoch一样，用来支持幂等性。
* records count：RecordBatch中Record的个数。

```
[root@node1 kafka_2.12-1.0.0]# bin/kafka-run-class.sh kafka.tools.DumpLogSegments 
--files /tmp/kafka-logs/msg_format_v2-0/00000000000000000000.log --print-data-log
Dumping /tmp/kafka-logs/msg_format_v2-0/00000000000000000000.log
Starting offset: 0
baseOffset: 0 lastOffset: 0 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 0 isTransactional: false position: 0 CreateTime: 1524709879130 isvalid: true size: 76 magic: 2 
compresscodec: NONE crc: 2857248333
```

可以看到size字段为76，我们根据上图中的v2版本的日志格式来验证一下，Record Batch Header部分共61B。Record部分中attributes占1B；timestamp delta值为0，占1B；offset delta值为0，占1B；key length值为3，占1B，key占3B；value length值为5，占1B，value占5B；headers count值为0，占1B, 无headers。Record部分的总长度 = 1B + 1B + 1B + 1B + 3B + 1B + 5B + 1B = 14B，所以Record的length字段值为14，编码为变长整型占1B。最后推到出这条消息的占用字节数= 61B + 14B + 1B = 76B，符合测试结果。同样再发一条key=null，value="value"的消息的话，可以计算出这条消息占73B。

这么看上去好像v2版本的消息比之前版本的消息占用空间要大很多，的确对于单条消息而言是这样的，如果我们连续往msg_format_v2中再发送10条value长度为6,key为null的消息，可以得到：

```
baseOffset: 2 lastOffset: 11 baseSequence: -1 lastSequence: -1 
producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 0 
isTransactional: false position: 149 CreateTime: 1524712213771 
isvalid: true size: 191 magic: 2 compresscodec: NONE crc: 820363253
```

本来应该占用740B大小的空间，实际上只占用了191B，如果在v0版本中这10条消息则需要占用320B的空间，v1版本则需要占用400B的空间，这样看来v2版本又节省了很多的空间，因为其将多个消息（Record）打包存放到单个RecordBatch中，又通过Varints编码极大的节省了空间。

就以v1和v2版本对比而立，至于哪个消息格式占用空间大是不确定的，要根据具体情况具体分析。比如每条消息的大小为16KB，那么一个消息集中只能包含有一条消息（参数batch.size默认大小为16384），所以v1版本的消息集大小为12B + 22B + 16384B = 16418B。而对于v2版本而言，其消息集大小为61B + 11B + 16384B = 17086B（length值为16384+，占用3B，value length值为16384，占用大小为3B，其余数值型的字段都可以只占用1B的空间）。可以看到v1版本又会比v2版本节省些许空间。