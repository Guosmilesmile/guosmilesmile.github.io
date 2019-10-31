---
title: Flink浅入浅出
date: 2019-10-31 23:11:46
tags:
categories:
	- Flink
---



## System Architecture


![image](https://note.youdao.com/yws/api/personal/file/3C921DF22FB1483DA8200FC6993411CC?method=download&shareKey=a924019c2b3b537d1bd95bfcd06d6b76)

![image](https://note.youdao.com/yws/api/personal/file/81D17722B2304165A251569F0CBCB2BA?method=download&shareKey=761e56d216c0832f923753677def7816)

当 Flink 集群启动后，首先会启动一个 JobManger 和一个或多个的 TaskManager。由 Client 提交任务给 JobManager，JobManager 再调度任务到各个 TaskManager 去执行，然后 TaskManager 将心跳和统计信息汇报给 JobManager。TaskManager 之间以流的形式进行数据的传输。上述三者均为独立的 JVM 进程。

* **Client** 为提交 Job 的客户端，可以是运行在任何机器上（与 JobManager 环境连通即可）。提交 Job 后，Client 可以结束进程（Streaming的任务），也可以不结束并等待结果返回。
* **JobManager** 主要负责调度 Job 并协调 Task 做 checkpoint，职责上很像 Storm 的 Nimbus。从 Client 处接收到 Job 和 JAR 包等资源后，会生成优化后的执行计划，并以 Task 的单元调度到各个 TaskManager 去执行
* **TaskManager** 在启动的时候就设置好了槽位数（Slot），每个 slot 能启动一个 Task，Task 为线程。从 JobManager 处接收需要部署的 Task，部署启动后，与自己的上游建立 Netty 连接，接收数据并处理
* **ResourceManager**：一般是Yarn，当TM有空闲的slot就会告诉JM，没有足够的slot也会启动新的TM。kill掉长时间空闲的TM。

可以看到 Flink 的任务调度是多线程模型，并且不同Job/Task混合在一个 TaskManager 进程中。虽然这种方式可以有效提高 CPU 利用率，但是个人不太喜欢这种设计，因为不仅缺乏资源隔离机制，同时也不方便调试。如果直接以standalone的模式运行，一个TM崩溃会导致其他程序一起被波及。类似 Storm 的进程模型，一个JVM 中只跑该 Job 的 Tasks 实际应用中更为合理。

### Graph

Flink中最容易混乱的是各种图。

Flink 中的执行图可以分成四层：StreamGraph -> JobGraph -> ExecutionGraph -> 物理执行图。

* **StreamGraph**：是根据用户通过 Stream API 编写的代码生成的最初的图。用来表示程序的拓扑结构。
* * StreamNode：用来代表 operator 的类，并具有所有相关的属性，如并发度、入边和出边等。
* * StreamEdge：表示连接两个StreamNode的边。
* **JobGraph**：StreamGraph经过优化后生成了 JobGraph，提交给 JobManager 的数据结构。主要的优化为，将多个符合条件的节点 chain 在一起作为一个节点，这样可以减少数据在节点之间流动所需要的序列化/反序列化/传输消耗。
* * JobVertex：经过优化后符合条件的多个StreamNode可能会chain在一起生成一个JobVertex，即一个JobVertex包含一个或多个operator，JobVertex的输入是JobEdge，输出是IntermediateDataSet。
* * IntermediateDataSet：表示JobVertex的输出，即经过operator处理产生的数据集。producer是JobVertex，consumer是JobEdge。
* * JobEdge：代表了job graph中的一条数据传输通道。source 是 IntermediateDataSet，target 是 JobVertex。即数据通过JobEdge由IntermediateDataSet传递给目标JobVertex。
* **ExecutionGraph**：JobManager 根据 JobGraph 生成ExecutionGraph。ExecutionGraph是JobGraph的并行化版本，是调度层最核心的数据结构。
* * ExecutionJobVertex：和JobGraph中的JobVertex一一对应。每一个ExecutionJobVertex都有和并发度一样多的 ExecutionVertex。
* * ExecutionVertex：表示ExecutionJobVertex的其中一个并发子任务，输入是ExecutionEdge，输出是IntermediateResultPartition。
* * IntermediateResult：和JobGraph中的IntermediateDataSet一一对应。一个IntermediateResult包含多个IntermediateResultPartition，其个数等于该operator的并发度。
* * IntermediateResultPartition：表示ExecutionVertex的一个输出分区，producer是ExecutionVertex，consumer是若干个ExecutionEdge。
* * ExecutionEdge：表示ExecutionVertex的输入，source是IntermediateResultPartition，target是ExecutionVertex。source和target都只能是一个。
* * Execution：是执行一个 ExecutionVertex 的一次尝试。当发生故障或者数据需要重算的情况下 ExecutionVertex 可能会有多个 ExecutionAttemptID。一个 Execution 通过 ExecutionAttemptID 来唯一标识。JM和TM之间关于 task 的部署和 task status 的更新都是通过 ExecutionAttemptID 来确定消息接受者。
* **物理执行图**：JobManager 根据 ExecutionGraph 对 Job 进行调度后，在各个TaskManager 上部署 Task 后形成的“图”，并不是一个具体的数据结构。
* * **Task**：Execution被调度后在分配的 TaskManager 中启动对应的 Task。Task 包裹了具有用户执行逻辑的 operator。
* * **ResultPartition**：代表由一个Task的生成的数据，和ExecutionGraph中的IntermediateResultPartition一一对应。
* * **ResultSubpartition**：是ResultPartition的一个子分区。每个ResultPartition包含多个ResultSubpartition，其数目要由下游消费 Task 数和DistributionPattern 来决定。
* * **InputGate**：代表Task的输入封装，和JobGraph中JobEdge一一对应。每个InputGate消费了一个或多个的ResultPartition。
* * **InputChannel**：每个InputGate会包含一个以上的InputChannel，和ExecutionGraph中的ExecutionEdge一一对应，也和ResultSubpartition一对一地相连，即一个InputChannel接收一个ResultSubpartition的输出。




![image](https://note.youdao.com/yws/api/personal/file/56744FB7B89448F2A58698D5D6151587?method=download&shareKey=c4dddaf0898abdac28d4123cd0a4d33c)


那么 Flink 为什么要设计这4张图呢，其目的是什么呢？Spark 中也有多张图，数据依赖图以及物理执行的DAG。其目的都是一样的，就是解耦，每张图各司其职，每张图对应了 Job 不同的阶段，更方便做该阶段的事情。我们给出更完整的 Flink Graph 的层次图。


![image](https://note.youdao.com/yws/api/personal/file/E38FE51B54CC46E48308788CD5BEBCDB?method=download&shareKey=b172b69f649959b0cea6d190515fdf10)





## API

![image](https://note.youdao.com/yws/api/personal/file/D2CAD0BF1CAB4BF093337C68F3C02D25?method=download&shareKey=03b69b2e109ea45ef7893c5a0fdb5f56)


## 如何生成 StreamGraph


### Transformation

StreamGraph 相关的代码主要在 org.apache.flink.streaming.api.graph 包中。
构造StreamGraph的入口函数是 StreamGraphGenerator.generate(env, transformations)。
该函数会由触发程序执行的方法StreamExecutionEnvironment.execute()调用到

StreamExecutionEnvironment，有下面一个属性：
```java
protected final List<StreamTransformation<?>> transformations = new ArrayList<>();
```


StreamTransformation把用户通过Streaming API提交的udf（如FlatMapFunction 的对象)作为自己的operator属性存储，同时还把上游的transformation作 为input属性存储。

用户通过api构造transformation存储到StreamExecutionEnvironment
![image](https://note.youdao.com/yws/api/personal/file/1E3A4624349E46D7B8061A819CD24BE1?method=download&shareKey=9af7d47db7aa71958f3d9776482299a8)

其分层实现如下图所示：



![image](https://note.youdao.com/yws/api/personal/file/174D7420E39249ADB6B99666A11D3B5D?method=download&shareKey=f28aa6ea89da56ae6cf184a1bb908b1d)

```
public class OneInputTransformation<IN, OUT> extends StreamTransformation<OUT> {

	private final StreamTransformation<IN> input;

	private final OneInputStreamOperator<IN, OUT> operator;

}
```

1. StreamExecutionEnvironment不存储SourceTransformation, 因为flink不允许提交只有Source的job，而根据其他类型的Transformation的input引用可以回溯到SourceTransformation。
2. Stream可以分为两种类型，一种是继承DataStream类，另一种不继承；功能上的区别在于，前者产生transformation（flink会根据transformation的组织情况构建DAG)，后者不产生transformation，但是会赋予Stream一些特殊的功能。可以产生transformation又被称为顶点，不产生顶点的为称为虚节点，虚节点主要用于说明数据的的分发策略和窗口等，例如gruop by，window, iterate, union。

关于2的具体说明：
* 并不是每一个 StreamTransformation 都会转换成 runtime 层中物理操作。有一些只是逻辑概念，比如 union、split/select、partition等。如下图所示的转换树，在运行时会优化成下方的操作图。

union、split/select、partition中的信息会被写入到 Source –> Map 的边中。通过源码也可以发现，UnionTransformation,SplitTransformation,SelectTransformation,PartitionTransformation由于不包含具体的操作所以都没有StreamOperator成员变量，而其他StreamTransformation的子类基本上都有。

![image](https://note.youdao.com/yws/api/personal/file/C09CD7A557F54A15AAA3DD4CE14C0DB7?method=download&shareKey=c827c042846fae0e5d66da23239af462)

### StreamOperator

DataStream 上的每一个 Transformation 都对应了一个 StreamOperator，StreamOperator是运行时的具体实现，会决定UDF(User-Defined Funtion)的调用方式.下图所示为 StreamOperator 的类图（点击查看大图）：

![image](https://note.youdao.com/yws/api/personal/file/FDEA3D778C64435BBDA3EC8E1D7B1419?method=download&shareKey=423692cd03fdf5c87c8d6bd894e5a6b9)


### 生成 StreamGraph 的源码分析

我们通过在DataStream上做了一系列的转换（map、filter等）得到了StreamTransformation集合，然后通过StreamGraphGenerator.generate获得StreamGraph，该方法的源码如下：

```java
    /**
	 * Transforms one {@code StreamTransformation}.
	 *
	 * <p>This checks whether we already transformed it and exits early in that case. If not it
	 * delegates to one of the transformation specific methods.
	 * 检查这个算子是否被加入，如果不存在，将这个转换算子转为一个
	 */
	private Collection<Integer> transform(StreamTransformation<?> transform) {

		// 判断传入的transform是否已经被转化过, 如果已经转化过, 则直接返回转化后对应的结果
		if (alreadyTransformed.containsKey(transform)) {
			return alreadyTransformed.get(transform);
		}

		LOG.debug("Transforming " + transform);


		if (transform.getMaxParallelism() <= 0) {

			// if the max parallelism hasn't been set, then first use the job wide max parallelism
			// from the ExecutionConfig.
			int globalMaxParallelismFromConfig = env.getConfig().getMaxParallelism();
			if (globalMaxParallelismFromConfig > 0) {
				transform.setMaxParallelism(globalMaxParallelismFromConfig);
			}
		}

		//至少调用一次以触发有关MissingTypeInfo的异常
		// call at least once to trigger exceptions about MissingTypeInfo
		transform.getOutputType();

		Collection<Integer> transformedIds;
        //针对不同的StreamTransformation的子类实现，委托不同的方法进行转化
		if (transform instanceof OneInputTransformation<?, ?>) {
			//单流输入转换
			transformedIds = transformOneInputTransform((OneInputTransformation<?, ?>) transform);
		} else if (transform instanceof TwoInputTransformation<?, ?, ?>) {
			//双流输入转换
			transformedIds = transformTwoInputTransform((TwoInputTransformation<?, ?, ?>) transform);
		} else if (transform instanceof SourceTransformation<?>) {
			//数据源转换
			transformedIds = transformSource((SourceTransformation<?>) transform);
		} else if (transform instanceof SinkTransformation<?>) {
			//sink转换
			transformedIds = transformSink((SinkTransformation<?>) transform);
		} else if (transform instanceof UnionTransformation<?>) {
			//union转换
			transformedIds = transformUnion((UnionTransformation<?>) transform);
		} else if (transform instanceof SplitTransformation<?>) {
			//split转换
			transformedIds = transformSplit((SplitTransformation<?>) transform);
		} else if (transform instanceof SelectTransformation<?>) {
			//select转换
			transformedIds = transformSelect((SelectTransformation<?>) transform);
		} else if (transform instanceof FeedbackTransformation<?>) {
			//反馈转换（迭代计算）
			transformedIds = transformFeedback((FeedbackTransformation<?>) transform);
		} else if (transform instanceof CoFeedbackTransformation<?>) {
			//Co反馈转换（迭代计算）
			transformedIds = transformCoFeedback((CoFeedbackTransformation<?>) transform);
		} else if (transform instanceof PartitionTransformation<?>) {
			//分区转换
			transformedIds = transformPartition((PartitionTransformation<?>) transform);
		} else if (transform instanceof SideOutputTransformation<?>) {
			//流输出
			transformedIds = transformSideOutput((SideOutputTransformation<?>) transform);
		} else {
			throw new IllegalStateException("Unknown transformation: " + transform);
		}

		//因为存在迭代计算，需要再次校验是否是已经转换的算子
		// need this check because the iterate transformation adds itself before
		// transforming the feedback edges
		if (!alreadyTransformed.containsKey(transform)) {
			alreadyTransformed.put(transform, transformedIds);
		}

		//装填各种属性
		if (transform.getBufferTimeout() >= 0) {
			streamGraph.setBufferTimeout(transform.getId(), transform.getBufferTimeout());
		}
		if (transform.getUid() != null) {
			streamGraph.setTransformationUID(transform.getId(), transform.getUid());
		}
		if (transform.getUserProvidedNodeHash() != null) {
			streamGraph.setTransformationUserHash(transform.getId(), transform.getUserProvidedNodeHash());
		}

		if (transform.getMinResources() != null && transform.getPreferredResources() != null) {
			streamGraph.setResources(transform.getId(), transform.getMinResources(), transform.getPreferredResources());
		}

		return transformedIds;
	}
```

最终都会调用 transformXXX 来对具体的StreamTransformation进行转换。我们可以看下transformOnInputTransform(transform)的实现：

递归的将这个节点和上游节点加入图中，并将这两个点构成一条边，加入图中。
```java
/**
	 * Transforms a {@code OneInputTransformation}.
	 *
	 * <p>This recursively transforms the inputs, creates a new {@code StreamNode} in the graph and
	 * wired the inputs to this new node.
	 */
	private <IN, OUT> Collection<Integer> transformOneInputTransform(OneInputTransformation<IN, OUT> transform) {

		/// 先递归转化对应的input属性 (上游算子)
		Collection<Integer> inputIds = transform(transform.getInput());

		//在递归调用过程中，有可能已经被转化过的算子直接转换
		// the recursive call might have already transformed this
		if (alreadyTransformed.containsKey(transform)) {
			return alreadyTransformed.get(transform);
		}

		//判断transform的槽共享组的名称
		String slotSharingGroup = determineSlotSharingGroup(transform.getSlotSharingGroup(), inputIds);

		streamGraph.addOperator(transform.getId(),
				slotSharingGroup,
				transform.getCoLocationGroupKey(),
				transform.getOperator(),
				transform.getInputType(),
				transform.getOutputType(),
				transform.getName());

		if (transform.getStateKeySelector() != null) {
			TypeSerializer<?> keySerializer = transform.getStateKeyType().createSerializer(env.getConfig());
			streamGraph.setOneInputStateKey(transform.getId(), transform.getStateKeySelector(), keySerializer);
		}

		streamGraph.setParallelism(transform.getId(), transform.getParallelism());
		streamGraph.setMaxParallelism(transform.getId(), transform.getMaxParallelism());

		//构建edge ,上游和本节点id ，组合成一条边
		for (Integer inputId: inputIds) {
			streamGraph.addEdge(inputId, transform.getId(), 0);
		}

		return Collections.singleton(transform.getId());
	}
```

该函数首先会对该transform的上游transform进行递归转换，确保上游的都已经完成了转化。然后通过transform构造出StreamNode，最后与上游的transform进行连接，构造出StreamNode。

最后再来看下对逻辑转换（partition、union等）的处理，如下是transformPartition函数的源码：



```java
	/**
	 * Transforms a {@code PartitionTransformation}.
	 *  需要在流图中创建一个虚拟节点，维护分区信息。
	 * <p>For this we create a virtual node in the {@code StreamGraph} that holds the partition
	 * property. @see StreamGraphGenerator
	 */
	private <T> Collection<Integer> transformPartition(PartitionTransformation<T> partition) {
		//获取上游算子信息
		StreamTransformation<T> input = partition.getInput();
		List<Integer> resultIds = new ArrayList<>();

		//递归转化上游输入
		Collection<Integer> transformedIds = transform(input);
		for (Integer transformedId: transformedIds) {
			int virtualId = StreamTransformation.getNewNodeId();
			streamGraph.addVirtualPartitionNode(transformedId, virtualId, partition.getPartitioner());
			resultIds.add(virtualId);
		}

		return resultIds;
	}
```


keyby对应的算子是PartitionTransformation，是一个分区操作.对partition的转换没有生成具体的StreamNode和StreamEdge,需要在流图中创建一个虚拟节点，维护分区信息。在virtualPartitionNodes这个map中就新增映射：虚拟节点id ——> [上游节点id, partitioner],会把partition信息写入到edge中。具体见StreamGraph.addEdgeInternal。


### 实例讲解

如下程序，是一个从 Source 中按行切分成单词并过滤输出的简单流程序，其中包含了逻辑转换：随机分区shuffle。我们会分析该程序是如何生成StreamGraph的。

```java
DataStream<String> text = env.socketTextStream(hostName, port);
text.flatMap(new LineSplitter()).shuffle().filter(new HelloFilter()).print();
```

首先会在env中生成一棵transformation树，用List<StreamTransformation<?>>保存。其结构图如下：

![image](https://note.youdao.com/yws/api/personal/file/762DCAB08F8A4AA1AF26C58D57E85E94?method=download&shareKey=fd0e50ab78f984ed76fc2967cad8a39f)

其中符号*为input指针，指向上游的transformation，从而形成了一棵transformation树。然后，通过调用StreamGraphGenerator.generate(env, transformations)来生成StreamGraph。自底向上递归调用每一个transformation，也就是说处理顺序是Source->FlatMap->Shuffle->Filter->Sink。

![image](https://note.youdao.com/yws/api/personal/file/13C2597E71154BD08869DBD836E6EE54?method=download&shareKey=bbbe2ef15a2b1247d4557c4afdaae6fc)

如上图所示：

1. 首先处理的Source，生成了Source的StreamNode。
2. 然后处理的FlatMap，生成了FlatMap的StreamNode，并生成StreamEdge连接上游Source和FlatMap。由于上下游的并发度不一样（1:4），所以此处是Rebalance分区。
3. 然后处理的Shuffle，由于是逻辑转换，并不会生成实际的节点。将partitioner信息暂存在virtuaPartitionNodes中。
4. 在处理Filter时，生成了Filter的StreamNode。发现上游是shuffle，找到shuffle的上游FlatMap，创建StreamEdge与Filter相连。并把ShufflePartitioner的信息写到StreamEdge中。
5. 最后处理Sink，创建Sink的StreamNode，并生成StreamEdge与上游Filter相连。由于上下游并发度一样（4:4），所以此处选择 Forward 分区。


最后可以通过 UI可视化 来观察得到的 StreamGraph。

![image](https://note.youdao.com/yws/api/personal/file/0968FD77C6CF49F48D29F7C40726DDFD?method=download&shareKey=92adfed004150970b895e64163252066)




## 如何生成 JobGraph

StreamGraph和JobGraph有什么区别，StreamGraph是逻辑上的DAG图，不需要关心jobManager怎样去调度每个Operator的调度和执行；JobGraph是对StreamGraph进行切分，因为有些节点可以打包放在一起被JobManage安排调度，因此JobGraph的DAG每一个顶点就是JobManger的一个调度单位


根据用户用Stream API编写的程序，构造出一个代表拓扑结构的StreamGraph的。以 WordCount 为例，转换图如下图所示：
![image](https://note.youdao.com/yws/api/personal/file/EF7D858EB2F7473C81E03C0BBEBA6D70?method=download&shareKey=b61bdca75b60265139b0415e00aa3dee)

StreamGraph 和 JobGraph 都是在 Client 端生成的，也就是说我们可以在 IDE 中通过断点调试观察 StreamGraph 和 JobGraph 的生成过程。

StreamGraph的切分，实际上是逐条审查每一个StreamAdge和改SteamAdge两头连接的两个StreamNode的特性，来决定改StreamAdge两头的StreamNode是不是可以打包在一起，flink给出了明确的规则，看面的代码段：

```java
/**
	 * 判断两个node是否能链接到一起
	 * @param edge
	 * @param streamGraph
	 * @return
	 */
	public static boolean isChainable(StreamEdge edge, StreamGraph streamGraph) {
		//获取上游的Node
		StreamNode upStreamVertex = streamGraph.getSourceVertex(edge);
		//获取下游的Node
		StreamNode downStreamVertex = streamGraph.getTargetVertex(edge);

		//获取上游的算子
		StreamOperator<?> headOperator = upStreamVertex.getOperator();
		//获取下游的算子
		StreamOperator<?> outOperator = downStreamVertex.getOperator();

		/**
		 * 满足下面条件的两个node可以连接到一起
		 * 1. 下游的入边只有一条
		 * 2. 上下游算子均不为空
		 * 3. 上下游是否在同一个shareGrouop资源组中
		 * 4. 下游的连接策略是always
		 * 5. 下游的连接策略是head或者always（不为never）
		 * 6. 边的分区分发方式是ForwardPartitioner
		 * 7. 上下游的并行度一致
		 * 8. 全局配置是可以链接的
 		 */
		return downStreamVertex.getInEdges().size() == 1
			&& outOperator != null
			&& headOperator != null
			&& upStreamVertex.isSameSlotSharingGroup(downStreamVertex)
			&& outOperator.getChainingStrategy() == ChainingStrategy.ALWAYS
			&& (headOperator.getChainingStrategy() == ChainingStrategy.HEAD ||
			headOperator.getChainingStrategy() == ChainingStrategy.ALWAYS)
			&& (edge.getPartitioner() instanceof ForwardPartitioner)
			&& upStreamVertex.getParallelism() == downStreamVertex.getParallelism()
			&& streamGraph.isChainingEnabled();
	}
```



判断两个node是否可以链接在一起的条件主要是如下
* 下游的入边只有一条
* 上下游算子均不为空
* 上下游是否在同一个shareGrouop资源组中
* 下游的连接策略是always
* 下游的连接策略是head或者always（不为never)
* 边的分区分发方式是ForwardPartitioner
* 上下游的并行度一致
* 全局配置是可以链接的

createJobGraph()方法就是jobGraph进行配置的主要逻辑，如下
```java
private JobGraph createJobGraph() {

		//设置调度模式，采用的EAGER模式，既所有节点都是立即启动的
		// make sure that all vertices start immediately
		jobGraph.setScheduleMode(ScheduleMode.EAGER);

		// Generate deterministic hashes for the nodes in order to identify them across
		// submission iff they didn't change.
		// 遍历streamGraph，为每个node创建一个hash值，一个StreamNode的ID对应一个散列值。
		Map<Integer, byte[]> hashes = defaultStreamGraphHasher.traverseStreamGraphAndGenerateHashes(streamGraph);

		// Generate legacy version hashes for backwards compatibility
		// 生成旧版本的hash值，为了向后兼容
		List<Map<Integer, byte[]>> legacyHashes = new ArrayList<>(legacyStreamGraphHashers.size());
		for (StreamGraphHasher hasher : legacyStreamGraphHashers) {
			legacyHashes.add(hasher.traverseStreamGraphAndGenerateHashes(streamGraph));
		}

		Map<Integer, List<Tuple2<byte[], byte[]>>> chainedOperatorHashes = new HashMap<>();

		//最重要的函数，生成JobVertex，JobEdge等，并尽可能地将多个节点chain在一起
		setChaining(hashes, legacyHashes, chainedOperatorHashes);

		// 将每个JobVertex的入边集合也序列化到该JobVertex的StreamConfig中(出边集合已经在setChaining的时候写入了)
		setPhysicalEdges();

		//据group name，为每个 JobVertex 指定所属的 SlotSharingGroup,以及针对 Iteration的头尾设置  CoLocationGroup
		setSlotSharingAndCoLocation();

		//配置checkpoint
		configureCheckpointing();

		JobGraphGenerator.addUserArtifactEntries(streamGraph.getEnvironment().getCachedFiles(), jobGraph);

		// 设置ExecutionConfig
		// set the ExecutionConfig last when it has been finalized
		try {
			jobGraph.setExecutionConfig(streamGraph.getExecutionConfig());
		} catch (IOException e) {
			throw new IllegalConfigurationException("Could not serialize the ExecutionConfig." +
				"This indicates that non-serializable types (like custom serializers) were registered");
		}

		//返回转化好的jobGraph
		return jobGraph;
	}
```

### 核心函数setChaining
setChaining函数主要做的事情是将遍历所有的sourceNode，向下遍历，创建任务链和JobVertex顶点。从source开始遍历，将可以构成链的分组，然后每个组构成一个vertex顶点，并将上下游的vertex之间构成一个edge
```java
	/**
	 * 从source的node实例开始，创建任务链。 会递归的创建JobVertex. 从source开始遍历，
	 * 将可以构成链的分组，然后每个组构成一个vertex顶点，并将上下游的vertex之间构成一个edge
	 * Sets up task chains from the source {@link StreamNode} instances.
	 * <p>
	 * <p>This will recursively create all {@link JobVertex} instances.
	 */
	private void setChaining(Map<Integer, byte[]> hashes, List<Map<Integer, byte[]>> legacyHashes, Map<Integer, List<Tuple2<byte[], byte[]>>> chainedOperatorHashes) {
		for (Integer sourceNodeId : streamGraph.getSourceIDs()) {
			createChain(sourceNodeId, sourceNodeId, hashes, legacyHashes, 0, chainedOperatorHashes);
		}
	}
```
createChain主要的函数是如何通过source创建对应的链和定点以及出边。通过递归的方式，将所有可以链接在一起的node组成链，然后针对每个链的第一个Node建立vertx顶点。
```java
/**
	 * @param startNodeId           链开始的id
	 * @param currentNodeId         当前nodeId
	 * @param hashes
	 * @param legacyHashes
	 * @param chainIndex
	 * @param chainedOperatorHashes
	 * @return
	 */
	private List<StreamEdge> createChain(
		Integer startNodeId,
		Integer currentNodeId,
		Map<Integer, byte[]> hashes,
		List<Map<Integer, byte[]>> legacyHashes,
		int chainIndex,
		Map<Integer, List<Tuple2<byte[], byte[]>>> chainedOperatorHashes) {

		//是否已经创建的vertice，如果不存在就创建
		if (!builtVertices.contains(startNodeId)) {

			List<StreamEdge> transitiveOutEdges = new ArrayList<StreamEdge>();

			List<StreamEdge> chainableOutputs = new ArrayList<StreamEdge>();
			List<StreamEdge> nonChainableOutputs = new ArrayList<StreamEdge>();

			//获取node的出边，遍历
			for (StreamEdge outEdge : streamGraph.getStreamNode(currentNodeId).getOutEdges()) {
				// 两个StreamNode是否可以链接到一起执行的判断逻辑，如果是可以链接在一起的，加入可以链接的集合出边，如果不可以链接在一起，就加入到不可以链接的集合出边
				if (isChainable(outEdge, streamGraph)) {
					chainableOutputs.add(outEdge);
				} else {
					nonChainableOutputs.add(outEdge);
				}
			}

		//transitiveOutEdges存放了链路的出边，只有在nonChainable在add，在Chainable都递归到下一个node了。	//从每一个出边，开始遍历深搜。
			for (StreamEdge chainable : chainableOutputs) {
				transitiveOutEdges.addAll(
					createChain(startNodeId, chainable.getTargetId(), hashes, legacyHashes, chainIndex + 1, chainedOperatorHashes));
			}

			/**
			 * 对于每个不可连接的StreamEdge，将对于的StreamEdge就是当前链的一个输出StreamEdge，所以会添加到transitiveOutEdges这个集合中
			 * 然后递归调用其目标节点，注意，startNodeID变成了nonChainable这个StreamEdge的输出节点id，chainIndex也赋值为0，说明重新开始一条链的建立
			 */
			for (StreamEdge nonChainable : nonChainableOutputs) {
				transitiveOutEdges.add(nonChainable);
				createChain(nonChainable.getTargetId(), nonChainable.getTargetId(), hashes, legacyHashes, 0, chainedOperatorHashes);
			}

			List<Tuple2<byte[], byte[]>> operatorHashes =
				chainedOperatorHashes.computeIfAbsent(startNodeId, k -> new ArrayList<>());

			byte[] primaryHashBytes = hashes.get(currentNodeId);

			for (Map<Integer, byte[]> legacyHash : legacyHashes) {
				operatorHashes.add(new Tuple2<>(primaryHashBytes, legacyHash.get(currentNodeId)));
			}

			//为每个链创建名称
			chainedNames.put(currentNodeId, createChainedName(currentNodeId, chainableOutputs));
			//每个node都有分配的资源，如果在同一个链中，将资源合并
			chainedMinResources.put(currentNodeId, createChainedMinResources(currentNodeId, chainableOutputs));
			chainedPreferredResources.put(currentNodeId, createChainedPreferredResources(currentNodeId, chainableOutputs));

			//如果当前节点是这条链的起始点，创建一个JobVertex并返回一个StreamConfig，否则先创建一个空的 StreamConfig
			StreamConfig config = currentNodeId.equals(startNodeId)
				? createJobVertex(startNodeId, hashes, legacyHashes, chainedOperatorHashes)
				: new StreamConfig(new Configuration());

			setVertexConfig(currentNodeId, config, chainableOutputs, nonChainableOutputs);

			if (currentNodeId.equals(startNodeId)) {
				//如果是chain的起始节点。（不是chain的中间节点，会被标记成 chain start）
				config.setChainStart();
				config.setChainIndex(0);
				config.setOperatorName(streamGraph.getStreamNode(currentNodeId).getOperatorName());
				config.setOutEdgesInOrder(transitiveOutEdges);
				config.setOutEdges(streamGraph.getStreamNode(currentNodeId).getOutEdges());

				//将当前节点(headOfChain)与所有出边相，构建JobEdge连
				for (StreamEdge edge : transitiveOutEdges) {
					connect(startNodeId, edge);
				}

				config.setTransitiveChainedTaskConfigs(chainedConfigs.get(startNodeId));

			} else {

				//如果是 chain 中的子节点
				Map<Integer, StreamConfig> chainedConfs = chainedConfigs.get(startNodeId);

				if (chainedConfs == null) {
					chainedConfigs.put(startNodeId, new HashMap<Integer, StreamConfig>());
				}
				config.setChainIndex(chainIndex);
				StreamNode node = streamGraph.getStreamNode(currentNodeId);
				config.setOperatorName(node.getOperatorName());
				chainedConfigs.get(startNodeId).put(currentNodeId, config);
			}

			config.setOperatorID(new OperatorID(primaryHashBytes));

			//如果节点的输出StreamEdge已经为空，则说明是链的结尾
			if (chainableOutputs.isEmpty()) {
				config.setChainEnd();
			}
			return transitiveOutEdges;

		} else {
			return new ArrayList<>();
		}
	}
```

每个 JobVertex 都会对应一个可序列化的 StreamConfig, 用来发送给 JobManager 和 TaskManager。最后在 TaskManager 中起 Task 时,需要从这里面反序列化出所需要的配置信息, 其中就包括了含有用户代码的StreamOperator。

setChaining会对source调用createChain方法，该方法会递归调用下游节点，从而构建出node chains。createChain会分析当前节点的出边，根据Operator Chains中的chainable条件，将出边分成chainalbe和noChainable两类，并分别递归调用自身方法。之后会将StreamNode中的配置信息序列化到StreamConfig中。如果当前不是chain中的子节点，则会构建 JobVertex 和 JobEdge相连。如果是chain中的子节点，则会将StreamConfig添加到该chain的config集合中。一个node chains，除了 headOfChain node会生成对应的 JobVertex，其余的nodes都是以序列化的形式写入到StreamConfig中，并保存到headOfChain的 CHAINED_TASK_CONFIG 配置项中。


### flink任务调度

* Flink中的执行资源通过任务槽(Task Slots)定义。每个TaskManager都有一个或多个任务槽，每个槽都可以运行一个并行任务管道(pipeline)。管道由多个连续的任务组成，例如第n个MapFunction并行实例和第n个ReduceFunction并行实例。Flink经常并发地执行连续的任务：对于流程序，这在任何情况下都会发生，对于批处理程序，它也经常发生。
* 关于Flink调度，有两个非常重要的原则：
* * 1.同一个operator的各个subtask是不能呆在同一个SharedSlot中的，例如FlatMap[1]和FlatMap[2]是不能在同一个SharedSlot中的。
* * 2.Flink是按照拓扑顺序从Source一个个调度到Sink的。例如WordCount（Source并行度为1，其他并行度为2），那么调度的顺序依次是：Source -> FlatMap[1] -> FlatMap[2] -> KeyAgg->Sink[1] -> KeyAgg->Sink[2]。

![image](https://note.youdao.com/yws/api/personal/file/3137058979314CE8A93F70D857182DBF?method=download&shareKey=2db97d3217f62794683f4f9dcde154ee)


## Flink中Task间的数据传递


### 同一线程的Operator数据传递(同一个Task)


![image](https://note.youdao.com/yws/api/personal/file/D74FFBA07746402CA2672A42DB3FF925?method=download&shareKey=d730d08090d92224cf968549b0d84837)

在同一个线程中，在构成顶点的时候，会把数据相关的上下游数据放在同一个Task中，同一个chain。每个operation对应的ouput会有下一个operation的引用，直接调用下游operation的processElement实现数据的直接传递。

```java
static final class CopyingChainingOutput<T> extends ChainingOutput<T> {
```

简单介绍一下ChainingOutput和CopyingChainingOutput，ChainingOutput是CopyingChainingOutput的父类，两者的差别在于CopyingChainingOutput重写了pushToOperator方法，增加了这么一句代码
```
StreamRecord<T> copy = castRecord.copy(serializer.copy(castRecord.getValue())); 数据复制
```

下面是父类的详细代码
```java

	static class ChainingOutput<T> implements WatermarkGaugeExposingOutput<StreamRecord<T>> {

		protected final OneInputStreamOperator<T, ?> operator;
		protected final Counter numRecordsIn;
		protected final WatermarkGauge watermarkGauge = new WatermarkGauge();

		protected final StreamStatusProvider streamStatusProvider;
		
	
	
		@Override
		public void collect(StreamRecord<T> record) {
			if (this.outputTag != null) {
				// we are only responsible for emitting to the main input
				return;
			}

			pushToOperator(record);
		}
		
	
		protected <X> void pushToOperator(StreamRecord<X> record) {
			try {
				// we know that the given outputTag matches our OutputTag so the record
				// must be of the type that our operator expects.
				@SuppressWarnings("unchecked")
				StreamRecord<T> castRecord = (StreamRecord<T>) record;

				numRecordsIn.inc();
				operator.setKeyContextElement1(castRecord);
				operator.processElement(castRecord);
			}
			catch (Exception e) {
				throw new ExceptionInChainedOperatorException(e);
			}
		}
```

如果实体复用开启，那么使用ChainingOutput，否则使用CopyingChainingOutput。
```
containingTask.getExecutionConfig().isObjectReuseEnabled()
```
#### 为什么多个算子哪怕可以合为一条链也会造成一定的性能损耗？

默认实体复用是不开启的，而在多个算子中，是需要经过数据的copy。
```
StreamRecord<T> copy = castRecord.copy(serializer.copy(castRecord.getValue())); 数据复制
```

serializer会有多种类型，StringSerializer、TupleSerializer、PojoSerializer等等，String这种类型，copy方法会就是直接返回，对于Tuple、Pojo，Tuple会创建一个Tuple，将原来的数据set到新的Tuple中，如果是Pojo，会通过反射， 先将所有的属性设置为可以访问，屏蔽掉private的影响,然后根据字段对应的类型用对应的序列化器深度copy一个值.
```java
this.fields[i].setAccessible(true);
```

```java
            Object value = fields[i].get(from);
			if (value != null) {
					Object copy = fieldSerializers[i].copy(value);
					fields[i].set(target, copy);
			} else {
					fields[i].set(target, null);
			}
```
因此会出现序列化和逆序列化上的问题，因此合理的设置算子的层数，也是一种很重要的做法。



### 本地线程数据传递(同一个TM)

如果并行度一致，并且是forward，会被归在同一个operationChain，走上面的情况，如果不是，则会归属在不同的task中。不同task会出现两种情况，一种是task归属于同一个TM，一种是task归属于不同TM


不同个TM通过RecordWriter来发送数据到下游。

![image](https://note.youdao.com/yws/api/personal/file/DEC307A65DCB4008BF401E5C0EDEA534?method=download&shareKey=2e30476f270cad20ad2cedb16a00a4c3)

下游OneInputStreamTask.run()会通过死循环一直获取数据来消费
```java
while (running && inputProcessor.processInput()) {
			// all the work happens in the "processInput" method
		}
```

下游会通过wait的方式等待上游notify，然后从buffer中逆序列化成具体的数据交给userfunction去处理。

### 远程线程数据传递(不同TM)

![image](https://note.youdao.com/yws/api/personal/file/07BDD946C3E14C7DAD55C7438696A1E9?method=download&shareKey=c040cef54a82640144867cea05aa76bd)

与同一个TM不一样的地方在于SingleInputGate中的InputChannel，
同一个TM用的是LocalInputChannel，不同TM用的是RemoteInputChannel.

跟同一个TM差别在于是否真的有netty介入。

RPC通信基于Netty实现， 下图为Client端的RPC请求发送过程。PartitionRequestClient发出请求，交由Netty写到对应的socket。Netty读取Socket数据，解析Response后交由NetworkClientHandler处理。

![image](https://note.youdao.com/yws/api/personal/file/9C06B4E014704BF38E300533102F74DB?method=download&shareKey=2d36bc2d0e28e14631b70d00d3b5e91b)


## 通信机制和背压处理



 ### 物理传输
 
A.1→B.3、A.1→B.4 以及 A.2→B.3 和 A.2→B.4 的情况，如下图所示：

![image](https://note.youdao.com/yws/api/personal/file/6F5D2BE011BD493485719BF40FC8F5D4?method=download&shareKey=990790ecc03e7c10d3f5ef92bffda6b3)

每个子任务的结果称为结果分区，每个结果拆分到单独的子结果分区（ResultSubpartitions）中——每个逻辑通道有一个。Flink 不再处理单个记录，而是将一组序列化记录组装到网络缓冲区中。


### 造成背压场景

* 每当子任务的数据发送缓冲区耗尽时——数据驻留在 Subpartition 的缓冲区队列中或位于更底层的基于 Netty 的网络堆栈内，生产者就会被阻塞，无法继续发送数据，而受到反压。
* 接收器也是类似：Netty 收到任何数据都需要通过网络 Buffer 传递给 Flink。如果相应子任务的网络缓冲区中没有足够可用的网络 Buffer，Flink 将停止从该通道读取，直到 Buffer 可用。这将反压该多路复用上的所有发送子任务，因此也限制了其他接收子任务。

下图中子任务 B.4 过载了，它会对这条多路传输链路造成背压，还会阻止子任务 B.3 接收和处理新的缓存。

![image](https://note.youdao.com/yws/api/personal/file/C80951F0E65C4A219E951FE70386A8E6?method=download&shareKey=4fa4619890593c2886675e34caad4da4)

由于Flink是多线程执行task而不是多进程，因此在旧版的通信机制下，如果一个作业堵了，会导致其他作业一起中奖。
为了防止这种情况发生，Flink 1.5 引入了自己的流量控制机制。

### 基于信用的流量控制 


基于网络缓冲区的可用性实现.每个远程输入通道(RemoteInputChannel)现在都有自己的一组独占缓冲区(Exclusive buffer)，而不是只有一个共享的本地缓冲池(LocalBufferPool)。与之前不同，本地缓冲池中的缓冲区称为流动缓冲区(Floating buffer)，因为它们会在输出通道间流动并且可用于每个输入通道。

![image](https://note.youdao.com/yws/api/personal/file/E4AD271F5E604DD084A295BCD6428E29?method=download&shareKey=a8256273bfaac69aa25737442367db5c)


基于信用说简单点就是数据接收方会将自身的可用 Buffer 作为 Credit 告知数据发送方(1 buffer = 1 credit)。

每个 Subpartition 会跟踪下游接收端的 Credit(也就是可用于接收数据的 Buffer 数目)。只有在相应的通道(Channel)有 Credit 的时候 Flink 才会向更底层的网络协议栈发送数据(以 Buffer 为粒度)，并且每发送一个 Buffer 的数据，相应的通道上的 Credit 会减 1。

简单来说，每个Task就是一个仓库，每个仓库都有两个工人（浮动buffer），消费需要搬运的外来物件，每次搬运的时候，会跟外部通知现在可以承受多少的搬运（信用）。外部会告诉仓库外面有多少物件需要搬运（backlog），然后对应的仓库去找总部申请更多的员工（浮动缓冲区）。

### 背压处理

相比没有流量控制的接收器的背压机制，信用机制提供了更直接的控制逻辑：如果接收器能力不足，其可用信用将减到 0，并阻止发送方将缓存转发到较底层的网络栈上。这样只在这个逻辑信道上存在背压，并且不需要阻止从多路复用 TCP 信道读取内容。因此，其他接收器在处理可用缓存时就不受影响了。

![image](https://note.youdao.com/yws/api/personal/file/20A3A8689F2C433DA8B553D8D3EA5BE4?method=download&shareKey=0518c3a6cadf0478371daf1dff69d17b)

图中有两个地方和两个参数对应。

* Exclusive buffers：对应taskmanager.network.memory.buffers-per-channel。default为2，每个channel需要的独占buffer，一定要大于或者等于2.1个buffer用于接收数据，一个buffer用于序列化数据。
* buffer pool中的Floating buffers的个数：taskmanager.network.memory.floating-buffers-per-gate，default为8.在一个subtask中，会为每个下游task建立一个channel，每个channel中需要独占taskmanager.network.memory.buffers-per-channel个buffer。浮动缓冲区是基于backlog(子分区中的实时输出缓冲区)反馈分布的，可以帮助缓解由于子分区间数据分布不平衡而造成的反压力。接收器将使用它来请求适当数量的浮动缓冲区，以便更快处理 backlog。它将尝试获取与 backlog 大小一样多的浮动缓冲区，但有时并不会如意，可能只获取一点甚至获取不到缓冲。在节点和/或集群中机器数量较多的情况下，这个值应该增加，特别是在数据倾斜比较严重的时候。
 

### Reference

http://wuchong.me/blog/2016/05/03/flink-internals-overview/

http://wuchong.me/blog/2016/05/04/flink-internal-how-to-build-streamgraph/

https://zhuanlan.zhihu.com/p/22736103
