---
title: Flink 通信机制和背压处理
date: 2019-07-28 20:39:43
tags:
categories:
	- Flink
---


![image](https://note.youdao.com/yws/api/personal/file/56744FB7B89448F2A58698D5D6151587?method=download&shareKey=c4dddaf0898abdac28d4123cd0a4d33c)

### 本地线程数据传递(同一个TM)

以Operator FlatMap 所在线程 与 下游 Operator sum() 所在线程间的通信为例。这两个task线程共享同一个Buffer pool,通过wait()/notifyAll来同步。 Buffer和Netty中的ByteBuf功能类似，可以看作是一块共享的内存。inputGate负责读取Buffer或Event。

1. 当没有Buffer可以消费时，Operator sum()所在的线程阻塞(通过inputGate中的inputChannelWithData.wait()方法阻塞)[对应(1)]
2. 当FlatMap所在线程写入结果数据到ResultSubPartition,并flush到buffer后[(2)(3)]
3. 会唤醒Operator sum()所在的线程（通过inputChannelWithData.notifyAll()方法唤醒）。[(4)]
4. 线程被唤醒后会从Buffer中读取数据，经反序列化后，传递给Operator中的用户代码逻辑处理。[(5)]
交互过程如下图所示:

![image](https://note.youdao.com/yws/api/personal/file/DEC307A65DCB4008BF401E5C0EDEA534?method=download&shareKey=2e30476f270cad20ad2cedb16a00a4c3)



### 远程线程数据传递(不同TM)

远程线程的Operator数据传递与本地线程类似。不同点在于，当没有Buffer可以消费时，会通过PartitionRequestClient向Operator FlatMap所在的进程发起RPC请求。远程的PartitionRequestServerHandler接收到请求后，读取ResultPartition管理的Buffer。并返回给Client。

![image](https://note.youdao.com/yws/api/personal/file/07BDD946C3E14C7DAD55C7438696A1E9?method=download&shareKey=c040cef54a82640144867cea05aa76bd)

RPC通信基于Netty实现， 下图为Client端的RPC请求发送过程。PartitionRequestClient发出请求，交由Netty写到对应的socket。Netty读取Socket数据，解析Response后交由NetworkClientHandler处理。

![image](https://note.youdao.com/yws/api/personal/file/9C06B4E014704BF38E300533102F74DB?method=download&shareKey=2d36bc2d0e28e14631b70d00d3b5e91b)

### 同一线程的Operator数据传递(同一个Task)

![image](https://note.youdao.com/yws/api/personal/file/D74FFBA07746402CA2672A42DB3FF925?method=download&shareKey=d730d08090d92224cf968549b0d84837)

 这两个Operator在同一个线程中运行，数据不需要经过序列化和写到多线程共享的buffer中， Operator sum()通过Collector发送数据后，直接调用Operator sink的processElement方法传递数据。
 
 ### 物理传输
 
A.1→B.3、A.1→B.4 以及 A.2→B.3 和 A.2→B.4 的情况，如下图所示：


![image](https://note.youdao.com/yws/api/personal/file/6F5D2BE011BD493485719BF40FC8F5D4?method=download&shareKey=990790ecc03e7c10d3f5ef92bffda6b3)

每个子任务的结果称为结果分区，每个结果拆分到单独的子结果分区（ResultSubpartitions）中——每个逻辑通道有一个。Flink 不再处理单个记录，而是将一组序列化记录组装到网络缓冲区中。每个子任务可用于其自身的本地缓冲池中的缓冲区数量（每次发送方和接收方各一个）上限符合下列规则：
```
channels * buffers-per-channel + floating-buffers-per-gate
```

#### 造成背压场景

* 每当子任务的发送缓冲池耗尽时——也就是缓存驻留在结果子分区的缓存队列中或更底层的基于 Netty 的网络栈中时——生产者就被阻塞了，无法继续工作，并承受背压。
* 接收器也是类似：较底层网络栈中传入的 Netty 缓存需要通过网络缓冲区提供给 Flink。如果相应子任务的缓冲池中没有可用的网络缓存，Flink 将在缓存可用前停止从该通道读取。这将对这部分多路传输链路发送的所有子任务造成背压，因此也限制了其他接收子任务

下图中子任务 B.4 过载了，它会对这条多路传输链路造成背压，还会阻止子任务 B.3 接收和处理新的缓存。

![image](https://note.youdao.com/yws/api/personal/file/C80951F0E65C4A219E951FE70386A8E6?method=download&shareKey=4fa4619890593c2886675e34caad4da4)

为了防止这种情况发生，Flink 1.5 引入了自己的流量控制机制。

### 基于信用的流量控制

基于网络缓冲区的可用性实现.每个远程输入通道现在都有自己的一组独占缓冲区，而非使用共享的本地缓冲池。而本地缓冲池中的缓存称为浮动缓存，因为它们会浮动并可用于所有输入通道。

![image](https://note.youdao.com/yws/api/personal/file/E4AD271F5E604DD084A295BCD6428E29?method=download&shareKey=a8256273bfaac69aa25737442367db5c)

接收器将缓存的可用性声明为发送方的信用（1 缓存 = 1 信用）。每个结果子分区将跟踪其通道信用值。每个结果子分区将跟踪其通道信用值。如果信用可用，则缓存仅转发到较底层的网络栈，并且发送的每个缓存都会让信用值减去一。



图中有两个地方和两个参数对应。

* Exclusive buffers：对应taskmanager.network.memory.buffers-per-channel。default为2，每个channel需要的独占buffer，一定要大于或者等于2.1个buffer用于接收数据，一个buffer用于序列化数据。
* buffer pool中的Floating buffers的个数：taskmanager.network.memory.floating-buffers-per-gate，default为8.在一个subtask中，会为每个下游task建立一个channel，每个channel中需要独占taskmanager.network.memory.buffers-per-channel个buffer。浮动缓冲区是基于backlog(子分区中的实时输出缓冲区)反馈分布的，可以帮助缓解由于子分区间数据分布不平衡而造成的反压力。接收器将使用它来请求适当数量的浮动缓冲区，以便更快处理 backlog。它将尝试获取与 backlog 大小一样多的浮动缓冲区，但有时并不会如意，可能只获取一点甚至获取不到缓冲。在节点和/或集群中机器数量较多的情况下，这个值应该增加，特别是在数据倾斜比较严重的时候。



#### 背压处理

相比没有流量控制的接收器的背压机制，信用机制提供了更直接的控制逻辑：如果接收器能力不足，其可用信用将减到 0，并阻止发送方将缓存转发到较底层的网络栈上。这样只在这个逻辑信道上存在背压，并且不需要阻止从多路复用 TCP 信道读取内容。因此，其他接收器在处理可用缓存时就不受影响了。


![image](https://note.youdao.com/yws/api/personal/file/20A3A8689F2C433DA8B553D8D3EA5BE4?method=download&shareKey=0518c3a6cadf0478371daf1dff69d17b)


个人感觉，这是一个消费的拉的模型。



https://www.jianshu.com/p/5748df8428f9


https://www.jianshu.com/p/146370ac61c9

https://flink.apache.org/2019/06/05/flink-network-stack.html


https://blog.csdn.net/huaishu/article/details/93723889

http://network.51cto.com/art/201906/598525.htm