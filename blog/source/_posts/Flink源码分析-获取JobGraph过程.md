---
title: Flink源码分析 获取JobGraph过程
date: 2019-06-30 00:10:57
tags:
categories:
	- Flink
---

### 作业图(JobGraph)

作业图(JobGraph)是唯一被Flink的数据流引擎所识别的表述作业的数据结构，也正是这一共同的抽象体现了流处理和批处理在运行时的统一。

作业顶点(JobVertex)、中间数据集(IntermediateDataSet)、作业边(JobEdge)是组成JobGraph的基本元素。这三个对象彼此之间互为依赖：
* 一个JobVertex关联着若干个JobEdge作为输入端以及若干个IntermediateDataSet作为其生产的结果集；
* 一个IntermediateDataSet关联着一个JobVertex作为生产者以及若干个JobEdge作为消费者；
* 一个JobEdge关联着一个IntermediateDataSet可认为是源以及一个JobVertex可认为是目标消费者；


获取JobGraph是在生成streamGraph之后，核心的部分在 ctx.getClient().run。streamGraph流图是数据的流向图，每个算子都是一个node，在JobGraph中，是将具有一定条件的node连接在一起成为链，是资源规划层面的作业图，每个链有一个Vertex。
```java
@Override
	public JobExecutionResult execute(String jobName) throws Exception {
		Preconditions.checkNotNull(jobName, "Streaming Job name should not be null.");

		//获取程序对应的作业图
		StreamGraph streamGraph = this.getStreamGraph();
		streamGraph.setJobName(jobName);

		//清空所有算子
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
```

主要的方法是StreamGraph.getJobGraph
```java
/**
	 * Gets the assembled {@link JobGraph} with a given job id.
	 */
	@SuppressWarnings("deprecation")
	@Override
	public JobGraph getJobGraph(@Nullable JobID jobID) {
		// temporarily forbid checkpointing for iterative jobs
		if (isIterative() && checkpointConfig.isCheckpointingEnabled() && !checkpointConfig.isForceCheckpointing()) {
			throw new UnsupportedOperationException(
				"Checkpointing is currently not supported by default for iterative jobs, as we cannot guarantee exactly once semantics. "
					+ "State checkpoints happen normally, but records in-transit during the snapshot will be lost upon failure. "
					+ "\nThe user can force enable state checkpoints with the reduced guarantees by calling: env.enableCheckpointing(interval,true)");
		}

		return StreamingJobGraphGenerator.createJobGraph(this, jobID);
	}
```
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


判断两个node是否可以链接在一起的条件主要是如下
* 下游的入边只有一条
* 上下游算子均不为空
* 上下游是否在同一个shareGrouop资源组中
* 下游的连接策略是always
* 下游的连接策略是head或者always（不为never)
* 边的分区分发方式是ForwardPartitioner
* 上下游的并行度一致
* 全局配置是可以链接的

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

给链取名字的操作，创建链名称，如果不在同一条链上，返回操作符名称，如果在同一条且只有一条下游，xx -> xx
如果有两条下游  xx->(xx,xx)

```java
/**
	 * 创建链名称，如果不在同一条链上，返回操作符名称，如果在同一条且只有一条下游，xx -> xx
	 * 如果有两条下游  xx->(xx,xx)
	 *
	 * @param vertexID
	 * @param chainedOutputs
	 * @return
	 */
	private String createChainedName(Integer vertexID, List<StreamEdge> chainedOutputs) {
		String operatorName = streamGraph.getStreamNode(vertexID).getOperatorName();
		if (chainedOutputs.size() > 1) {
			List<String> outputChainedNames = new ArrayList<>();
			for (StreamEdge chainable : chainedOutputs) {
				outputChainedNames.add(chainedNames.get(chainable.getTargetId()));
			}
			return operatorName + " -> (" + StringUtils.join(outputChainedNames, ", ") + ")";
		} else if (chainedOutputs.size() == 1) {
			return operatorName + " -> " + chainedNames.get(chainedOutputs.get(0).getTargetId());
		} else {
			return operatorName;
		}
	}
```
connect是每个vertex与下一个vertex构建JobEdge的过程。
```java
/**
	 * 构建JobEdge
	 * @param headOfChain
	 * @param edge
	 */
	private void connect(Integer headOfChain, StreamEdge edge) {

		//将当前edge记录物理边界顺序集合中。 每条链路的出边，就是一个顶点的入边
		physicalEdgesInOrder.add(edge);

		Integer downStreamvertexID = edge.getTargetId();

		JobVertex headVertex = jobVertices.get(headOfChain);
		JobVertex downStreamVertex = jobVertices.get(downStreamvertexID);

		StreamConfig downStreamConfig = new StreamConfig(downStreamVertex.getConfiguration());

		downStreamConfig.setNumberOfInputs(downStreamConfig.getNumberOfInputs() + 1);

		StreamPartitioner<?> partitioner = edge.getPartitioner();
		JobEdge jobEdge;
		//分局分区方式构建Job边
		if (partitioner instanceof ForwardPartitioner || partitioner instanceof RescalePartitioner) {
			// 向前传递分区 or 可扩展分区
			jobEdge = downStreamVertex.connectNewDataSetAsInput(
				headVertex,
				DistributionPattern.POINTWISE,
				ResultPartitionType.PIPELINED_BOUNDED);
		} else {
			//其他分区
			jobEdge = downStreamVertex.connectNewDataSetAsInput(
				headVertex,
				DistributionPattern.ALL_TO_ALL,
				ResultPartitionType.PIPELINED_BOUNDED);
		}
		// set strategy name so that web interface can show it.
		jobEdge.setShipStrategyName(partitioner.toString());

		if (LOG.isDebugEnabled()) {
			LOG.debug("CONNECTED: {} - {} -> {}", partitioner.getClass().getSimpleName(),
				headOfChain, downStreamvertexID);
		}
	}
```

其中JobEdge是通过下游JobVertex的connectNewDataSetAsInput方法来创建的，在创建JobEdge的前，会先用上游JobVertex创建一个IntermediateDataSet实例，用来作为上游JobVertex的结果输出，然后作为JobEdge的输入，构建JobEdge实例，具体实现如下：
```java
public JobEdge connectNewDataSetAsInput(
			JobVertex input,
			DistributionPattern distPattern,
			ResultPartitionType partitionType) {

		// 创建输入JobVertex的输出数据集合
		IntermediateDataSet dataSet = input.createAndAddResultDataSet(partitionType);

		//构建JobEdge实例
		JobEdge edge = new JobEdge(dataSet, this, distPattern);
		//将JobEdge实例，作为当前JobVertex的输入
		this.inputs.add(edge);
		//设置中间结果集合dataSet的消费者是上面创建的JobEdge
		dataSet.addConsumer(edge);
		return edge;
	}
```
### Reference
https://blog.csdn.net/super_wj0820/article/details/81142710