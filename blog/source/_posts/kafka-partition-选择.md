---
title: kafka partition 数量选择
date: 2019-03-23 21:10:33
tags:
categories: kafka
---
### 越多的分区可以提供更高的吞吐量
首先我们需要明白以下事实：在kafka中，单个patition是kafka并行操作的最小单元。在producer和broker端，向每一个分区写入数据是可以完全并行化的，此时，可以通过加大硬件资源的利用率来提升系统的吞吐量，例如对数据进行压缩。在consumer段，kafka只允许单个partition的数据被一个consumer线程消费。因此，在consumer端，每一个Consumer Group内部的consumer并行度完全依赖于被消费的分区数量。综上所述，通常情况下，在一个Kafka集群中，partition的数量越多，意味着可以到达的吞吐量越大。

　　我们可以粗略地通过吞吐量来计算kafka集群的分区数量。假设对于单个partition，producer端的可达吞吐量为p，Consumer端的可达吞吐量为c，期望的目标吞吐量为t，那么集群所需要的partition数量至少为max(t/p,t/c)。在producer端，单个分区的吞吐量大小会受到批量大小、数据压缩方法、 确认类型（同步/异步）、复制因子等配置参数的影响。经过测试，在producer端，单个partition的吞吐量通常是在10MB/s左右。在consumer端，单个partition的吞吐量依赖于consumer端每个消息的应用逻辑处理速度。因此，我们需要对consumer端的吞吐量进行测量。
　　
### 越多的分区需要打开更多地文件句柄
在kafka的broker中，每个分区都会对照着文件系统的一个目录。在kafka的数据日志文件目录中，每个日志数据段都会分配两个文件，一个索引文件和一个数据文件。当前版本的kafka，每个broker会为每个日志段文件打开一个index文件句柄和一个数据文件句柄。因此，随着partition的增多，需要底层操作系统配置更高的文件句柄数量限制。

### 越多的partition意味着需要客户端需要更多的内存

如果partition的数量增加，消息将会在producer端按更多的partition进行积累。众多的partition所消耗的内存汇集起来，有可能会超过设置的内容大小限制。当这种情况发生时，producer必须通过消息堵塞或者丢失一些新消息的方式解决上述问题，但是这两种做法都不理想。为了避免这种情况发生，我们必须重新将produder的内存设置得更大一些。

### 不要设置太多的partition
分区是Kafka可伸缩性的关键，但这并不意味着应该有太多的分区，topic中的每个分区都使用大量RAM(文件描述符)。随着分区的增加，CPU上的负载也会增加，因为Kafka需要跟踪所有分区。如果在broker多的情况下，一个topic分配50个partition是一个比较合理的数值

### 在cpu核心和消费者之间保持良好的平衡
Kafka中的每个分区都是单线程的，如果服务器上的核心数量很低，那么太多的分区将无法充分发挥其潜力。因此，您需要在核心和消费者之间保持良好的平衡。