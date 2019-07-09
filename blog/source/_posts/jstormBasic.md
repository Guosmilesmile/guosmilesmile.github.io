---
title: JSTORM 基础概念
date: 2019-03-17 10:55:08
tags:
categories:
	- Jstorm
---

### JSTORM

## 基础概念

#### 拓扑(Topologies)
一个Storm拓扑打包了一个实时处理程序的逻辑。一个Storm拓扑跟一个MapReduce的任务(job)是类似的。主要区别是MapReduce任务最终会结束，而拓扑会一直运行（当然直到你杀死它)。一个拓扑是一个通过流分组(stream grouping)把Spout和Bolt连接到一起的拓扑结构。图的每条边代表一个Bolt订阅了其他Spout或者Bolt的输出流。一个拓扑就是一个复杂的多阶段的流计算。

<!--more--> 

#### 元组(Tuple)
元组是Storm提供的一个轻量级的数据格式，可以用来包装你需要实际处理的数据。元组是一次消息传递的基本单元。


#### Spouts

Spout(喷嘴，这个名字很形象)是Storm中流的来源。也可以称为source。


Spout可以一次给多个流吐数据。此时需要通过OutputFieldsDeclarer的declareStream函数来声明多个流并在调用SpoutOutputCollector提供的emit方法时指定元组吐给哪个流。
```
        @Override
        public void declareOutputFields(OutputFieldsDeclarer declarer) {
                declarer.declare(new Fields("data", "chan_id", "stat_time"));
                declarer.declareStream("fliterData", new Fields("topic", "msg"));
        }
        
```
在declare中，需要指定下发的流的名称，才可以将数据区分开来，如果填写的是null，那么会用默认名称default。读取那个stream，在topology定义的时候指定。
通过context下发到下游
```
this.collector.emit(streamName,new Values(JSONObject.toJSONString(entry.getValue()), entry.getValue().getChannelUrl(), entry.getValue().getStatisticTime()));
```

Spout中最主要的函数是nextTuple，Storm框架会不断调用它去做元组的轮询。如果没有新的元组过来，就直接返回，否则把新元组吐到拓扑里。nextTuple必须是非阻塞的，因为Storm在同一个线程里执行Spout的函数。

#### Bolts

在拓扑中所有的计算逻辑都是在Bolt中实现的。Bolt就是流水线上的一个处理单元，把数据的计算处理过程合理的拆分到多个Bolt、合理设置Bolt的task数量，能够提高Bolt的处理能力，提升流水线的并发度。

##### 订阅数据流
当你声明了一个Bolt的输入流，也就订阅了另外一个组件的某个特定的输出流。
```
//redBolt是上游bolt的名称，declarer.shuffleGrouping(componentId,streamId)
//订阅了redBolt组件上的默认流
declarer.shuffleGrouping("redBolt")
//订阅指定流
declarer.shuffleGrouping("redBolt", DEFAULT_STREAM_ID)
```

在Bolt中最主要的函数是execute函数，它使用一个新的元组当作输入。Bolt使用OutputCollector对象来吐出新的元组.
```
必须注意OutputCollector不是线程安全的，所以所有的吐数据(emit)、确认(ack)、通知失败(fail)必须发生在同一个线程里。
```


#### 任务(Tasks)
每个Spout和Bolt会以多个任务(Task)的形式在集群上运行。每个任务对应一个执行线程，流分组定义了如何从一组任务(同一个Bolt)发送元组到另外一组任务(另外一个Bolt)上。

#### 组件(Component)

组件(component)是对Bolt和Spout的统称

#### 流分组(Stream Grouping)

定义拓扑的时候，一部分工作是指定每个Bolt应该消费哪些流。流分组定义了一个流在一个消费它的Bolt内的多个任务(task)之间如何分组。流分组跟计算机网络中的路由功能是类似的，决定了每个元组在拓扑中的处理路线。

在Storm中有七个内置的流分组策略，你也可以通过实现CustomStreamGrouping接口来自定义一个流分组策略:

