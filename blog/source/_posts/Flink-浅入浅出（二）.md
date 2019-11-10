---
title: Flink 浅入浅出（二）
date: 2019-11-10 09:46:16
tags:
categories:
	- Flink
---

## CheckPoint 

### 有状态与无状态介绍

* 无状态：数据的计算与上一次的计算结果无关。例如map,flatMap
* 有状态: 数据的计算与上一次的计算结果有关，例如时间窗口内的sum，需要累加求和。

举例

1. 无状态
* * 比如：我们只是进行一个字符串拼接，输入 a，输出 a_666,输入b，输出 b_666
输出的结果跟之前的状态没关系，符合幂等性。
幂等性：就是用户对于同一操作发起的一次请求或者多次请求的结果是一致的，不会因为多次点击而产生了副作用
2. 有状态
* * 计算pv、uv。输出的结果跟之前的状态有关系，不符合幂等性，访问多次，pv会增加

### Statebackend 的分类

下图阐释了目前 Flink 内置的三类 state backend，其中 MemoryStateBackend 和 FsStateBackend 在运行时都是存储在 java heap 中的，只有在执行 Checkpoint 时，FsStateBackend 才会将数据以文件格式持久化到远程存储上。而 RocksDBStateBackend 则借用了 RocksDB（内存磁盘混合的 LSM DB）对 state 进行存储。

