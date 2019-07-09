---
title: Flink源码分析 获取StreamGraph的过程
date: 2019-06-23 21:37:00
tags:
categories:
	- Flink
---
```
env.execute();
```

#### StreamContextEnvironment

```java
public JobExecutionResult execute(String jobName) throws Exception {
   Preconditions.checkNotNull("Streaming Job name should not be null.");
   StreamGraph streamGraph = this.getStreamGraph();
   streamGraph.setJobName(jobName);
   transformations.clear();
   // execute the programs
   if (ctx instanceof DetachedEnvironment) {
      LOG.warn("Job was executed in detached mode, the results will be available on completion.");
      ((DetachedEnvironment) ctx).setDetachedPlan(streamGraph);
      return DetachedEnvironment.DetachedJobExecutionResult.INSTANCE;
   } else {
      return ctx
         .getClient()
         .run(streamGraph, ctx.getJars(), ctx.getClasspaths(), ctx.getUserCodeClassLoader(), ctx.getSavepointRestoreSettings())
         .getJobExecutionResult();
   }
}
```
关键在于获取流计划
```java
StreamGraph streamGraph = this.getStreamGraph();
```

```java
public StreamGraph getStreamGraph() {
		if (transformations.size() <= 0) {
			throw new IllegalStateException("No operators defined in streaming topology. Cannot execute.");
		}
		return StreamGraphGenerator.generate(this, transformations);
	}
```
transformations这个列表就是在对数据流的处理过程中，会将flatMap、reduce这些转换操作对应的StreamTransformation保存下来的列表，根据对数据流做的转换操作.
* 表示flatMap操作的OneInputTransformation对象，其input属性指向的是数据源的转换SourceTransformation。 
* 表示reduce操作的OneInputTransformation对象，其input属性指向的是表示keyBy的转换PartitionTransformation，而PartitionTransformation的input属性指向的是flatMap的转换OneInputTransformation； 
* sink操作对应的SinkTransformation对象，其input属性指向的是reduce转化的OneInputTransformation对象。

```java
	/**
	 * Generates a {@code StreamGraph} by traversing the graph of {@code StreamTransformations}
	 * starting from the given transformations.
	 *
	 * @param env The {@code StreamExecutionEnvironment} that is used to set some parameters of the
	 *            job  evnironment这个入参用来设定作业的参数
	 * @param transformations The transformations starting from which to transform the graph  从算子的集合转为图
	 *
	 * @return The generated {@code StreamGraph}
	 */
	public static StreamGraph generate(StreamExecutionEnvironment env, List<StreamTransformation<?>> transformations) {
		return new StreamGraphGenerator(env).generateInternal(transformations);
	}
	
```


这个构造函数是private的，所以StreamGraphGenerator的实例构造只能通过其静态的generate方法。另外在构造函数中，初始化了一个StreamGraph实例，并设置了一些属性值，然后给env赋值，并初始化alreadyTransformed为一个空map。

```java
	/**
	 * This starts the actual transformation, beginning from the sinks.
	 * 从每个sink开始遍历，将每个转换算子转进行转换，返回一个图
	 */
	private StreamGraph generateInternal(List<StreamTransformation<?>> transformations) {
		for (StreamTransformation<?> transformation: transformations) {
			transform(transformation);
		}
		return streamGraph;
	}
```

从上面可以看出，就是遍历每一个操作，转为node，生成图。

#### transform
针对每一个操作，通过对应的转换函数，将这个算子生成对应的node，加入到图中，以id为key，node为value。如果是source，直接加入，如果是其他的，从这个点开始往上遍历，递归的将一个个点加入图中。并且将该点和上一个点，沟通一条边
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
这就是针对一个个算子的转换过程，先从source看起，在看operation和sink。

##### source，transformSource

先获取该算子属于哪个solt组，然后将souce信息注册到流图中。
```java

	/**
	 * Transforms a {@code SourceTransformation}.
	 */
	private <T> Collection<Integer> transformSource(SourceTransformation<T> source) {
		//基于用户设定的slolt组和input来判断一个算子归属于那个slot
		String slotSharingGroup = determineSlotSharingGroup(source.getSlotSharingGroup(), Collections.emptyList());

		
		//在流图中添加source信息
		//CoLocationGroupKey 是一个尚未开放不用一定会对外的特性，如果算子拥有相同的groupkey，会被置于同一个solt中
		streamGraph.addSource(source.getId(),
				slotSharingGroup,
				source.getCoLocationGroupKey(),
				source.getOperator(),
				null,
				source.getOutputType(),
				"Source: " + source.getName());
		if (source.getOperator().getUserFunction() instanceof InputFormatSourceFunction) {
			InputFormatSourceFunction<T> fs = (InputFormatSourceFunction<T>) source.getOperator().getUserFunction();
			streamGraph.setInputFormat(source.getId(), fs.getFormat());
		}
		streamGraph.setParallelism(source.getId(), source.getParallelism());
		streamGraph.setMaxParallelism(source.getId(), source.getMaxParallelism());
		return Collections.singleton(source.getId());
	}
```

