---
title: Flink 使用调优(二)
date: 2019-07-27 11:09:01
tags:
categories:
	- Flink
---
这次调优的场景是处于批处理和yarn模式。


### 背景

获取位于hdfs的两个数据的多份文件，数据是按照5分钟或者一小时归档成一个文件夹。并且每个文件夹内部有多个文件，分析一天的数据，又碎又散。


两份数据需要先union后再join。

### 调优

#### taskmanager.network.memory

数据需要继续union和join，数据会出现shuffle，占用大量的networkBuffers，这部分内存是由taskmanager.network.memory提供，如果这部分内存配置不够，不管是在流式还是批处理，都会导致数据吞吐上不去，计算缓慢。

![image](https://note.youdao.com/yws/api/personal/file/E4AD271F5E604DD084A295BCD6428E29?method=download&shareKey=a8256273bfaac69aa25737442367db5c)


##### task数据传输

######  本地传输

如果 Task 1 和 Task 2 运行在同一个 worker 节点（TaskManager），该 buffer 可以直接交给下一个 Task。一旦 Task 2 消费了该 buffer，则该 buffer 会被缓冲池1回收。如果 Task 2 的速度比 1 慢，那么 buffer 回收的速度就会赶不上 Task 1 取 buffer 的速度，导致缓冲池1无可用的 buffer，Task 1 等待在可用的 buffer 上。最终形成 Task 1 的降速。

###### 远程传输

如果 Task 1 和 Task 2 运行在不同的 worker 节点上，那么 buffer 会在发送到网络（TCP Channel）后被回收。在接收端，会从 LocalBufferPool 中申请 buffer，然后拷贝网络中的数据到 buffer 中。如果没有可用的 buffer，会停止从 TCP 连接中读取数据。在输出端，通过 Netty 的水位值机制来保证不往网络中写入太多数据。如果网络中的数据（Netty输出缓冲中的字节数）超过了高水位值，我们会等到其降到低水位值以下才继续写入数据。这保证了网络中不会有太多的数据。如果接收端停止消费网络中的数据（由于接收端缓冲池没有可用 buffer），网络中的缓冲数据就会堆积，那么发送端也会暂停发送。另外，这会使得发送端的缓冲池得不到回收，writer 阻塞在向 LocalBufferPool 请求 buffer，阻塞了 writer 往 ResultSubPartition 写数据。

关于通信机制可以参考这篇文章

https://guosmilesmile.github.io/2019/07/28/Flink-%E9%80%9A%E4%BF%A1%E6%9C%BA%E5%88%B6%E5%92%8C%E8%83%8C%E5%8E%8B%E5%A4%84%E7%90%86/

#### 并行度设置

env.readfile()后多个dataset进行union在进行join。

由于flink读取hdfs，会通过inputsplit读取文件，如果文件太多还切割，就会导致一天的文件有100个，设置30个并行度，机会出现3000个source的并行度，导致数据大量的出现shuffle，导致taskmanager.network.memory 不够使用甚至影响性能。

详细关于flink读取hdfs的方法和inputsplit可以参考如下内容。

https://guosmilesmile.github.io/2019/06/06/Flink-%E8%AF%BB%E5%8F%96HDFS%E4%B8%AD%E7%9A%84%E6%95%B0%E6%8D%AE%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%E8%AF%BB%E5%8F%96/

https://guosmilesmile.github.io/2019/06/11/HDFS-File-Block-%E5%92%8C-Input-Split/


读取细碎的小文件，可以将并行度设置为1或者适当调大一点，减少网络传输。




#### yarnSession创建


如果需要20个并行度，创建一个yarnSession，但是yarnSession上是创建一个TM，上面有20个slot

```
/opt/flink/1.8-SNAPSHOT/bin/yarn-session.sh -nm unhealthCgLive -s 20 -jm 1024  -tm 40960 -qu r-data_caculate-QAW -nl realtime-flink_qaw   -D taskmanager.memory.preallocate=true -D taskmanager.memory.off-heap=true -D taskmanager.network.memory.fraction=0.4  -d

```

这样会导致这20个slot位于同一台机器上，这台机器跑高。


正确做法是将创建一个yarnSession，一个TM上有5个slot，如果启动的时候需要20个slot，就会创建4个TM，分布在4台机器上。

```

/opt/flink/1.8-SNAPSHOT/bin/yarn-session.sh -nm unhealthCgLive -s 5 -jm 1024  -tm 10240 -qu r-data_caculate-QAW -nl realtime-flink_qaw   -D taskmanager.memory.preallocate=true -D taskmanager.memory.off-heap=true -D taskmanager.network.memory.fraction=0.4  -d
```


### 选择Join类型

当Flink处理批量数据，集群上的每个节点都会拥有部分数据，针对两份数据进行join，有如下策略可以选择：

* Repartition-repartition strategy：在这种情况下，两个数据集都按其key分区并通过网络发送。 这意味着如果数据集很大，则可能需要花费大量时间在网络上复制它们。

![image](https://note.youdao.com/yws/api/personal/file/146D46BDA62442E2B098763DF7C806CC?method=download&shareKey=bef270d28e951e5c42bd817c573117ef)
* Broadcast-forward strategy：在这种情况下，一个数据集保持不变，但第二个数据集将复制到集群中具有第一个数据集的一部分的每台机器上。

![image](https://note.youdao.com/yws/api/personal/file/43D8EDC8BEA846EAAA0E67C5F50986A6?method=download&shareKey=831fbda14e3a4b30a74de9b969af7cb6)


有如下四个具体的选择：


* OPTIMIZER_CHOOSES：相当于不提供任何提示，将选择留给系统。

*  BROADCAST_HASH_FIRST：广播第一个输入并从中构建哈希表，由第二个输入探测。如果第一个输入非常小，这是一个很好的策略。

 * BROADCAST_HASH_SECOND：广播第二个输入并从中构建一个哈希表，由第一个输入探测。如果第二个输入非常小，这是一个好策略。

* REPARTITION_HASH_FIRST：系统对每个输入进行分区（shuffle）（除非输入已经分区）并从第一个输入构建哈希表。如果第一个输入小于第二个输入，则此策略很好，但两个输入仍然很大。
    注意：如果不能进行大小估算，并且不能重新使用预先存在的分区和排序顺序，则这是系统使用的默认回退策略。

*   REPARTITION_HASH_SECOND：系统对每个输入进行分区（shuffle）（除非输入已经被分区）并从第二个输入构建哈希表。如果第二个输入小于第一个输入，则此策略很好，但两个输入仍然很大。

 * REPARTITION_SORT_MERGE：系统对每个输入进行分区（shuffle）（除非输入已经被分区）并对每个输入进行排序（除非它已经排序）。输入通过已排序输入的流 合并来连接。如果已经对一个或两个输入进行了排序，则此策略很好。


![image](https://note.youdao.com/yws/api/personal/file/34C2D63058D24917836D11496692F157?method=download&shareKey=ab37b2cac8bf86dcb35732c20c0d846d)



### Referenece
https://flink.apache.org/news/2015/03/13/peeking-into-Apache-Flinks-Engine-Room.html    