* 洗牌分组(Shuffle grouping): 随机分配元组到Bolt的某个任务上，这样保证同一个Bolt的每个任务都能够得到相同数量的元组。
* 字段分组(Fields grouping): 按照指定的分组字段来进行流的分组。例如，流是用字段“user-id"来分组的，那有着相同“user-id"的元组就会分到同一个任务里，但是有不同“user-id"的元组就会分到不同的任务里。这是一种非常重要的分组方式，通过这种流分组方式，我们就可以做到让Storm产出的消息在这个"user-id"级别是严格有序的，这对一些对时序敏感的应用(例如，计费系统)是非常重要的。
* **Partial Key grouping**(只在storm有，jstorm不存在): 跟字段分组一样，流也是用指定的分组字段进行分组的，但是在多个下游Bolt之间是有负载均衡的，这样当输入数据有倾斜时可以更好的利用资源
* All grouping: 流会复制给Bolt的所有任务。小心使用这种分组方式。在拓扑中，如果希望某类元祖发送到所有的下游消费者，就可以使用这种All grouping的流分组策略。
* **Global grouping**: 整个流会分配给Bolt的一个任务。会分配给有最小ID的任务。
* 不分组(None grouping): 说明不关心流是如何分组的。目前，None grouping等价于洗牌分组。
* Direct grouping：一种特殊的分组。对于这样分组的流，元组的生产者决定消费者的哪个任务会接收处理这个元组。只能在声明做直连的流(direct streams)上声明Direct 
* groupings分组方式。只能通过使用emitDirect系列函数来吐元组给直连流。一个Bolt可以通过提供的TopologyContext来获得消费者的任务ID，也可以通过OutputCollector对象的emit函数(会返回元组被发送到的任务的ID)来跟踪消费者的任务ID。在ack的实现中，Spout有两个直连输入流，ack和ackFail，使用了这种直连分组的方式。
Local or shuffle grouping：如果目标Bolt在同一个worker进程里有一个或多个任务，元组就会通过洗牌的方式分配到这些同一个进程内的任务里。否则，就跟普通的洗牌分组一样。这种方式的好处是可以提高拓扑的处理效率，因为worker内部通信就是进程内部通信了，相比拓扑间的进程间通信要高效的多。worker进程间通信是通过使用Netty来进行网络通信的。（但是会出现压在同一个worker下，出现性能问题）