核心在于streamGraph.addSource这个方法,这个方法只干了两件事，第一件事添加算子，第二件是加入到source集合中。

```java
    
    public <IN, OUT> void addSource(Integer vertexID,
		String slotSharingGroup,
		@Nullable String coLocationGroup,
		StreamOperator<OUT> operatorObject,
		TypeInformation<IN> inTypeInfo,
		TypeInformation<OUT> outTypeInfo,
		String operatorName) {
		addOperator(vertexID, slotSharingGroup, coLocationGroup, operatorObject, inTypeInfo, outTypeInfo, operatorName);
		sources.add(vertexID);
	}
```

添加操作符节点的逻辑，这个函数最主要的操作是添加了一个node。
```java
public <IN, OUT> void addOperator(
			Integer vertexID,
			String slotSharingGroup,
			@Nullable String coLocationGroup,
			StreamOperator<OUT> operatorObject,
			TypeInformation<IN> inTypeInfo,
			TypeInformation<OUT> outTypeInfo,
			String operatorName) {

        //根据算子的不同类型, 决定不同的节点类, 进行节点的构造
		if (operatorObject instanceof StoppableStreamSource) {
			//可以停止的source流，转成一个task,添加一个node
			addNode(vertexID, slotSharingGroup, coLocationGroup, StoppableSourceStreamTask.class, operatorObject, operatorName);
		} else if (operatorObject instanceof StreamSource) {
			//可以source流，转成一个task,添加一个node
			addNode(vertexID, slotSharingGroup, coLocationGroup, SourceStreamTask.class, operatorObject, operatorName);
		} else {
			//others，转成一个task,添加一个node
			addNode(vertexID, slotSharingGroup, coLocationGroup, OneInputStreamTask.class, operatorObject, operatorName);
		}

		//各种序列化操作
		TypeSerializer<IN> inSerializer = inTypeInfo != null && !(inTypeInfo instanceof MissingTypeInfo) ? inTypeInfo.createSerializer(executionConfig) : null;

		TypeSerializer<OUT> outSerializer = outTypeInfo != null && !(outTypeInfo instanceof MissingTypeInfo) ? outTypeInfo.createSerializer(executionConfig) : null;

		setSerializers(vertexID, inSerializer, null, outSerializer);
        
        //根据操作符类型, 进行输出数据类型设置
		if (operatorObject instanceof OutputTypeConfigurable && outTypeInfo != null) {
			@SuppressWarnings("unchecked")
			OutputTypeConfigurable<OUT> outputTypeConfigurable = (OutputTypeConfigurable<OUT>) operatorObject;
			// sets the output type which must be know at StreamGraph creation time
			outputTypeConfigurable.setOutputType(outTypeInfo, executionConfig);
		}

		if (operatorObject instanceof InputTypeConfigurable) {
			InputTypeConfigurable inputTypeConfigurable = (InputTypeConfigurable) operatorObject;
			inputTypeConfigurable.setInputType(inTypeInfo, executionConfig);
		}

		if (LOG.isDebugEnabled()) {
			LOG.debug("Vertex: {}", vertexID);
		}
	}
```

添加node方法，新建了一个StreamNode实例作为新的节点，然后保存到streamNodes这个map中，key是节点id，value就是节点StreamNode
```java
        protected StreamNode addNode(Integer vertexID,
		String slotSharingGroup,
		@Nullable String coLocationGroup,
		Class<? extends AbstractInvokable> vertexClass,
		StreamOperator<?> operatorObject,
		String operatorName) {

		//校验添加新节点的id, 与已添加的节点id, 是否有重复, 如果有, 则抛出异常
		if (streamNodes.containsKey(vertexID)) {
			throw new RuntimeException("Duplicate vertexID " + vertexID);
		}

		//新建一个node节点
		StreamNode vertex = new StreamNode(environment,
			vertexID,
			slotSharingGroup,
			coLocationGroup,
			operatorObject,
			operatorName,
			new ArrayList<OutputSelector<?>>(),
			vertexClass);

		// 将新构建的节点保存记录
		streamNodes.put(vertexID, vertex);

		return vertex;
	}
```

到这里，数据源节点就添加到了streamGraph中。

#### flatmap类型的operation--transformOneInputTransform
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