![image](https://note.youdao.com/yws/api/personal/file/A48E29640AC647208A81F7928A829CF8?method=download&shareKey=ca789ce34002661163da43cb677da55f)


### Flink CheckPoint介绍    

CheckPoint介绍的存在就是为了解决flink任务failover掉之后，能够正常恢复任务。

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


### CheckPoint 调优
#### 超时原因

超时的原因会是什么呢？主要是一下两种:

* Barrier对齐
* 异步状态遍历和写hdfs


StreamTask收集到相应的inputChannel的barrier，收集齐之后就将barrier下发，并开始自己task的checkpoint逻辑，如果上下游是rescale或者forward的形式，下游只需要等待1个并发的barrier，因为是point-to-point的形式，如果是hash或者rebalance，下游的每一个task开始checkpoint的前提就是要收集齐上游所有并发的barrier。


####   相邻Checkpoint的间隔时间设置
在极大规模状态数据集下，应用每次的checkpoint时长都超过系统设定的最大时间（也就是checkpoint间隔时长），那么会发生什么样的事情

答案是应用会一直在做checkpoint，因为当应用发现它刚刚做完一次checkpoint后，又已经到了下次checkpoint的时间了，由于需要对齐barrier，因此在极限的情况下，会停止消费数据，但是checkpoint每隔一段时间又会不停的向下发送，使得会有一堆checkpoint往下发，导致用户程序无法运行。

#### Checkpoint的资源设置

当我们对越多的状态数据集做checkpoint时，需要消耗越多的资源。因为Flink在checkpoint时是首先在每个task上做数据checkpoint，然后在外部存储中做checkpoint持久化。在这里的一个优化思路是：在总状态数据固定的情况下，当每个task平均所checkpoint的数据越少，那么相应地checkpoint的总时间也会变短。所以我们可以为每个task设置更多的并行度（即分配更多的资源）来加速checkpoint的执行过程。

##### Checkpoint的task本地性恢复
为了快速的状态恢复，每个task会同时写checkpoint数据到本地磁盘和远程分布式存储，也就是说，这是一份双拷贝。只要task本地的checkpoint数据没有被破坏，系统在应用恢复时会首先加载本地的checkpoint数据，这样就大大减少了远程拉取状态数据的过程。此过程如下图所示：

![image](https://note.youdao.com/yws/api/personal/file/94984CC673A04EC288358F3316A09395?method=download&shareKey=36a3f508a92609bac47bebc409ac207d)

#### 外部State的存储选择

使用RocksDB来作为增量checkpoint的存储，并在其中不是持续增大，可以进行定期合并清楚历史状态。

### 增量checkpoint

Flink 增量式的检查点以“RocksDB”为基础，RocksDB是一个基于 LSM树的KV存储，新的数据保存在内存中，称为memtable。如果Key相同，后到的数据将覆盖之前的数据，一旦memtable写满了，RocksDB将数据压缩并写入到磁盘。memtable的数据持久化到磁盘后，他们就变成了不可变的sstable。

Flink跟踪前一个checkpoint创建和删除的RocksDB sstable文件，因为sstable是不可变的，Flink可以因此计算出 状态有哪些改变


如果使用增量式的checkpoint，那么在错误恢复的时候，不需要考虑很多的配置项。一旦发生了错误，Flink的JobManager会告诉 task需要从最新的checkpoint中恢复，它可以是全量的或者是增量的。之后TaskManager从分布式系统中下载checkpoint文件， 然后从中恢复状态。




#### 示例


![image](https://note.youdao.com/yws/api/personal/file/332DA16AB01049E1A2D9919621A37310?method=download&shareKey=d741767a5668ffe1eabb19b47a14ade8)

该例子中，子任务的操作是一个keyed-state，一个checkpoint文件保存周期是可配置的，本例中是2，配置方式state.checkpoints.num-retained,默认为1，用于指定保留的已完成的checkpoints个数。上面展示了每次checkpoint时RocksDB示例中存储的状态以及文件引用关系等。
* 对于checkpoint CP1，本地RocksDB目录包含两个磁盘文件（sstable），它基于checkpoint的name来创建目录。当完成checkpoint，将在共享注册表(shared state registry)中创建两个实体并将其count置为1.在共享注册表中存储的Key是由操作、子任务以及原始存储名称组成，同时注册表维护了一个Key到实际文件存储路径的Map。
* 对于checkpoint CP2，RocksDB已经创建了两个新的sstable文件，老的两个文件也存在。在CP2阶段，新的两个生成新文件，老的两个引用原来的存储。当checkpoint结束，所有引用文件的count加1。
* 对于checkpoint CP3，RocksDB的compaction将sstable-(1)，sstable-(2)以及sstable-(3)合并为sstable-(1,2,3)，同时删除了原始文件。合并后的文件包含原始文件的所有信息，并删除了重复的实体。除了该合并文件，sstable-(4)还存在，同时有一个sstable-(5)创建出来。Flink将新的sstable-(1,2,3)和sstable-(5)存储到底层，sstable-(4)引用CP2中的，并对相应引用次数count加1.老的CP1的checkpoint现在可以被删除，由于其retained已达到2，作为删除的一部分，Flink将所有CP1中的引用文件count减1.
* 对于checkpoint CP4，RocksDB合并sstable-(4)、sstable-(5)以及新的sstable-(6)成sstable-(4,5,6)。Flink将该新的sstable存储，并引用sstable-(1,2,3)，并将sstable-(1,2,3)的count加1，删除CP2中retained到2的。由于sstable-(1), sstable-(2), 和sstable-(3)降到了0，Flink将其从底层删除。

## 时间窗口


### 什么是 Window

在流处理应用中，数据是连续不断的，因此我们不可能等到所有数据都到了才开始处理。当然我们可以每来一个消息就处理一次，但是有时我们需要做一些聚合类的处理，例如：在过去的1分钟内有多少用户点击了我们的网页。在这种情况下，我们必须定义一个窗口，用来收集最近一分钟内的数据，并对这个窗口内的数据进行计算。

窗口可以是时间驱动的（Time Window，例如：每30秒钟），也可以是数据驱动的（Count Window，例如：每一百个元素）。一种经典的窗口分类可以分成：翻滚窗口（Tumbling Window，无重叠），滚动窗口（Sliding Window，有重叠），和会话窗口（Session Window，活动间隙）。

下图给出了几种经典的窗口切分概述图：

![image](https://note.youdao.com/yws/api/personal/file/E19B69CA6D074B10BC9139F87AE6B971?method=download&shareKey=a408ef8a652150e367ad1d15569b05ef)


### Time Window

Time Window 是根据时间对数据流进行分组的。Flink 提出了三种时间的概念，分别是event time（事件时间：事件发生时的时间），ingestion time（摄取时间：事件进入流处理系统的时间），processing time（处理时间：消息被计算处理的时间）。Flink 中窗口机制和时间类型是完全解耦的，也就是说当需要改变时间类型时不需要更改窗口逻辑相关的代码。


#### Tumbling Time Window

我们需要统计每一分钟中用户购买的商品的总数，需要将用户的行为事件按每一分钟进行切分，这种切分被成为翻滚时间窗口（Tumbling Time Window）。翻滚窗口能将数据流切分成不重叠的窗口，每一个事件只能属于一个窗口。通过使用 DataStream API，我们可以这样实现：

```
// Stream of (userId, buyCnt)
val buyCnts: DataStream[(Int, Int)] = ...

val tumblingCnts: DataStream[(Int, Int)] = buyCnts
  // key stream by userId
  .keyBy(0) 
  // tumbling time window of 1 minute length
  .timeWindow(Time.minutes(1))
  // compute sum over buyCnt
  .sum(1)

```

#### Sliding Time Window

但是对于某些应用，它们需要的窗口是不间断的，需要平滑地进行窗口聚合。比如，我们可以每30秒计算一次最近一分钟用户购买的商品总数。这种窗口我们称为滑动时间窗口（Sliding Time Window）。在滑窗中，一个元素可以对应多个窗口。通过使用 DataStream API，我们可以这样实现：


```
val slidingCnts: DataStream[(Int, Int)] = buyCnts
  .keyBy(0) 
  // sliding time window of 1 minute length and 30 secs trigger interval
  .timeWindow(Time.minutes(1), Time.seconds(30))
  .sum(1)
```


### Count Window

#### Tumbling Count Window

当我们想要每100个用户购买行为事件统计购买总数，那么每当窗口中填满100个元素了，就会对窗口进行计算，这种窗口我们称之为翻滚计数窗口（Tumbling Count Window），上图所示窗口大小为3个。通过使用 DataStream API，我们可以这样实现：


```
// Stream of (userId, buyCnts)
val buyCnts: DataStream[(Int, Int)] = ...

val tumblingCnts: DataStream[(Int, Int)] = buyCnts
  // key stream by sensorId
  .keyBy(0)
  // tumbling count window of 100 elements size
  .countWindow(100)
  // compute the buyCnt sum 
  .sum(1)
```

#### Sliding Count Window

当然Count Window 也支持 Sliding Window，虽在上图中未描述出来，但和Sliding Time Window含义是类似的，例如计算每10个元素计算一次最近100个元素的总和，代码示例如下。

```
val slidingCnts: DataStream[(Int, Int)] = vehicleCnts
  .keyBy(0)
  // sliding count window of 100 elements size and 10 elements trigger interval
  .countWindow(100, 10)
  .sum(1)
```

### 剖析 Window API


得益于 Flink Window API 松耦合设计，我们可以非常灵活地定义符合特定业务的窗口。Flink 中定义一个窗口主要需要以下三个组件。

#### Window Assigner

用来决定某个元素被分配到哪个/哪些窗口中去。

![image](https://note.youdao.com/yws/api/personal/file/63F95FC1519F4E1D9D28386A80548608?method=download&shareKey=f845c330a2d5325208d05fa79a9573c2)

#### Trigger

触发器。决定了一个窗口何时能够被计算或清除，每个窗口都会拥有一个自己的Trigger。
![image](https://note.youdao.com/yws/api/personal/file/51C3CF4968A74FC3BEA2614608D0F729?method=download&shareKey=f74e4cb0142c00ee91a6cb2da82e16c6)
#### Evictor

可以译为“驱逐者”。在Trigger触发之后，在窗口被处理之前，Evictor（如果有Evictor的话）会用来剔除窗口中不需要的元素，相当于一个filter。

![image](https://note.youdao.com/yws/api/personal/file/A5E033FF46FF45CF889BED0370894690?method=download&shareKey=5a17eaabad6fb2577097fc358aa336c3)

上述三个组件的不同实现的不同组合，可以定义出非常复杂的窗口。Flink 中内置的窗口也都是基于这三个组件构成的，当然内置窗口有时候无法解决用户特殊的需求，所以 Flink 也暴露了这些窗口机制的内部接口供用户实现自定义的窗口。下面我们将基于这三者探讨窗口的实现机制。

### Window 的实现

下图描述了 Flink 的窗口机制以及各组件之间是如何相互工作的。


![image](https://note.youdao.com/yws/api/personal/file/B0FA29A9684F4832A8F151499399A9C2?method=download&shareKey=61e62fb0f6f0720a0267d8c17eaca61e)

上图中的组件都位于一个算子（window operator）中，数据流源源不断地进入算子，每一个到达的元素都会被交给 WindowAssigner。WindowAssigner 会决定元素被放到哪个或哪些窗口（window），可能会创建新窗口。因为一个元素可以被放入多个窗口中，所以同时存在多个窗口是可能的。

注意，Window本身只是一个ID标识符，其内部可能存储了一些元数据，如TimeWindow中有开始和结束时间，但是并不会存储窗口中的元素。

```java
@PublicEvolving
public abstract class Window {

	/**
	 * Gets the largest timestamp that still belongs to this window.
	 *
	 * @return The largest timestamp that still belongs to this window.
	 */
	public abstract long maxTimestamp();
}

```

```java
@PublicEvolving
public class TimeWindow extends Window {

	private final long start;
	private final long end;

    和一堆窗口是否重叠，交叉，合并等方法
}

```

窗口中的元素实际存储在 Key/Value State 中，key为Window，value为元素集合（或聚合值）。为了保证窗口的容错性，该实现依赖了 Flink 的 State 机制.纯内存的MemoryStateBackend，依赖文件系统或者hdfs的FsStateBackend以及依赖第三方的RockDBStateBackend。

```java
	/** The state in which the window contents is stored. Each window is a namespace */
	private transient InternalAppendingState<K, W, IN, ACC, ACC> windowState;


    	windowState.setCurrentNamespace(window);
		windowState.add(element.getValue());
```


每一个窗口都拥有一个属于自己的 Trigger，Trigger上会有定时器，用来决定一个窗口何时能够被计算或清除。每当有元素加入到该窗口，或者之前注册的定时器超时了，那么Trigger都会被调用。Trigger的返回结果可以是 
* continue（不做任何操作）
* fire（处理窗口数据） 
* purge（移除窗口和窗口中的数据）
* fire + purge。     

一个Trigger的调用结果只是fire的话，那么会计算窗口并保留窗口原样，也就是说窗口中的数据仍然保留不变，等待下次Trigger fire的时候再次执行计算。一个窗口可以被重复计算多次知道它被 purge 了。在purge之前，窗口会一直占用着内存。

```
	            //将元素交给trigger去判断是否应该触发计算 
				TriggerResult triggerResult = triggerContext.onElement(element);

				if (triggerResult.isFire()) {
					ACC contents = windowState.get();
					if (contents == null) {
						continue;
					}
					//触发计算
					emitWindowContents(window, contents);
				}

				//是否清空窗口
				if (triggerResult.isPurge()) {
					windowState.clear();
				}
```

EventTimeTrigger.onElement

```java
	@Override
	public TriggerResult onElement(Object element, long timestamp, TimeWindow window, TriggerContext ctx) throws Exception {
		if (window.maxTimestamp() <= ctx.getCurrentWatermark()) {
			// if the watermark is already past the window fire immediately
			return TriggerResult.FIRE;
		} else {
			ctx.registerEventTimeTimer(window.maxTimestamp());
			return TriggerResult.CONTINUE;
		}
	}
```


窗口和时间进行注册，time=window.maxTimestamp()，注册的是窗口和这个窗口的最大时间
```java
internalTimerService.registerEventTimeTimer(window, time);
```


触发event窗口计算是在InternalTimerServiceImpl。根据source下发的watermark，它会移除timestamp小于等于指定time的eventTimerTimer，然后回调triggerTarget.onEventTime方法。而watermark是定时发送到，也不会出现没有数据就无法触发计算的情况。
```java
public void advanceWatermark(long time) throws Exception {
		currentWatermark = time;

		InternalTimer<K, N> timer;

		while ((timer = eventTimeTimersQueue.peek()) != null && timer.getTimestamp() <= time) {
			eventTimeTimersQueue.poll();
			keyContext.setCurrentKey(timer.getKey());
			triggerTarget.onEventTime(timer);
		}
	}
```

Flink的所有内置窗口分配程序(包括会话窗口)都负责在适当的时候清除它们的内容。清洗是作为一个单独的步骤完成的，而不是与触发相结合。



针对延迟的数据，如果有配置旁路，会直接从旁路返回给用户，如果没有配置则直接丢弃
```java
if (isSkippedElement && isElementLate(element)) {
			if (lateDataOutputTag != null){
				sideOutput(element);
			} else {
				this.numLateRecordsDropped.inc();
			}
		}
```

## Flink内存基础

### JVM存在的问题


##### Java对象开销（对象头的冗余信息）

相对于C/C++等更加接近底层的语言，Java对象的存储密度相对偏低，例如[1]，"abcd"这样简单的字符串在UTF-8编码中需要4个字节存储，但采用了UTF-16编码存储字符串的Java需要8个字节，同时Java对象还有header等其他额外信息，一个4字节字符串对象在Java中需要48字节的空间来存储。对于大部分的大数据应用，内存都是稀缺资源，更有效率的内存存储，意味着CPU数据访问吐吞量更高，以及更少磁盘落地的存在。

##### 对象存储结构引发的cache miss（内存的不连续性）

为了缓解CPU处理速度与内存访问速度的差距，现代CPU数据访问一般都会有多级缓存。当从内存加载数据到缓存时，一般是以cache line为单位加载数据，所以当CPU访问的数据如果是在内存中连续存储的话，访问的效率会非常高。

如果CPU要访问的数据不在当前缓存所有的cache line中，则需要从内存中加载对应的数据，这被称为一次cache miss。当cache miss非常高的时候，CPU大部分的时间都在等待数据加载，而不是真正的处理数据。Java对象并不是连续的存储在内存上，同时很多的Java数据结构的数据聚集性也不好。

##### 大数据的垃圾回收（对象的不复用）

Java的垃圾回收机制一直让Java开发者又爱又恨，一方面它免去了开发者自己回收资源的步骤，提高了开发效率，减少了内存泄漏的可能，另一方面垃圾回收也是Java应用的不定时炸弹，有时秒级甚至是分钟级的垃圾回收极大影响了Java应用的性能和可用性。在时下数据中心，大容量内存得到了广泛的应用，甚至出现了单台机器配置TB内存的情况，同时，大数据分析通常会遍历整个源数据集，对数据进行转换、清洗、处理等步骤。在这个过程中，会产生海量的Java对象，JVM的垃圾回收执行效率对性能有很大影响。通过JVM参数调优提高垃圾回收效率需要用户对应用和分布式计算框架以及JVM的各参数有深入了解，而且有时候这也远远不够。


### Flink的处理策略

#### 定制的序列化工具

显式内存管理的前提步骤就是序列化，将Java对象序列化成二进制数据存储在内存上（on heap或是off-heap）。通用的序列化框架，如Java默认使用java.io.Serializable将Java对象及其成员变量的所有元信息作为其序列化数据的一部分，序列化后的数据包含了所有反序列化所需的信息。这在某些场景中十分必要，但是对于Flink这样的分布式计算框架来说，这些元数据信息可能是冗余数据。


![image](https://note.youdao.com/yws/api/personal/file/4FE22225B71B474A834F13F69B7C789E?method=download&shareKey=54f98b410bf649be112f69db79b4afc7)


#### 显式的内存管理

一般通用的做法是批量申请和释放内存，每个JVM实例有一个统一的内存管理器，所有内存的申请和释放都通过该内存管理器进行。这可以避免常见的内存碎片问题，同时由于数据以二进制的方式存储，可以大大减轻垃圾回收压力。

Flink将内存分为3个部分，每个部分都有不同用途：

* Network buffers: 一些以32KB Byte数组为单位的buffer，主要被网络模块用于数据的网络传输。（这部分内存直接走堆外内存）
* Memory Manager pool大量以32KB Byte数组为单位的内存池，所有的运行时算法（例如Sort/Shuffle/Join）都从这个内存池申请内存，并将序列化后的数据存储其中，结束后释放回内存池。
* Remaining (Free) Heap主要留给UDF中用户自己创建的Java对象，由JVM管理。

![image](https://note.youdao.com/yws/api/personal/file/62ECB4950CA34D59890438085F1BCBDB?method=download&shareKey=0a75582d05b65b097282ad2214c7150d)


Flink自己实现了基于内存的序列化框架，里面维护着key和pointer的概念，它的key是连续存储，在cpu层面会做一些优化，cache miss概率极低。比较和排序的时候不需要比较真正的数据，先通过这个key比较，只有当它相等的时候，才会从内存中把这个数据反序列化出来，再去对比具体的数据，这是个不错的性能优化点。而且在Flink 1.9中，进行了进一步的优化，只序列化需要对比的字段，而不用序列化整个实体。

### Flink抽象出的内存类型
* HEAP：JVM堆内存
* OFF_HEAP：非堆内存

这在Flink中被定义为一个枚举类型：MemoryType。

```java
@Internal
public enum MemoryType {

	/**
	 * Denotes memory that is part of the Java heap.
	 */
	HEAP,

	/**
	 * Denotes memory that is outside the Java heap (but still part of tha Java process).
	 */
	OFF_HEAP
}
```

### MemorySegment

Flink所管理的内存被抽象为数据结构：MemorySegment。内存管理的最小模块。

HeapMemorySegment(弃用)和HybridMemorySegment是对MemorySegment的实现。


![image](https://note.youdao.com/yws/api/personal/file/19A35F484C1943729D5838E65CC5D1A2?method=download&shareKey=93d97caa3cdba310ff5608d651bda411)

这两个的差别在HybridMemorySegment包含HeapMemorySegment的功能，既可以对堆内进行处理，也可以对堆外进行处理，但对单个字节的操作效率稍差。



MemorySegment有两个构造函数,分别针对堆内内存和堆外内存。

堆内内存：
```java
MemorySegment(byte[] buffer, Object owner) {
		if (buffer == null) {
			throw new NullPointerException("buffer");
		}

		this.heapMemory = buffer;
		this.address = BYTE_ARRAY_BASE_OFFSET;// byte数组会占用一部分内存，起始位置不一样
		this.size = buffer.length;
		this.addressLimit = this.address + this.size;
		this.owner = owner;
	}
```

堆外内存：
```java
MemorySegment(long offHeapAddress, int size, Object owner) {
		if (offHeapAddress <= 0) {
			throw new IllegalArgumentException("negative pointer or size");
		}
		if (offHeapAddress >= Long.MAX_VALUE - Integer.MAX_VALUE) {
			// this is necessary to make sure the collapsed checks are safe against numeric overflows
			throw new IllegalArgumentException("Segment initialized with too large address: " + offHeapAddress
					+ " ; Max allowed address is " + (Long.MAX_VALUE - Integer.MAX_VALUE - 1));
		}

		this.heapMemory = null;
		this.address = offHeapAddress;
		this.addressLimit = this.address + size;
		this.size = size;
		this.owner = owner;
	}
```


![image](https://note.youdao.com/yws/api/personal/file/011D7C8167754FD6ABCE456E14027D6E?method=download&shareKey=e6ed309d42f4e524243d1198119d3ebc)


* UNSAFE : 用来对堆/非堆内存进行操作，是JVM的非安全的API
* BYTE_ARRAY_BASE_OFFSET : 二进制字节数组的起始索引，相对于字节数组对象
* LITTLE_ENDIAN ： 布尔值，是否为小端对齐（涉及到字节序的问题）
* heapMemory : 如果为堆内存，则指向访问的内存的引用，否则若内存为非堆内存，则为null
* address : 字节数组对应的相对地址（若heapMemory为null，即可能为off-heap内存的绝对地址，后续会详解）
* addressLimit : 标识地址结束位置（address+size）
* size : 内存段的字节数


提供了一大堆get/put方法，这些getXXX/putXXX大都直接或者间接调用了unsafe.getXXX/unsafe.putXXX。


比较
```java
public final int compare(MemorySegment seg2, int offset1, int offset2, int len) {
		while (len >= 8) {
			long l1 = this.getLongBigEndian(offset1);
			long l2 = seg2.getLongBigEndian(offset2);

			if (l1 != l2) {
				return (l1 < l2) ^ (l1 < 0) ^ (l2 < 0) ? -1 : 1;
			}

			offset1 += 8;
			offset2 += 8;
			len -= 8;
		}
		while (len > 0) {
			int b1 = this.get(offset1) & 0xff;
			int b2 = seg2.get(offset2) & 0xff;
			int cmp = b1 - b2;
			if (cmp != 0) {
				return cmp;
			}
			offset1++;
			offset2++;
			len--;
		}
		return 0;
	}
```

自实现的比较方法，用于对当前memory segment偏移offset1长度为len的数据与seg2偏移起始位offset2长度为len的数据进行比较。


1. 第一个while是逐字节比较，如果len的长度大于8就从各自的起始偏移量开始获取其数据的长整形表示进行对比，如果相等则各自后移8位(一个字节)，并且长度减8，以此循环往复。

getLongBigEndian获取一个长整形,判断是否是大端序，如果是小端序，就进行反转
```java
public final long getLongBigEndian(int index) {
		if (LITTLE_ENDIAN) {
			return Long.reverseBytes(getLong(index));
		} else {
			return getLong(index);
		}
	}
```

0x1234567的大端字节序和小端字节序的写法如下图。

![image](https://note.youdao.com/yws/api/personal/file/E691216E78614EDE94E60DBCCEDAD627?method=download&shareKey=ead114f4fa539399c0eade3135d0ab6e)

2. 第二个循环比较的是最后剩余不到一个字节(八个比特位)，因此是按位比较



#### HybridMemorySegment

它既支持on-heap内存也支持off-heap内存，通过如下实现区分
```java
 unsafe.XXX(Object o, int offset/position, ...)
 ```
这些方法有如下特点：
1. 如果对象o不为null，并且后面的地址或者位置是相对位置，那么会直接对当前对象（比如数组）的相对位置进行操作，既然这里对象不为null，那么这种情况自然满足on-heap的场景；
2. 如果对象o为null，并且后面的地址是某个内存块的绝对地址，那么这些方法的调用也相当于对该内存块进行操作。这里对象o为null，所操作的内存块不是JVM堆内存，这种情况满足了off-heap的场景。

针对堆内内存和堆外内存的构造函数也不一样

堆内内存
```java
	HybridMemorySegment(byte[] buffer, Object owner) {
		super(buffer, owner);
		this.offHeapBuffer = null;
	}
```

堆外内存,使用ByteBuffer，拥有这个实现DirectByteBuffer（直接内存）。
```java

HybridMemorySegment(ByteBuffer buffer, Object owner) {
		super(checkBufferAndGetAddress(buffer), buffer.capacity(), owner);
		this.offHeapBuffer = buffer;
	}
```


##### 如何获得某个off-heap数据的内存地址

堆外内存的address和堆内内存的不一样，堆内内存的address为0开始，堆外是一串绝对值地址。


off-heap使用的类是ByteBuffer，继承于Buffer，获取buffer类中的address需要使用反射,因为是一个私有变量


```java
private static final Field ADDRESS_FIELD;

	static {
		try {
			ADDRESS_FIELD = java.nio.Buffer.class.getDeclaredField("address");
			ADDRESS_FIELD.setAccessible(true);
		}
		catch (Throwable t) {
			throw new RuntimeException(
					"Cannot initialize HybridMemorySegment: off-heap memory is incompatible with this JVM.", t);
		}
	}
	
private static long getAddress(ByteBuffer buffer) {
		if (buffer == null) {
			throw new NullPointerException("buffer is null");
		}
		try {
			return (Long) ADDRESS_FIELD.get(buffer);
		}
		catch (Throwable t) {
			throw new RuntimeException("Could not access direct byte buffer address.", t);
		}
	}
```


### MemoryManager

![image](https://note.youdao.com/yws/api/personal/file/F1F06793CD8F408B9CCE6D471CE7AA30?method=download&shareKey=6484343ac7362ab189ddf0e589077d6f)


MemoryManager提供了两个内部类HybridHeapMemoryPool和HybridOffHeapMemoryPool，代表堆内内存池和堆外内存池.

为了提升memory segment操作效率，MemoryManager鼓励长度相等的memory segment。由此引入了page的概念。其实page跟memory segment没有本质上的区别，只不过是为了体现memory segment被分配为均等大小的内存空间而引入的。可以将这个类比于操作系统的页式内存分配，page这里看着同等大小的block即可. page提供了切页操作，如果剩下2个byte的空间，又要录入4个byte，如果是具有切页的inputView或者outPutView，就可以切换到下一个segment，否则抛出异常。


### DataInput 数据视图

提供了基于page的对view的进一步实现，说得更直白一点就是，它提供了跨越多个memory page的数据访问(input/output)视图。它包含了从page中读取/写入数据的解码/编码方法以及跨越page的边界检查（边界检查主要由实现类来完成）。

![image](https://note.youdao.com/yws/api/personal/file/6096E819C1E84DE3985771AEBD0DC4FE?method=download&shareKey=cae73308266945a7fb6b1847cefc84d9)

### 类型机制与内存管理

![image](https://note.youdao.com/yws/api/personal/file/B10CA158FF0E4D68BEF5F7CEB869FA2E?method=download&shareKey=b594fcc514856b56390a039fab1e8614)


在Flink中， java的String会对应的是 Types.STRING，然后通过 StringSerializer序列化String实体到PageOutPutView中，PageOutPutView会将数据写到Segment中。

## Flink序列化


TypeInformation 类是描述一切类型的公共基类，它和它的所有子类必须可序列化（Serializable），因为类型信息将会伴随 Flink 的作业提交，被传递给每个执行节点。

类型信息由 TypeInformation 类表示，TypeInformation 支持以下几种类型：

* BasicTypeInfo: 任意Java 基本类型（装箱的）或 String 类型。
* BasicArrayTypeInfo: 任意Java基本类型数组（装箱的）或 String 数组。
* WritableTypeInfo: 任意 Hadoop Writable 接口的实现类。
* TupleTypeInfo: 任意的 Flink Tuple 类型(支持Tuple1 to Tuple25)。Flink tuples 是固定长度固定类型的Java Tuple实现。
* CaseClassTypeInfo: 任意的 Scala CaseClass(包括 Scala tuples)。
* PojoTypeInfo: 任意的 POJO (Java or Scala)，例如，Java对象的所有成员变量，要么是 public 修饰符定义，要么有 getter/setter 方法。
* GenericTypeInfo: 任意无法匹配之前几种类型的类。

![image](https://note.youdao.com/yws/api/personal/file/9B6167B879D44D0897292590B37E9D61?method=download&shareKey=475b5c6e1e1984afcbc727f42d246e3c)

### TypSerializer

所有序列化的基础类。此接口描述Flink运行时处理数据类型所需的方法。 具体来说，此接口包含序列化和复制方法。

其中有一个深度拷贝的方法duplicate，因为这个序列化器是非线程安全的，如果序列化器不具备状态，直接返回自身，如果有状态，调用这个方法返回一个深度拷贝（抽象方法，具体实现子类自行实现。）

```java
/**
	 * Creates a deep copy of this serializer if it is necessary, i.e. if it is stateful. This
	 * can return itself if the serializer is not stateful.
	 *
	 * We need this because Serializers might be used in several threads. Stateless serializers
	 * are inherently thread-safe while stateful serializers might not be thread-safe.
	 */
	public abstract TypeSerializer<T> duplicate();
```

还有序列化和逆序列化的定义，将数据与DataView进行转换，dataView作为MemorySegment 的抽象显示，序列化与逆序列化就是讲自定义的内存结构与实体进行转换的过程。

```java
/**
	 * Serializes the given record to the given target output view.
	 * 
	 * @param record The record to serialize.
	 * @param target The output view to write the serialized data to.
	 * 
	 * @throws IOException Thrown, if the serialization encountered an I/O related error. Typically raised by the
	 *                     output view, which may have an underlying I/O channel to which it delegates.
	 */
	public abstract void serialize(T record, DataOutputView target) throws IOException;

	/**
	 * De-serializes a record from the given source input view.
	 * 
	 * @param source The input view from which to read the data.
	 * @return The deserialized element.
	 * 
	 * @throws IOException Thrown, if the de-serialization encountered an I/O related error. Typically raised by the
	 *                     input view, which may have an underlying I/O channel from which it reads.
	 */
	public abstract T deserialize(DataInputView source) throws IOException;
```

选几个具体的序列化器分析一下

分为定长(LongSerializer)和变长(StringSerializer)，pojoSerializer

##### LongSerializer

```java
@Override
	public void serialize(Long record, DataOutputView target) throws IOException {
		target.writeLong(record);
	}

	@Override
	public Long deserialize(DataInputView source) throws IOException {
		return source.readLong();
	}
```
直接调用DataView的方法

AbstractPagedOutputView.writeLong

```java
public void writeLong(long v) throws IOException {
		if (this.positionInSegment < this.segmentSize - 7) {
			this.currentSegment.putLongBigEndian(this.positionInSegment, v);
			this.positionInSegment += 8;
		}
		else if (this.positionInSegment == this.segmentSize) {
			advance();
			writeLong(v);
		}
		else {
			writeByte((int) (v >> 56));
			writeByte((int) (v >> 48));
			writeByte((int) (v >> 40));
			writeByte((int) (v >> 32));
			writeByte((int) (v >> 24));
			writeByte((int) (v >> 16));
			writeByte((int) (v >>  8));
			writeByte((int) v);
		}
	}
```

通过this.positionInSegment < this.segmentSize - 7判断剩余的长度，如果剩余的长度还够8位，那么直接put进去

```java
this.currentSegment.putLongBigEndian(this.positionInSegment, v);
```
这里会判断是大端序还是小端序，如果是小端，需要做一次转换
```java
public final void putLongBigEndian(int index, long value) {
		if (LITTLE_ENDIAN) {
			putLong(index, Long.reverseBytes(value));
		} else {
			putLong(index, value);
		}
	}
```
如果剩余的不足8位，且刚好剩余为0，通过advance进行切换页操作，然后写。如果剩余的还有一些，因为long是8个byte，所以每次写一个byte进去，不够再切页。切页操作是由具体的子类实现的。




##### StringSerializer 

重点在于序列化和逆序列化
```java
@Override
	public void serialize(String record, DataOutputView target) throws IOException {
		StringValue.writeString(record, target);
	}

	@Override
	public String deserialize(DataInputView source) throws IOException {
		return StringValue.readString(source);
	}
```
重点的放在StringValue中。

##### StringValue

变长的string，是比较复杂的。如下是几个重要的规则
* byte的范围在-128 --- 127， -128 --- 0 表示大于127并且这个string未结束的标识。
* 以一个byte为一位，以有符号位写入
* 在0-128(2^8)之间的直接写入
*  **大于128的字符读取结束的条件是遇到小于128的byte，所以变长字符的写入要大于128，通过每次写入7位，第八位强制设为1实现（强行转为无符号位）。** 


举例：
一个汉字转成int后为27979，那么这个根据上面的规律得到的是28107 218 1。以有符号写入，会得到 -53 -38 1。 从这个就可以很简单的看到，以负数开始，到正数结尾，为一个char的构成。




在这个类中，定义了一个高bit  0x1 << 7(128，ASCII 定义了128个字符)

```java
	private static final int HIGH_BIT = 0x1 << 7;
```

先研究写入，写入的时候，是通过out.write(byte)写入数据，一次最多只能写入8位数据。flink的做法是每次获取7位的数据，然后第八位通过（ | HIGH_BIT[1000 0000]） 强制置为1，在二进制中，置为第八位代表正负，1为负，0为正。然后右移7位，获取下一个高7位的数据，以此类推，写入到流中。



测 这个字 对应的 (int)s.charAt(0)为27979，
* 写入的时候，先获取长度+1，第一位为2。（如果长度为1，是null）
* 将27979与HIGH_BIT进行或操作将第八位置为1，得到28107，其实真实写入的是28107的后八位11001011
* 右移7位后进行计算得到218
* 再右移一次得到1
* 序列化后的是  2 28107 218 1 
* 以有符号位写入byte中得到2 -53 -38 1. 

因为有符号范围在 -128  ---- 127之间，无符号是在0-256，由于写是默认按照有符号写，所以写11001011会变成-53.


```java
public static final void writeString(CharSequence cs, DataOutput out) throws IOException {
		if (cs != null) {
			// the length we write is offset by one, because a length of zero indicates a null value
			int lenToWrite = cs.length()+1;
			if (lenToWrite < 0) {
				throw new IllegalArgumentException("CharSequence is too long.");
			}
	
			// write the length, variable-length encoded
			while (lenToWrite >= HIGH_BIT) {
				out.write(lenToWrite | HIGH_BIT);
				lenToWrite >>>= 7;
			}
			out.write(lenToWrite);
	
			// write the char data, variable length encoded
			for (int i = 0; i < cs.length(); i++) {
				int c = cs.charAt(i);
	
				while (c >= HIGH_BIT) {
					out.write(c | HIGH_BIT);
					c >>>= 7;
				}
				out.write(c);
			}
		} else {
			out.write(0);
		}
	}
```

#### 逆序列化

逆序列化的时候，逆序列化 2 -53 -38 1. 

```java
        if (c < HIGH_BIT) {
				data[i] = (char) c;
			}
```

默认如果小于128，就会当做是结束条件，直接转换成char。所以在变长的string中，还没结束的部分，一定要大于128，这就是为什么写入的时候，一定要强制将第八位置为1.

* 读取的2-1=1获取数据长度为1。 
* -53 通过in.readUnsignedByte()得到203,对应11001011，取七位1001011，作为低七位
* -38，通过in.readUnsignedByte()得到218，继续获取218[11011010]，取七位[1011010]，再进行左移7位操作作为高一阶的七位，拼上低七位，得到10110101001011。
* 最后一个1，进行左移14位操作作为再高一阶的七位，拼上低十四位得到110110101001011对应的十进制为27979，转为ascII为测这个字。




```java
public static String readString(DataInput in) throws IOException {
		// the length we read is offset by one, because a length of zero indicates a null value
		int len = in.readUnsignedByte();
		
		if (len == 0) {
			return null;
		}

		if (len >= HIGH_BIT) {
			int shift = 7;
			int curr;
			len = len & 0x7f;
			while ((curr = in.readUnsignedByte()) >= HIGH_BIT) {
				len |= (curr & 0x7f) << shift;
				shift += 7;
			}
			len |= curr << shift;
		}
		
		// subtract one for the null length
		len -= 1;
		
		final char[] data = new char[len];

		for (int i = 0; i < len; i++) {
			int c = in.readUnsignedByte();
			if (c < HIGH_BIT) {
				data[i] = (char) c;
			} else {
				int shift = 7;
				int curr;
				c = c & 0x7f;
				while ((curr = in.readUnsignedByte()) >= HIGH_BIT) {
					c |= (curr & 0x7f) << shift;
					shift += 7;
				}
				c |= curr << shift;
				data[i] = (char) c;
			}
		}
		
		return new String(data, 0, len);
	}
```



POJO多个一个字节的header，PojoSerializer只负责将header序列化进去，并委托每个字段对应的serializer对字段进行序列化。

```java
	// Flags for the header
	private static byte IS_NULL = 1;
	private static byte NO_SUBCLASS = 2;
	private static byte IS_SUBCLASS = 4;
	private static byte IS_TAGGED_SUBCLASS = 8;
```

序列化的时候，会先判断是否是null，是的话，置为IS_NULL.

接着进行header的判断。
如果Class<?> actualClass = value.getClass();等于构造函数的class，说明这个class不是一个subClass，置为IS_SUBCLASS。如果在registeredClasses可以获取到子类，说明这个类自身存在子类，他是一个父类，置为IS_TAGGED_SUBCLASS
先将flag写入，target.writeByte(flags);
如果是subclass，将类的全名写入序列化中，如果



如果是NO_SUBCLASS，直接

```
header{
    flag,
    subFlag,(class tag id  or the full classname)
}
```
写完header，再委托每个字段对应的serializer对字段进行序列化。
```java
public void serialize(T value, DataOutputView target) throws IOException {
		int flags = 0;
		// handle null values
		if (value == null) {
			flags |= IS_NULL;
			target.writeByte(flags);
			return;
		}

		Integer subclassTag = -1;
		Class<?> actualClass = value.getClass();
		TypeSerializer subclassSerializer = null;
		if (clazz != actualClass) {
			subclassTag = registeredClasses.get(actualClass);
			if (subclassTag != null) {
				flags |= IS_TAGGED_SUBCLASS;
				subclassSerializer = registeredSerializers[subclassTag];
			} else {
				flags |= IS_SUBCLASS;
				subclassSerializer = getSubclassSerializer(actualClass);
			}
		} else {
			flags |= NO_SUBCLASS;
		}

		target.writeByte(flags);

		// if its a registered subclass, write the class tag id, otherwise write the full classname
		if ((flags & IS_SUBCLASS) != 0) {
			target.writeUTF(actualClass.getName());
		} else if ((flags & IS_TAGGED_SUBCLASS) != 0) {
			target.writeByte(subclassTag);
		}

		// if its a subclass, use the corresponding subclass serializer,
		// otherwise serialize each field with our field serializers
		if ((flags & NO_SUBCLASS) != 0) {
			try {
				for (int i = 0; i < numFields; i++) {
					Object o = (fields[i] != null) ? fields[i].get(value) : null;
					if (o == null) {
						target.writeBoolean(true); // null field handling
					} else {
						target.writeBoolean(false);
						fieldSerializers[i].serialize(o, target);
					}
				}
			} catch (IllegalAccessException e) {
				throw new RuntimeException("Error during POJO copy, this should not happen since we check the fields before.", e);
			}
		} else {
			// subclass
			if (subclassSerializer != null) {
				subclassSerializer.serialize(value, target);
			}
		}
	}
```


![image](https://note.youdao.com/yws/api/personal/file/4FE22225B71B474A834F13F69B7C789E?method=download&shareKey=54f98b410bf649be112f69db79b4afc7)

用户为了保证数据能使用Flink自带的序列化器，有时候不得不自己再重写一个POJO类，把外部系统中数据的值再“映射”到这个POJO类中；而根据开发人员对POJO的理解不同，写出来的效果可能不一样，比如之前有个用户很肯定地说自己是按照POJO的规范来定义的类，我查看后发现原来他不小心多加了个logger，这从侧面说明还是有一定的用户使用门槛的。

```java
// Not a POJO demo.public
class Person {  
    
    private Logger logger = LoggerFactory.getLogger(Person.class);  
    
    public String name; 
    
    public int age;
    
}
```


## Flink 如何直接操作二进制数据

### 排序

Flink 提供了如 group、sort、join 等操作，这些操作都需要访问海量数据。这里，我们以sort为例，这是一个在 Flink 中使用非常频繁的操作。

首先，Flink 会从 MemoryManager 中申请一批 MemorySegment，我们把这批 MemorySegment 称作 sort buffer，用来存放排序的数据。

将实际的数据和指针加定长key分开存放有两个目的。第一，交换定长块（key+pointer）更高效，不用交换真实的数据也不用移动其他key和pointer。第二，这样做是缓存友好的，因为key都是连续存储在内存中的，可以大大减少 cache miss（后面会详细解释）。

排序的关键是比大小和交换。Flink 中，会先用 key 比大小，这样就可以直接用二进制的key比较而不需要反序列化出整个对象。因为key是定长的，所以如果key相同（或者没有提供二进制key），那就必须将真实的二进制数据反序列化出来，然后再做比较。之后，只需要交换key+pointer就可以达到排序的效果，真实的数据不用移动。

最后，访问排序后的数据，可以沿着排好序的key+pointer区域顺序访问，通过pointer找到对应的真实数据，并写到内存或外部

![image](https://note.youdao.com/yws/api/personal/file/11B1637FC25C4C3686FAD944DCA964FF?method=download&shareKey=d3c087ab46b1888a2002fc238598ef44)

![image](https://note.youdao.com/yws/api/personal/file/86CB5AB7CCAE4DB8AB9F185D0860CA82?method=download&shareKey=a3527963a8da3fadb9da6cab519a985a)

可见NormalizedKeySorter.java


