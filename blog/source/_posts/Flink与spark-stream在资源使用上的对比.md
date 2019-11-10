---
title: Flink与spark stream在资源使用上的对比
date: 2019-11-08 20:53:14
tags:
categories:
	- Flink
---

### Spark Streaming 迁移到Flink的效果小结

小米的业务从Spark Streaming迁移到Flink的过程，比如数据处理的延迟、资源使用的变化、作业的稳定性等。

* 对于无状态作业，数据处理的延迟由之前Spark Streaming的16129ms降低到Flink的926ms，有94.2%的显著提升（有状态作业也有提升，但是和具体业务逻辑有关，不做介绍）；
* 对后端存储系统的写入延迟从80ms降低到了20ms左右，如下图（这是因为Spark Streaming的mini batch模式会在batch最后有批量写存储系统的操作，从而造成写请求尖峰，Flink则没有类似问题）:
![image](https://note.youdao.com/yws/api/personal/file/DACB961CED1E428E81C1BBF94461C198?method=download&shareKey=b169692ed9fbf90c12abbb05eb7888dd)

* 对于简单的从消息队列Talos到存储系统HDFS的数据清洗作业（ETL），由之前Spark Streaming的占用210个CPU Core降到了Flink的32个CPU Core，资源利用率提高了84.8%；

其中前两点优化效果是比较容易理解的，主要是第三点我们觉得有点超出预期。为了验证这一点，信息流推荐的同学帮助我们做了一些测试，尝试把之前的Spark Streaming作业由210个CPU Core降低到64个，但是测试结果是作业出现了数据拥堵。这个Spark Streaming测试作业的batch interval 是10s，大部分batch能够在8s左右运行完，偶尔抖动的话会有十几秒，但是当晚高峰流量上涨之后，这个Spark Streaming作业就会开始拥堵了，而Flink使用32个CPU Core却没有遇到拥堵问题。


很显然，更低的资源占用帮助业务更好的节省了成本，节省出来的计算资源则可以让更多其他的业务使用；为了让节省成本能够得到“理论”上的支撑，我们尝试从几个方面研究并对比了Spark Streaming和Flink的一些区别。

### 调度计算VS调度数据

对于任何一个分布式计算框架而言，如果“数据”和“计算”不在同一个节点上，那么它们中必须有一个需要移动到另一个所在的节点。如果把计算调度到数据所在的节点，那就是“调度计算”，反之则是“调度数据”；在这一点上Spark Streaming和Flink的实现是不同的。


Spark在调度该分片的计算的时候，会尽量把该分片的计算调度到数据所在的节点，从而提高计算效率。


”调度计算”的方法在批处理中有很大的优势，因为“计算”相比于“数据”来讲一般信息量比较小，如果“计算”可以在“数据”所在的节点执行的话，会省去大量网络传输，节省带宽的同时提高了计算效率。但是在流式计算中，以Spark Streaming的调度方法为例，由于需要频繁的调度”计算“，则会有一些效率上的损耗。


首先，每次”计算“的调度都是要消耗一些时间的，比如“计算”信息的序列化 → 传输 → 反序列化 → 初始化相关资源 → 计算执行→执行完的清理和结果上报等，这些都是一些“损耗”。

另外，用户的计算中一般会有一些资源的初始化逻辑，比如初始化外部系统的客户端（类似于Kafka Producer或Consumer)；每次计算的重复调度容易导致这些资源的重复初始化，需要用户对执行逻辑有一定的理解，才能合理地初始化资源，避免资源的重复创建；这就提高了使用门槛，容易埋下隐患；通过业务支持发现，在实际生产过程中，经常会遇到大并发的Spark Streaming作业给Kafka或HBase等存储系统带来巨大连接压力的情况，就是因为用户在计算逻辑中一直重复创建连接。


Spark在官方文档提供了一些避免重复创建网络连接的示例代码，其核心思想就是通过连接池来复用连接：

```
rdd.foreachPartition { partitionOfRecords =>
// ConnectionPool is a static, lazily initialized pool of connections
    val connection = ConnectionPool.getConnection()
    partitionOfRecords.foreach(record => connection.send(record))
    ConnectionPool.returnConnection(connection)  // return to the pool for future reuse
  }
}
```
需要指出的是，即使用户代码层面合理的使用了连接池，由于同一个“计算”逻辑不一定调度到同一个计算节点，还是可能会出现在不同计算节点上重新创建连接的情况。


Flink和Storm类似，都是通过“调度数据”来完成计算的，也就是“计算逻辑”初始化并启动后，如果没有异常会一直执行，源源不断地消费上游的数据，处理后发送到下游；有点像工厂里的流水线，货物在传送带上一直传递，每个工人专注完成自己的处理逻辑即可。


