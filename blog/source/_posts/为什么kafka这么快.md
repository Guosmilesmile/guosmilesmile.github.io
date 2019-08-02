---
title: 为什么kafka这么快
date: 2019-07-09 21:51:29
tags:
categories:
	- Kafka
---


###  批量处理

生产者聚合了一批消息，然后再做2次rpc将消息存入broker（ack）

### 客户端优化

采用了双线程：主线程和Sender线程。主线程负责将消息置入客户端缓存，Sender线程负责从缓存中发送消息，而这个缓存会聚合多个消息为一个批次

### 日志格式

https://guosmilesmile.github.io/2019/07/10/Kafka%E6%B6%88%E6%81%AF%E6%A0%BC%E5%BC%8F%E7%9A%84%E6%BC%94%E5%8F%98/


### 消息压缩

Kafka支持多种消息压缩方式（gzip、snappy、lz4）。对消息进行压缩可以极大地减少网络传输 量、降低网络 I/O，从而提高整体的性能。消息压缩是一种使用时间换空间的优化方式，如果对 时延有一定的要求，则不推荐对消息进行压缩。


### 建立索引，方便快速定位查询
每个日志分段文件对应了两个索引文件，主要用来提高查找消息的效率，这也是提升性能的一种方式。

### 分区

增加分区数可以增加并行度增加吞吐量。


### 顺序IO

操作系统可以针对线性读写做深层次的优化，比如预读(read-ahead，提前将一个比较大的磁盘块读入内存) 和后写(write-behind，将很多小的逻辑写操作合并起来组成一个大的物理写操作)技术


### 零拷贝

Zero Copy技术提升了消费的效率。

https://guosmilesmile.github.io/2019/03/19/kafka-zero-copy/



### 页缓存


页缓存是操作系统实现的一种主要的磁盘缓存，以此用来减少对磁盘 I/O 的操作。

* 读
当一个进程准备读取磁盘上的文件内容时，操作系统会先查看待读取的数据所在的页 (page)是否在页缓存(pagecache)中，如果存在(命中)则直接返回数据，从而避免了对物 理磁盘的 I/O 操作;如果没有命中，则操作系统会向磁盘发起读取请求并将读取的数据页存入 页缓存，之后再将数据返回给进程。
* 写
如果一个进程需要将数据写入磁盘，那么操作系统也会检测数据对应的页是否在页缓存中，如果不存在，则会先在页缓存中添加相应的页，最后将数据写入对应的页。被修改过后的页也就变成了脏页，操作系统会在合适的时间把脏页中的 数据写入磁盘，以保持数据的一致性。


Kafka 中大量使用了页缓存，这是 Kafka 实现高吞吐的重要因素之一。消息都是先被写入页缓存，然后由操作系统负责具体的刷盘任务的。

如果Kafka写入到mmap之后就立即flush然后再返回Producer叫同步(sync)；写入mmap之后立即返回Producer不调用flush叫异步(async)。

1、kafka一开始是把数据写到PageCache，也就是缓存，如果消费者一直在消费，而且速度大于等于kafka的生产者发送数据的速度，那么消费者会一直从PageCache读取数据，速度等同于内存的操作，不会因为kafka写入磁盘的操作影响吞吐量。

2、当kafka的消费者消费速度不及生产者生产速度时，PageCache存的数据已经是最新的数据了，kafka消费端需要的数据已经被存储磁盘中，这时，kafka的消费速度会受到磁盘的读取速度影响。


##### Mmap（Memory Mapped Files，内存映射文件）

Mmap 方法为我们提供了将文件的部分或全部映射到内存地址空间的能力，同当这块内存区域被写入数据之后[dirty]，操作系统会用一定的算法把这些数据写入到文件中

```java
(1)RandomAccessFile raf = new RandomAccessFile (File, "rw");

(2)FileChannel channel = raf.getChannel();

(3)MappedByteBuffer buff = channel.map(FileChannel.MapMode.READ_WRITE,startAddr,SIZE);

(4)buf.put((byte)255);

(5)buf.write(byte[] data)
```

其中最重要的就是那个buff，它是文件在内存中映射的标的物，通过对buff的read/write我们就可以间接实现对于文件的读写操作，当然写操作是操作系统帮忙完成的。


##### Tip
1. Kafka官方并不建议通过Broker端的log.flush.interval.messages和log.flush.interval.ms来强制写盘，认为数据的可靠性应该通过Replica来保证，而强制Flush数据到磁盘会对整体性能产生影响。
2. 可以通过调整/proc/sys/vm/dirty_background_ratio和/proc/sys/vm/dirty_ratio来调优性能。
* 脏页率超过第一个指标会启动pdflush开始Flush Dirty PageCache。
* 脏页率超过第二个指标会阻塞所有的写操作来进行Flush。



### Reference
https://www.2cto.com/kf/201705/638958.html

https://mp.weixin.qq.com/s/G5nfLpPOr80pk1sHzrLuOA

https://www.xuebuyuan.com/1691696.html