#### Workers(工作进程)
拓扑以一个或多个Worker进程的方式运行。每个Worker进程是一个物理的Java虚拟机，执行拓扑的一部分任务。例如，如果拓扑的并发设置成了300，分配了50个Worker，那么每个Worker执行6个任务(作为Worker内部的线程）。Storm会尽量把所有的任务均分到所有的Worker上。

#### 拓扑的组成部分

在Storm集群上运行的拓扑主要包含以下的三个实体：

* Worker进程
* Executors
* Tasks(任务)          
![image](https://note.youdao.com/yws/api/personal/file/813ECBCBE33B4AAD925DC47F7E63A803?method=download&shareKey=4797a8cc32bfb3eabe45cb1a918cacad)

一个worker进程属于一个特定的拓扑并且执行这个拓扑的一个或多个component（spout或者bolt)的一个或多个executor。一个worker进程就是一个Java虚拟机(JVM)，它执行一个拓扑的一个子集。

一个executor是由一个worker进程产生的一个线程，它运行在worker的Java虚拟机里。一个executor为同一个component(spout或bolt)运行一个或多个任务。一个executor总会有一个线程来运行executor所有的task,这说明task在executor内部是串行执行的。

![image](https://note.youdao.com/yws/api/personal/file/82B6A306A6714191B8F6D3E1BF1ED7EC?method=download&shareKey=197482437549a59830a0346bec60d533)

用途 | 描述 |  配置项 | 如何通过代码设置(例子)
---|---|---|---|
worker进程数量 | 拓扑在集群机器上运行时需要的worker进程数据量 | Config#TOPOLOGY_WORKERS| Config#setNumWorkers
每个组件需要创建的executor数量 | executor线程的数量 |没有单独配置项|TopologyBuilder#setSpout() 和 TopologyBuidler#setBolt() Storm 0.8之后使用 parallelism_hint参数来指定executor的初始数量
task数量(在jstorm中该参数被废弃，默认就是一个excutor对应一个task.JStorm认为Executor的存在收益比太低，虽然它支持不停机动态扩大Task的数量，但同时增加了理解成本，增加了应用开发人员编程的复杂度，所以JStorm中去掉了Executor) | 每个组件需要创建的task数量|Config#TOPOLOGY_TASKS|ComponentConfigurationDeclarer#setNumTasks()

下面是一段如何实际设置这些配置的示例性代码：
```
topologyBuilder.setBolt("green-bolt", new GreenBolt(), 2)
.setNumTasks(4)
.shuffleGrouping("blue-spout");
```
在上述代码中我们配置Storm以两个executor和4个task的初始数量去运行greenBolt。Storm会在每个executor（线程）上运行两个任务。如果你没有显示配置任务的数量，Storm会默认每个executor运行一个任务。


#### 内部通信
Storm中，Worker之间使用Netty进行网络通信。

在Storm拓扑的一个Worker进程内部，多个Task之间也会进行通信。比如上图二中的Task 6和Task 3。Storm中Worker进程内部的消息通信依赖于LMAX Disruptor这个高性能线程间通信的消息通信库。

JStorm与Storm在内部消息传递机制上的主要差别：
JStorm中独立出一个线程来专门负责消息的反序列化，这样执行线程单独执行，而不是Storm那样，一个线程负责执行反序列化并执行用户的逻辑。相当于是把流水线拆解的更小了。这样对于反序列化时延跟执行时延在同一个数量级的应用性能提升比较明显。

![image](https://note.youdao.com/yws/api/personal/file/595E3C2C67684906ACAE35FDB708A049?method=download&shareKey=6bf190219c0d3c16bf1b5fb0b83407bb)


##### 详细解释

每个Worker进程有一个NettyServer，它监听在Worker的TCP端口上，其他需要跟它通信的Worker会作为NettyClient分别建立连接。当NettyServer接收到消息，会根据taskId参数把消息放到对应的反序列化队列(DeserializedQueue)里面。 

topology.executor.receive.buffer.size决定了反序列化队列的大小。TaskReceiver中的反序列化线程专门负责消费反序列化队列中的消息：先将消息反序列化，然后放到执行队列(Execute Queue)中去。

执行队列的消费者是BoltExecutor线程，它负责从队列中取出消息，执行用户的代码逻辑。执行完用户的代码逻辑后，最终通过OutputCollect输出消息，此时消息里已经生成了目标task的taskId。topology.executor.receive.buffer.size决定了执行队列的大小。可以看到JStorm中执行队列跟反序列化队列的大小是同一个配置项，即它们是一致的。

#### 如何配置Storm的内部消息缓存
上面提到的众多配置项都在conf/defaults.yaml里有定义。可以通过在Storm集群的conf/storm.yaml里进行配置来全局的覆值。也可以通过Storm的Java API backtype.storm.Config 来对单个的Storm拓扑进行配置。（http://nathanmarz.github.io/storm/doc/backtype/storm/Config.html）
TOPOLOGY_EXECUTOR_RECEIVE_BUFFER_SIZE =topology.executor.receive.buffer.size



#### 消息可靠性

##### 消息树
![image](https://note.youdao.com/yws/api/personal/file/50CED120A8D848FBA99FDEDEB7038546?method=download&shareKey=86148fc758742fb66dc035c5f81f6e37)

##### 一个消息被完整处理是什么意思？

**指一个从Spout发出的元组所触发的消息树中所有的消息都被Storm处理了。如果在指定的超时时间里，这个Spout元组触发的消息树中有任何一个消息没有处理完，就认为这个Spout元组处理失败了。这个超时时间是通过每个拓扑的Config.TOPOLOGY_MESSAGE_TIMEOUT_SECS配置项来进行配置的，默认是30秒**


在前面消息树的例子里，只有消息树中所有的消息(包含一条Spout消息，六条split Bolt消息,六条count Bolt消息)都被Storm处理完了，才算是这条Spout消息被完整处理了。

##### 元组创建时通知Storm

在Storm消息树(元组树)中添加一个子结点的操作叫做锚定(anchoring)。在应用程序发送一个新元组时候，Storm会在幕后做锚定。

```
//锚定
_collector.emit(tuple, new Values(word));
//不锚定
_collector.emit(new Values(word));
```
每个单词元组是通过把输入的元组作为emit函数中的第一个参数来做锚定的。通过锚定，Storm就能够得到元组之间的关联关系(输入元组触发了新的元组)，继而构建出Spout元组触发的整个消息树。所以当下游处理失败时，就可以通知Spout当前消息树根节点的Spout元组处理失败，让Spout重新处理。相反，如果在emit的时候没有指定输入的元组，叫做不锚定。

这样发射单词元组，会导致这个元组不被锚定(unanchored)，这样Storm就不能得到这个元组的消息树，继而不能跟踪消息树是否被完整处理。这样下游处理失败，不能通知到上游的Spout任务。不同的应用的有不同的容错处理方式，有时候需要这样不锚定的场景。

一个输出的元组可以被锚定到多个输入元组上，叫做多锚定(multi-anchoring)。这在做流的合并或者聚合的时候非常有用。一个多锚定的元组处理失败，会导致Spout上重新处理对应的多个输入元组。多锚定是通过指定一个多个输入元组的列表而不是单个元组来完成的。

```
List<Tuple> anchors = new ArrayList<Tuple>();
anchors.add(tuple1);  
anchors.add(tuple2);
_collector.emit(anchors, new Values(word));
```

##### 元组处理完后通知Storm

锚定的作用就是指定元组树的结构--下一步是当元组树中某个元组已经处理完成时，通知Storm。通知是通过OutputCollector中的ack和fail函数来完成的。例如上面流式计算单词个数例子中的split Bolt的实现SplitSentence类，可以看到句子被切分成单词后，当所有的单词元组都被发射后，会确认(ack)输入的元组处理完成。

可以利用OutputCollector的fail函数来立即通知Storm，当前消息树的根元组处理失败了。例如，应用程序可能捕捉到了数据库客户端的一个异常，就显示地通知Storm输入元组处理失败。通过显示地通知Storm元组处理失败，这个Spout元组就不用等待超时而能更快地被重新处理。

Storm需要占用内存来跟踪每个元组，所以每个被处理的元组都必须被确认。因为如果不对每个元组进行确认，任务最终会耗光可用的内存。

做聚合或者合并操作的Bolt可能会延迟确认一个元组，直到根据一堆元组计算出了一个结果后，才会确认。聚合或者合并操作的Bolt，通常也会对他们的输出元组进行多锚定。

##### acker任务
一个Storm拓扑有一组特殊的"acker"任务，它们负责跟踪由每个Spout元组触发的消息的处理状态。当一个"acker"看到一个Spout元组产生的有向无环图中的消息被完全处理，就通知当初创建这个Spout元组的Spout任务，这个元组被成功处理。可以通过拓扑配置项Config.TOPOLOGY_ACKER_EXECUTORS来设置一个拓扑中acker任务executor的数量。Storm默认TOPOLOGY_ACKER_EXECUTORS和拓扑中配置的Worker的数量相同--对于需要处理大量消息的拓扑来说，需要增大acker executor的数量。


##### 元组的生命周期

理解Storm的可靠性实现方式的最好方法是查看元组的生命周期和元组构成的有向无环图。当拓扑的Spout或者Bolt中创建一个元组时，都会被赋予一个随机的64比特的标识(message id)。acker任务使用这些id来跟踪每个Spout元组产生的有向无环图的处理状态。在Bolt中产生一个新的元组时，会从锚定的一个或多个输入元组中拷贝所有Spout元组的message-id，所以每个元组都携带了自己所在元组树的根节点Spout元组的message-id。当确认一个元组处理成功了，Storm就会给对应的acker任务发送特定的消息--通知acker当前这个Spout元组产生的消息树中某个消息处理完了，而且这个特定消息在消息树中又产生了一个新消息(新消息锚定的输入是这个特定的消息)。


个例子，假设"D"元组和"E"元组是基于“C”元组产生的，那么下图描述了确认“C”元组成功处理后，元组树的变化。图中虚线框表示的元组代表已经在消息树上被删除了：
![image](https://note.youdao.com/yws/api/personal/file/4F6F3541D36244C181EB20E170F9AEA9?method=download&shareKey=9b766e0b06cb195588162670274a4d10)

由于在“C”从消息树中删除(通过acker函数确认成功处理)的同时，“D”和“E”也被添加到(通过emit函数来锚定的)元组树中，所以这棵树从来不会被提早处理完。

正如上面已经提到的，在一个拓扑中，可以有任意数量的acker任务。这导致了如下的两个问题:
1. 当拓扑中的一个元组确认被处理完，或者产生一个新的元组时，Storm应该通知哪个acker任务？
2. 通知了acker任务后，acker任务如何通知到对应的Spout任务？

Storm采用对元组中携带的Spout元组message-id哈希取模的方法来把一个元组映射到一个acker任务上(所以同一个消息树里的所有消息都会映射到同一个acker任务)。因为每个元组携带了自己所处的元组树中根节点Spout元组(可能有多个)的标识，所以Storm就能决定通知哪个acker任务。




## Mess（杂项）

Config.TOPOLOGY_MAX_SPOUT_PENDING    

* 同时活跃的batch数量，你必须设置同时处理的batch数量。你可以通过”topology.max.spout.pending” 来指定， 如果你不指定，默认是1。
* topology.max.spout.pending 的意义在于 ，缓存spout 发送出去的tuple，当下流的bolt还有topology.max.spout.pending 个 tuple 没有消费完时，spout会停下来，等待下游bolt去消费，当tuple 的个数少于topology.max.spout.pending个数时，spout 会继续从消息源读取消息。（这个属性只对可靠消息处理有用）

##### 参数调整
* storm.messaging.netty.server_worker_threads：为接收消息线程；
* storm.messaging.netty.client_worker_threads：发送消息线程的数量；
* netty.transfer.batch.size：是指每次Netty Client 向Netty Server 发送的数据的大小，
如果需要发送的Tuple 消息大于netty.transfer.batch.size ， 则Tuple 消息会按照netty.transfer.batch.size 进行切分，然后多次发送。
* storm.messaging.netty.buffer_size：为每次批量发送的Tuple 序列化之后的Task
Message 消息的大小。
* storm.messaging.netty.flush.check.interval.ms：表示当有TaskMessage 需要发送的时候， Netty Client 检查可以发送数据的频率。
降低storm.messaging.netty.flush.check.interval.ms 的值， 可以提高时效性。增加netty.transfer.batch.size 和storm.messaging.netty.buffer_size 的值，可以提升网络传输的吐吞量，使得网络的有效载荷提升（减少TCP 包的数量，并且TCP 包中的有效数据量增加），通常时效性就会降低一些。因此需要根据自身的业务情况，合理在吞吐量和时效性直接的平衡。

#### Partial key gourping
* storm的PartialKeyGrouping是解决fieldsGrouping造成的bolt节点skewed load的问题
* fieldsGrouping采取的是对所选字段进行哈希然后与taskId数量向下取模来选择taskId的下标
* PartialKeyGrouping在1.2.2版本的实现是使用guava提供的Hashing.murmur3_128哈希函数计算哈希值，然后取绝对值与taskId数量取余数得到两个可选的taskId下标；在2.0.0版本则使用key的哈希值作为seed，采用Random函数来计算两个taskId的下标。注意这里返回两个值供bolt做负载均衡选择，这是与fieldsGrouping的差别。在得到两个候选taskId之后，PartialKeyGrouping额外维护了taskId的使用数，每次选择使用少的，与此同时也更新每次选择的计数。
* 值得注意的是在wordCount的bolt使用PartialKeyGrouping，同一个单词不再固定发给相同的task，因此这里还需要RollingCountAggBolt按fieldsGrouping进行合并。


[storm的PartialKeyGrouping](https://www.jianshu.com/p/e054505e5253)   