虽然“调度数据”和“调度计算”有各自的优势，但是在流式计算的实际生产场景中，“调度计算”很可能“有力使不出来”；比如一般流式计算都是消费消息队列Kafka或Talos的数据进行处理，而实际生产环境中为了保证消息队列的低延迟和易维护，一般不会和计算节点（比如Yarn服务的节点）混布，而是有各自的机器（不过很多时候是在同一机房）；所以无论是Spark还是Flink，都无法避免消息队列数据的跨网络传输。所以从实际使用体验上讲，Flink的调度数据模式，显然更容易减少损耗，提高计算效率，同时在使用上更符合用户“直觉”，不易出现重复创建资源的情况。(在批处理上，reduce不管在哪个框架都会需要出现shuffle的情况，但是在spark中可以现在本地计算中提前reduce，再到下游reduce，性能上也会比较好。如果出现source的数据有两种处理方式，那么数据需要下发两次，而在spark中，还是在本地计算。这就是为什么spark中复用数据为什么要用persist。  )

#### 举例

如下作业图，dataSource的数据经过两次不一样的转换分支成两份数据，在Flink中，调度数据的模式，source的数据会下发两份，在网络中流转，内存溢出的部分需要在io在读写，在几个T的数据上，性能是不行的。

在spark中，调度计算的模式，source的数据的两种转换模式，还是在本地计算，后续的各种转换也在本地计算，只有在reduce的时候才会出现网络shuffle，在海量数据的批处理中，性能上会比Flink好。dataSource需要被计算两遍，如果想要减少被计算的次数，就需要使用到persist功能。