### addEdge 
```java
public void addEdge(Integer upStreamVertexID, Integer downStreamVertexID, int typeNumber) {
		addEdgeInternal(upStreamVertexID,
				downStreamVertexID,
				typeNumber,
				null,
				new ArrayList<String>(),
				null);

	}

```
核心方法在addEdgeInternal方法.该方法中，会构建一个StreamEdge。StreamEdge是用来描述流拓扑中的一个边界，其有对一个的源StreamNode和目标StreamNode，以及数据在源到目标直接转发时，进行的分区与select等操作的逻辑。接下来看StreamEdge的新增逻辑。
```java
private void addEdgeInternal(Integer upStreamVertexID,
			Integer downStreamVertexID,
			int typeNumber,
			StreamPartitioner<?> partitioner,
			List<String> outputNames,
			OutputTag outputTag) {

		//  虚拟节点----分区信息等等
		if (virtualSideOutputNodes.containsKey(upStreamVertexID)) {
			int virtualId = upStreamVertexID;
			upStreamVertexID = virtualSideOutputNodes.get(virtualId).f0;
			if (outputTag == null) {
				outputTag = virtualSideOutputNodes.get(virtualId).f1;
			}
			addEdgeInternal(upStreamVertexID, downStreamVertexID, typeNumber, partitioner, null, outputTag);
		} else if (virtualSelectNodes.containsKey(upStreamVertexID)) {
			int virtualId = upStreamVertexID;
			upStreamVertexID = virtualSelectNodes.get(virtualId).f0;
			if (outputNames.isEmpty()) {
				// selections that happen downstream override earlier selections
				outputNames = virtualSelectNodes.get(virtualId).f1;
			}
			addEdgeInternal(upStreamVertexID, downStreamVertexID, typeNumber, partitioner, outputNames, outputTag);
		} else if (virtualPartitionNodes.containsKey(upStreamVertexID)) {
			int virtualId = upStreamVertexID;
			upStreamVertexID = virtualPartitionNodes.get(virtualId).f0;
			if (partitioner == null) {
				partitioner = virtualPartitionNodes.get(virtualId).f1;
			}
			addEdgeInternal(upStreamVertexID, downStreamVertexID, typeNumber, partitioner, outputNames, outputTag);
		} else {
			//根据id，获取上下游两个node节点
			StreamNode upstreamNode = getStreamNode(upStreamVertexID);
			StreamNode downstreamNode = getStreamNode(downStreamVertexID);

			//如果没有指定分区器，并且上游节点和下游节点的操作符的并行度也是一样的话，就采用forward分区，否则，采用rebalance分区
			// If no partitioner was specified and the parallelism of upstream and downstream
			// operator matches use forward partitioning, use rebalance otherwise.
			if (partitioner == null && upstreamNode.getParallelism() == downstreamNode.getParallelism()) {
				partitioner = new ForwardPartitioner<Object>();
			} else if (partitioner == null) {
				partitioner = new RebalancePartitioner<Object>();
			}

			//如果是Forward分区, 而上下游的并行度不一致, 则抛异常, 这里是进行双重校验
			if (partitioner instanceof ForwardPartitioner) {
				if (upstreamNode.getParallelism() != downstreamNode.getParallelism()) {
					throw new UnsupportedOperationException("Forward partitioning does not allow " +
							"change of parallelism. Upstream operation: " + upstreamNode + " parallelism: " + upstreamNode.getParallelism() +
							", downstream operation: " + downstreamNode + " parallelism: " + downstreamNode.getParallelism() +
							" You must use another partitioning strategy, such as broadcast, rebalance, shuffle or global.");
				}
			}

			//新建StreamEdge实例
			StreamEdge edge = new StreamEdge(upstreamNode, downstreamNode, typeNumber, outputNames, partitioner, outputTag);

			//将新建的StreamEdge添加到源节点的输出边界集合中，目标节点的输入边界集合中
			getStreamNode(edge.getSourceId()).addOutEdge(edge);
			getStreamNode(edge.getTargetId()).addInEdge(edge);
		}
	}

```

#### keyby和reduce。
reduce操作对应StreamTransformation算子，keyby这个操作，并没有将operation加入到算子集合中，keyby只算分区，不算操作。keyby这个分区在图的构成，是在reduce递归加入图的时候加入生成节点加入图的。

keyby对应的算子是PartitionTransformation，是一个分区操作.需要在流图中创建一个虚拟节点，维护分区信息。在virtualPartitionNodes这个map中就新增映射：虚拟节点id ——> [上游节点id, partitioner]。



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



reduce自身的StreamTransformation子类实例，因为也是OneInputTransformation实例，跟flatmap类型相似，不同点在于构建StreamEdge，区别在StreamGrap.addEdgeInternal

```java

        if (virtualPartitionNodes.containsKey(upStreamVertexID)) {
			int virtualId = upStreamVertexID;
			//将上游id替换长keyby上游的id，因为keyby是虚拟节点
			upStreamVertexID = virtualPartitionNodes.get(virtualId).f0;
			if (partitioner == null) {
				partitioner = virtualPartitionNodes.get(virtualId).f1;
			}
			
			//并且在构成edge的时候将keyBy对应的分区器设置给partitioner变量
			addEdgeInternal(upStreamVertexID, downStreamVertexID, typeNumber, partitioner, outputNames, outputTag);
		}
```

这样就在flatMap和reduce对应的StreamNode之间构建了一个StreamEdge，且该StreamEdge中包含了keyBy转换中设置的分区器。

### Sink---transformSink

```java
/**
	 * Transforms a {@code SourceTransformation}.
	 */
	private <T> Collection<Integer> transformSink(SinkTransformation<T> sink) {

		//递归转换上游算子，获取上游的id
		Collection<Integer> inputIds = transform(sink.getInput());

		//根据上游和本算子的slot名称，计算归属的slot组
		String slotSharingGroup = determineSlotSharingGroup(sink.getSlotSharingGroup(), inputIds);

		//添加sink
		streamGraph.addSink(sink.getId(),
				slotSharingGroup,
				sink.getCoLocationGroupKey(),
				sink.getOperator(),
				sink.getInput().getOutputType(),
				null,
				"Sink: " + sink.getName());

		streamGraph.setParallelism(sink.getId(), sink.getParallelism());
		streamGraph.setMaxParallelism(sink.getId(), sink.getMaxParallelism());

		//根据上游id和sink的id，构建边
		for (Integer inputId: inputIds) {
			streamGraph.addEdge(inputId,
					sink.getId(),
					0
			);
		}
		
		if (sink.getStateKeySelector() != null) {
			TypeSerializer<?> keySerializer = sink.getStateKeyType().createSerializer(env.getConfig());
			streamGraph.setOneInputStateKey(sink.getId(), sink.getStateKeySelector(), keySerializer);
		}

		return Collections.emptyList();
	}

```
streamGraph.addSink这个方法内部的方法和addSource一致，添加算子，以及将sinkid加入到sink中.只要是真实的数据操作算子，都会经过addOperator这个方法，只是source和sink会单独将id加入一个集合中。（虚拟节点相关的例如PartitionTransformation就不会调用addOperator）
```java
public <IN, OUT> void addSink(Integer vertexID,
		String slotSharingGroup,
		@Nullable String coLocationGroup,
		StreamOperator<OUT> operatorObject,
		TypeInformation<IN> inTypeInfo,
		TypeInformation<OUT> outTypeInfo,
		String operatorName) {
		addOperator(vertexID, slotSharingGroup, coLocationGroup, operatorObject, inTypeInfo, outTypeInfo, operatorName);
		sinks.add(vertexID);
	}
```

### 构成的json图
将上述json字符串放如到 http://flink.apache.org/visualizer/ ，可以转化为图形表示
```json
{
	"nodes": [{
		"id": 1,
		"type": "Source: Socket Stream",
		"pact": "Data Source",
		"contents": "Source: Socket Stream",
		"parallelism": 1
	}, {
		"id": 2,
		"type": "Flat Map",
		"pact": "Operator",
		"contents": "Flat Map",
		"parallelism": 1,
		"predecessors": [{
			"id": 1,
			"ship_strategy": "FORWARD",
			"side": "second"
		}]
	}, {
		"id": 4,
		"type": "TriggerWindow(SlidingProcessingTimeWindows(5000, 1000), ReducingStateDescriptor{serializer=org.apache.flink.api.java.typeutils.runtime.PojoSerializer@e982bfcb, reduceFunction=com.mogujie.function.test.flink.SocketWindowWordCount$1@272ed83b}, ProcessingTimeTrigger(), WindowedStream.reduce(WindowedStream.java:241))",
		"pact": "Operator",
		"contents": "TriggerWindow(SlidingProcessingTimeWindows(5000, 1000), ReducingStateDescriptor{serializer=org.apache.flink.api.java.typeutils.runtime.PojoSerializer@e982bfcb, reduceFunction=com.mogujie.function.test.flink.SocketWindowWordCount$1@272ed83b}, ProcessingTimeTrigger(), WindowedStream.reduce(WindowedStream.java:241))",
		"parallelism": 1,
		"predecessors": [{
			"id": 2,
			"ship_strategy": "HASH",
			"side": "second"
		}]
	}, {
		"id": 5,
		"type": "Sink: Unnamed",
		"pact": "Data Sink",
		"contents": "Sink: Unnamed",
		"parallelism": 1,
		"predecessors": [{
			"id": 4,
			"ship_strategy": "FORWARD",
			"side": "second"
		}]
	}]
}
```

### Reference 
https://blog.csdn.net/qq_21653785/article/details/79499127