![image](https://note.youdao.com/yws/api/personal/file/0F303EFBB041469688AE82B3078698E1?method=download&shareKey=3363a1f0c5ebce41c9c7e3c3b095de88)




不过这里不得不提的一点是，Spark Streaming的“调度计算”模式，对于处理计算系统中的“慢节点”或“异常节点”有天然的优势。比如如果Yarn集群中有一台节点磁盘存在异常，导致计算不停地失败，Spark可以通过blacklist机制停止调度计算到该节点，从而保证整个作业的稳定性。或者有一台计算节点的CPU Load偏高，导致处理比较慢，Spark也可以通过speculation机制及时把同一计算调度到其他节点，避免慢节点拖慢整个作业；而以上特性在Flink中都是缺失的。

### Mini batch vs streaming

![image](https://note.youdao.com/yws/api/personal/file/999B0F5019E74ED8A2557E1C6CE28EEF?method=download&shareKey=2eea4aaaefef8aedc656607922aa3160)


Spark Streaming并不是真正意义上的流式计算，而是从批处理衍生出来的mini batch计算。如图所示，Spark根据RDD依赖关系中的shuffle dependency进行作业的Stage划分，每个Stage根据RDD的partition信息切分成不同的分片；在实际执行的时候，只有当每个分片对应的计算结束之后，整个个Stage才算计算完成。

这种模式容易出现“长尾效应”，比如如果某个分片数据量偏大，那么其他分片也必须等这个分片计算完成后，才能进行下一轮的计算(Spark speculation对这种情况也没有好的作用，因为这个是由于分片数据不均匀导致的），这样既增加了其他分片的数据处理延迟，也浪费了资源。
 而Flink则是为真正的流式计算而设计的（并且把批处理抽象成有限流的数据计算），上游数据是持续发送到下游的，这样就避免了某个长尾分片导致其他分片计算“空闲”的情况，而是持续在处理数据，这在一定程度上提高了计算资源的利用率，降低了延迟。
 
 ![image](https://note.youdao.com/yws/api/personal/file/1A0C6A6AD15A4E3F9C0F2A5DD3D0617D?method=download&shareKey=661ac7697291b42e1874ff9d1922f5f3)
 
 当然，这里又要说一下mini batch的优点了，那就在异常恢复的时候，可以以比较低的代价把缺失的分片数据恢复过来，这个主要归功于RDD的依赖关系抽象；如上图所示，如果黑色块表示的数据丢失（比如节点异常），Spark仅需要通过重放“Good-Replay”表示的数据分片就可以把丢失的数据恢复，这个恢复效率是很高的。
 
 ![image](https://note.youdao.com/yws/api/personal/file/F8BB2E3C7212479487D374197CB7A083?method=download&shareKey=824d47947e2b55274b27cc2a194a146b)
 
 而Flink的话则需要停止整个“流水线”上的算子，并从Checkpoint恢复和重放数据；虽然Flink对这一点有一些优化，比如可以配置failover strategy为region来减少受影响的算子，不过相比于Spark只需要从上个Stage的数据恢复受影响的分片来讲，代价还是有点大。
总之，通过对比可以看出，Flink的streaming模式对于低延迟处理数据比较友好，Spark的mini batch模式则于异常恢复比较友好；如果在大部分情况下作业运行稳定的话，Flink在资源利用率和数据处理效率上确实更占优势一些。

### 数据序列化

简单来说，数据的序列化是指把一个object转化为byte stream，反序列化则相反。序列化主要用于对象的持久化或者网络传输。
常见的序列化格式有binary、json、xml、yaml等；常见的序列化框架有Java原生序列化、Kryo、Thrift、Protobuf、Avro等。

对于分布式计算来讲，数据的传输效率非常重要。好的序列化框架可以通过较低    的序列化时间和较低的内存占用大大提高计算效率和作业稳定性。在数据序列化上，Flink和Spark采用了不同的方式；Spark对于所有数据默认采用Java原生序列化方式，用户也可以配置使用Kryo；而Flink则是自己实现了一套高效率的序列化方法。

首先说一下Java原生的序列化方式，这种方式的好处是比较简单通用，只要对象实现了Serializable接口即可；缺点就是效率比较低，而且如果用户没有指定serialVersionUID的话，很容易出现作业重新编译后，之前的数据无法反序列化出来的情况（这也是Spark Streaming Checkpoint的一个痛点，在业务使用中经常出现修改了代码之后，无法从Checkpoint恢复的问题）；当然Java原生序列化还有一些其他弊端，这里不做深入讨论。


有意思的是，Flink官方文档里对于不要使用Java原生序列化强调了三遍，甚至网上有传言Oracle要抛弃Java原生序列化：

![image](https://note.youdao.com/yws/api/personal/file/342FD6BB32B34BCBB7529DA212377DD0?method=download&shareKey=3b257660d05fc407927839f8887415cd)


相比于Java原生序列化方式，无论是在序列化效率还是序列化结果的内存占用上，Kryo则更好一些（Spark声称一般Kryo会比Java原生节省10x内存占用）；Spark文档中表示它们之所以没有把Kryo设置为默认序列化框架的唯一原因是因为Kryo需要用户自己注册需要序列化的类，并且建议用户通过配置开启Kryo。


虽然如此，根据Flink的测试，Kryo依然比Flink自己实现的序列化方式效率要低一些；如图所示是Flink序列化器（PojoSerializer、RowSerializer、TupleSerializer）和Kryo等其他序列化框架的对比，可以看出Flink序列化器还是比较占优势的：

![image](https://note.youdao.com/yws/api/personal/file/8978E0BE31E54BCD874616C1EA0BF4F7?method=download&shareKey=a34ffc4ee18334d4f221e7fc2f236324)

在一个Flink作业DAG中，上游和下游之间传输的数据类型是固定且已知的，所以在序列化的时候只需要按照一定的排列规则把“值”信息写入即可（当然还有一些其他信息，比如是否为null）。

![image](https://note.youdao.com/yws/api/personal/file/4FE22225B71B474A834F13F69B7C789E?method=download&shareKey=54f98b410bf649be112f69db79b4afc7)

如图所示是一个内嵌POJO的Tuple3类型的序列化形式，可以看出这种序列化方式非常地“紧凑”，大大地节省了内存并提高了效率。另外，Flink自己实现的序列化方式还有一些其他优势，比如直接操作二进制数据等。


凡事都有两面性，自己实现序列化方式也是有一些劣势，比如状态数据的格式兼容性（State Schema Evolution）；如果你使用Flink自带的序列化框架序进行状态保存，那么修改状态数据的类信息后，可能在恢复状态时出现不兼容问题（目前Flink仅支持POJO和Avro的格式兼容升级）。


另外，用户为了保证数据能使用Flink自带的序列化器，有时候不得不自己再重写一个POJO类，把外部系统中数据的值再“映射”到这个POJO类中；而根据开发人员对POJO的理解不同，写出来的效果可能不一样，比如之前有个用户很肯定地说自己是按照POJO的规范来定义的类，我查看后发现原来他不小心多加了个logger，这从侧面说明还是有一定的用户使用门槛的。

```java
// Not a POJO demo.public
class Person {  
    
    private Logger logger = LoggerFactory.getLogger(Person.class);  
    
    public String name; 
    
    public int age;
    
}
```

### Reference 

https://mp.weixin.qq.com/s?__biz=MzUxMDQxMDMyNg==&mid=2247486025&idx=1&sn=db7349110955cba08aaee635e0dfaf8b&chksm=f9022170ce75a866e42dbc8bb10eb8e22169c6580717ce4aa14dc2a2810d5ce69416f0e99b37&mpshare=1&scene=1&srcid=1108vO9W7jEoR0xqvzgt9gGj&sharer_sharetime=1573174888778&sharer_shareid=797dbcdd3a4e624875c639b16a4ef5d9&key=ed2336ce379cc05e14c01f2b0d8c98ec59a01b6fe4ea08a8aac319f39023d25106e1e0c688ecef2ea0e10acd2f2900097b2060001c7381f8e2b503e7981050282ad55c7f735534baa04f939be43e4750&ascene=1&uin=MjU3NDYyMjA0Mw%3D%3D&devicetype=Windows+10&version=62070155&lang=zh_CN&pass_ticket=1ctJ%2BFuaUWk7VBXSrXBZ1YeSi1wfMVlUYZ6uM1zRdYiO%2B%2BVRPuo%2F7PNMBqlvcYlg