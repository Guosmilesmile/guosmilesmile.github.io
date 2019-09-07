---
title: Flink源码解析 Flink中Task间的数据传递
date: 2019-09-06 23:12:05
tags:
categories:
	- Flink
---

![image](https://note.youdao.com/yws/api/personal/file/482B071BB9574F26A49F8B2C8245A085?method=download&shareKey=f91aa5fda263f74e7a6b199c5464f1a6)


## 数据传递


AbstractStreamOperator$CountingOutput.collect
```java

@Override
		public void collect(StreamRecord<OUT> record) {
			numRecordsOut.inc();
			output.collect(record);
		}
```

同一个TM和同一线程的Operator的区分在这里。如果是同一个TM

用到的是RecordWriterOutput.collect，如果是同一个线程的operator，用到的是OperatorChain$CopyingChainingOutput.collect

### 同一线程的Operator数据传递(同一个Task)


![image](https://note.youdao.com/yws/api/personal/file/D74FFBA07746402CA2672A42DB3FF925?method=download&shareKey=d730d08090d92224cf968549b0d84837)

allOperators中有算子和out的关系，在调用out（CopyingChainingOutput）的时候，会调用pushToOperator函数，这个函数内部会通过深拷贝复制出一个实体，在这个out中，存在一个属性，这个属性是下游的function，然后调用function函数进行计算并继续out出去。


##### 构建operation  chain

这个过程就是将整个operation chain构建出来，然后将CopyingChainingOutput中注入下游operation，形成当前operation包含CopyingChainingOutput，CopyingChainingOutput中有下游的operation。

实现数据从上游处理后，调用out.collect可以直接传递给下游的operation。


operatorChain = new OperatorChain<>(this, recordWriters);获取这个task的整个操作链和headOperator,初始化output.

```java


public OperatorChain(
			StreamTask<OUT, OP> containingTask,
			List<RecordWriter<SerializationDelegate<StreamRecord<OUT>>>> recordWriters) {
			
			
			

```


重点函数

```java
            //	创建操作链，并且将 设置链的leads
			// we create the chain of operators and grab the collector that leads into the chain
			this.chainEntryPoint = createOutputCollector(
				containingTask,
				configuration,
				chainedConfigs,
				userCodeClassloader,
				streamOutputMap,
				allOps);
```
OperatorChain
```java
private <IN, OUT> WatermarkGaugeExposingOutput<StreamRecord<IN>> createChainedOperator(
			StreamTask<?, ?> containingTask,
			StreamConfig operatorConfig,
			Map<Integer, StreamConfig> chainedConfigs,
			ClassLoader userCodeClassloader,
			Map<StreamEdge, RecordWriterOutput<?>> streamOutputs,
			List<StreamOperator<?>> allOperators,
			OutputTag<IN> outputTag) {
```


先获取同一个操作链中head的outputEdge，先通过递归的方式，创建每个operation对应的output，如下

```java
WatermarkGaugeExposingOutput<StreamRecord<OUT>> chainedOperatorOutput = createOutputCollector(
			containingTask,
			operatorConfig,
			chainedConfigs,
			userCodeClassloader,
			streamOutputs,
			allOperators);

		//获取对应的流式操作符
		// now create the operator and give it the output collector to write its output to
		OneInputStreamOperator<IN, OUT> chainedOperator = operatorConfig.getStreamOperator(userCodeClassloader);

		//将操作函数和out组成chainedOperator加入allOperators
		chainedOperator.setup(containingTask, operatorConfig, chainedOperatorOutput);

		allOperators.add(chainedOperator);
```


然后将操作函数和out组成chainedOperator加入allOperators.

```java
currentOperatorOutput = new CopyingChainingOutput<>(chainedOperator, inSerializer, outputTag, this);
```
通过如上方法，将chainedOperator（下一个operation）和CopyingChainingOutput绑定在一起，调用和CopyingChainingOutput的collect方法可以直接调用operation.processElements实现内部调用。。




#### 同一个线程的Operator中数据的复制
OperatorChain 
```java
StreamRecord<T> copy = castRecord.copy(serializer.copy(castRecord.getValue())); 数据复制
operator.processElement(copy);
```
serializer会有多种类型，StringSerializer、TupleSerializer、PojoSerializer等等，String这种类型，copy方法会就是直接返回，对于Tuple、Pojo，Tuple会创建一个Tuple，将原来的数据set到新的Tuple中，如果是Pojo，会通过反射，
先将所有的属性设置为可以访问，屏蔽掉private的影响
```java
this.fields[i].setAccessible(true);
```
然后根据字段对应的类型用对应的序列化器深度copy一个值

```java
            Object value = fields[i].get(from);
			if (value != null) {
					Object copy = fieldSerializers[i].copy(value);
					fields[i].set(target, copy);
			} else {
					fields[i].set(target, null);
			}
```



### 本地线程数据传递(同一个TM)

如果并行度一致，并且是forward，会被归在同一个operationChain，走上面的情况，如果不是，则会归属在不同的task中。不同task会出现两种情况，一种是task归属于同一个TM，一种是task归属于不同TM


不同个TM通过RecordWriter来发送数据到下游。


```
graph TB
A[RecordWriter]--包含-->B[ResultPartitionWriter]


```


#### RecordWriter初始化
在StreamTask.createRecordWriters. 某一个顶点，以后多个输出那么List<StreamEdge> outEdgesInOrder会是n，如果只有一条支流，那么就是1.

RecordWriter的初始化是在Task创建StreamTask的时候调用如下构建的

Task.java
```java
	invokable = loadAndInstantiateInvokable(userCodeClassLoader, nameOfInvokableClass, env);
```

StreamTask.java
```java
@VisibleForTesting
	public static <OUT> List<RecordWriter<SerializationDelegate<StreamRecord<OUT>>>> createRecordWriters(
			StreamConfig configuration,
			Environment environment) {
		List<RecordWriter<SerializationDelegate<StreamRecord<OUT>>>> recordWriters = new ArrayList<>();
		List<StreamEdge> outEdgesInOrder = configuration.getOutEdgesInOrder(environment.getUserClassLoader());
		Map<Integer, StreamConfig> chainedConfigs = configuration.getTransitiveChainedTaskConfigsWithSelf(environment.getUserClassLoader());

		for (int i = 0; i < outEdgesInOrder.size(); i++) {
			StreamEdge edge = outEdgesInOrder.get(i);
			recordWriters.add(
				createRecordWriter(
					edge,
					i,
					environment,
					environment.getTaskInfo().getTaskName(),
					chainedConfigs.get(edge.getSourceId()).getBufferTimeout()));
		}
		return recordWriters;
	}
	
```

如果下游有一个，就创建一RecordWriter，如果下游有两个（重复消费），就创建两个RecordWriter。

```java
	private static <OUT> RecordWriter<SerializationDelegate<StreamRecord<OUT>>> createRecordWriter(
			StreamEdge edge,
			int outputIndex,
			Environment environment,
			String taskName,
			long bufferTimeout) {
		@SuppressWarnings("unchecked")
		StreamPartitioner<OUT> outputPartitioner = (StreamPartitioner<OUT>) edge.getPartitioner();

		LOG.debug("Using partitioner {} for output {} of task {}", outputPartitioner, outputIndex, taskName);

		ResultPartitionWriter bufferWriter = environment.getWriter(outputIndex);

		// we initialize the partitioner here with the number of key groups (aka max. parallelism)
		if (outputPartitioner instanceof ConfigurableStreamPartitioner) {
			int numKeyGroups = bufferWriter.getNumTargetKeyGroups();
			if (0 < numKeyGroups) {
				((ConfigurableStreamPartitioner) outputPartitioner).configure(numKeyGroups);
			}
		}

		RecordWriter<SerializationDelegate<StreamRecord<OUT>>> output =
			RecordWriter.createRecordWriter(bufferWriter, outputPartitioner, bufferTimeout, taskName);
		output.setMetricGroup(environment.getMetricGroup().getIOMetricGroup());
		return output;
	}
```
重点在于如下代码，可以构建出下游有多少消费者
```java
ResultPartitionWriter bufferWriter = environment.getWriter(outputIndex);
```

environment的Writer是对应到Task中的producedPartitions，producedPartitions的初始化是在Task初始化的时候.

Task.java的构造函数.
```java
// Produced intermediate result partitions
		this.producedPartitions = new ResultPartition[resultPartitionDeploymentDescriptors.size()];

		int counter = 0;

		for (ResultPartitionDeploymentDescriptor desc: resultPartitionDeploymentDescriptors) {
			ResultPartitionID partitionId = new ResultPartitionID(desc.getPartitionId(), executionId);

			this.producedPartitions[counter] = new ResultPartition(
				taskNameWithSubtaskAndId,
				this,
				jobId,
				partitionId,
				desc.getPartitionType(),
				desc.getNumberOfSubpartitions(),
				desc.getMaxParallelism(),
				networkEnvironment.getResultPartitionManager(),
				resultPartitionConsumableNotifier,
				ioManager,
				desc.sendScheduleOrUpdateConsumersMessage());

			++counter;
		}
```
如果是同一个operationChain中，resultPartitionDeploymentDescriptors为0，如果是出现不同个task的情况，resultPartitionDeploymentDescriptors会是下游的个数（source(1并行度)--->map(3并行度)会有一个resultPartitionDeploymentDescriptors会是下游操作类型的个数）

这个时候就初始化好了ResultPartition，如果下游的并行度为3，那么该数组为3，对应三个ResultPartition。




#### resultPartitionDeploymentDescriptors怎么来的？

resultPartitionDeploymentDescriptors是TaskDeploymenentDescriptor的一部分信息，TaskDeploymenentDescriptor包含了部署一个task在一个taskManger中的所有信息。






#### RecordWriter传递数据

RecordWriterOutput.collect

```java
    @Override
	public void collect(StreamRecord<OUT> record) {
		if (this.outputTag != null) {
			// we are only responsible for emitting to the main input
			return;
		}

		pushToRecordWriter(record);
	}
	
	private <X> void pushToRecordWriter(StreamRecord<X> record) {
		serializationDelegate.setInstance(record);

		try {
			recordWriter.emit(serializationDelegate);
		}
		catch (Exception e) {
			throw new RuntimeException(e.getMessage(), e);
		}
	}
```

RecordWriter

```java
    public void emit(T record) throws IOException, InterruptedException {
		checkErroneous();
		emit(record, channelSelector.selectChannel(record));
	}
	
	private void emit(T record, int targetChannel) throws IOException, InterruptedException {
		serializer.serializeRecord(record);

		if (copyFromSerializerToTargetChannel(targetChannel)) {
			serializer.prune();
		}
	}

```

copyFromSerializerToTargetChannel这个会将数据发到resultSubpartition,传递到buffer中，再针对是否是本地数据进行操作，如果是不同TM，就会通过rpc发送，如果是同一个TM，会放在IG中，通知下游来获取

```java
serializer.serializeRecord(record);
```
数据序列化会调用SerializationDelegate.write,
```java
@Override
	public void write(DataOutputView out) throws IOException {
		this.serializer.serialize(this.instance, out);
	}
````
实际调用了StreamElementSerializer.serialize

```java
@Override
	public void serialize(StreamElement value, DataOutputView target) throws IOException {
		if (value.isRecord()) {
			StreamRecord<T> record = value.asRecord();

			if (record.hasTimestamp()) {
				target.write(TAG_REC_WITH_TIMESTAMP);
				target.writeLong(record.getTimestamp());
			} else {
				target.write(TAG_REC_WITHOUT_TIMESTAMP);
			}
			typeSerializer.serialize(record.getValue(), target);
		}
		else if (value.isWatermark()) {
			target.write(TAG_WATERMARK);
			target.writeLong(value.asWatermark().getTimestamp());
		}
		else if (value.isStreamStatus()) {
			target.write(TAG_STREAM_STATUS);
			target.writeInt(value.asStreamStatus().getStatus());
		}
		else if (value.isLatencyMarker()) {
			target.write(TAG_LATENCY_MARKER);
			target.writeLong(value.asLatencyMarker().getMarkedTime());
			target.writeLong(value.asLatencyMarker().getOperatorId().getLowerPart());
			target.writeLong(value.asLatencyMarker().getOperatorId().getUpperPart());
			target.writeInt(value.asLatencyMarker().getSubtaskIndex());
		}
		else {
			throw new RuntimeException();
		}
	}

```
针对数据还是waterMark或者其他状态数据进行处理。如果是数据，先判断数据是否有时间，调用这个数据对应类型的序列化器（TupleSerializer）的serialize方法，将数据写成buffer转成DataOutputView实体吗，变成序列化数据.


然后在RecordWriteer.java中的emit方法调用copyFromSerializerToTargetChannel将序列化的数据发送出去
```java
private void emit(T record, int targetChannel) throws IOException, InterruptedException {
		serializer.serializeRecord(record);

		if (copyFromSerializerToTargetChannel(targetChannel)) {
			serializer.prune();
		}
	}
```
将序列化数据写到buffer中
```java
private boolean copyFromSerializerToTargetChannel(int targetChannel) throws IOException, InterruptedException {
		// We should reset the initial position of the intermediate serialization buffer before
		// copying, so the serialization results can be copied to multiple target buffers.
		serializer.reset();

		boolean pruneTriggered = false;
		BufferBuilder bufferBuilder = getBufferBuilder(targetChannel);
		SerializationResult result = serializer.copyToBufferBuilder(bufferBuilder);
		while (result.isFullBuffer()) {
			numBytesOut.inc(bufferBuilder.finish());
			numBuffersOut.inc();

			// If this was a full record, we are done. Not breaking out of the loop at this point
			// will lead to another buffer request before breaking out (that would not be a
			// problem per se, but it can lead to stalls in the pipeline).
			if (result.isFullRecord()) {
				pruneTriggered = true;
				bufferBuilders[targetChannel] = Optional.empty();
				break;
			}

			bufferBuilder = requestNewBufferBuilder(targetChannel);
			result = serializer.copyToBufferBuilder(bufferBuilder);
		}
		checkState(!serializer.hasSerializedData(), "All data should be written at once");

		if (flushAlways) {
			targetPartition.flush(targetChannel);
		}
		return pruneTriggered;
	}
```
![image](https://note.youdao.com/yws/api/personal/file/DEC307A65DCB4008BF401E5C0EDEA534?method=download&shareKey=2e30476f270cad20ad2cedb16a00a4c3)


下游OneInputStreamTask.run()会通过死循环一直获取数据来消费
```java
while (running && inputProcessor.processInput()) {
			// all the work happens in the "processInput" method
		}
```
inputProcessor对应的是StreamInputProcessor，中间有一个变量为inputGate.
```java
DeserializationResult result = currentRecordDeserializer.getNextRecord(deserializationDelegate);
```
上面的代码会从buffer中逆序列化成具体的数据，然后交给userfunction处理..

buffer的获取，是通过如下先获取buffer，再将buffer放入currentRecordDeserializer，调用上面的currentRecordDeserializer.getNextRecord(deserializationDelegate);将buffer中解析到的数据逆序列化成deserializationDelegate。
```
final BufferOrEvent bufferOrEvent = barrierHandler.getNextNonBlocked();
```
```java
currentRecordDeserializer.setNextBuffer(bufferOrEvent.getBuffer());
```

#### 如何从上游中获取buffer

StreamInputProcessor
```java
	final BufferOrEvent bufferOrEvent = barrierHandler.getNextNonBlocked();
```

实际调用BarrierTrack.getNextNonBlocked()

```java
Optional<BufferOrEvent> next = inputGate.getNextBufferOrEvent();

```

SingleInputGate
```java
@Override
	public Optional<BufferOrEvent> getNextBufferOrEvent() throws IOException, InterruptedException {
		return getNextBufferOrEvent(true);
	}

private Optional<BufferOrEvent> getNextBufferOrEvent(boolean blocking) throws IOException, InterruptedException {

················
		if (blocking) {
			inputChannelsWithData.wait();
		}
					

·····················	

}
```

这个地方会wait，等待上游的ResultSubpartition去notify




### 远程线程数据传递(不同TM)

![image](https://note.youdao.com/yws/api/personal/file/07BDD946C3E14C7DAD55C7438696A1E9?method=download&shareKey=c040cef54a82640144867cea05aa76bd)

与同一个TM不一样的地方在于SingleInputGate中的InputChannel，
同一个TM用的是LocalInputChannel，不同TM用的是RemoteInputChannel.

RemoteInputCahnnel获取数据

```java
@Override
	Optional<BufferAndAvailability> getNextBuffer() throws IOException {
		checkState(!isReleased.get(), "Queried for a buffer after channel has been closed.");
		checkState(partitionRequestClient != null, "Queried for a buffer before requesting a queue.");

		checkError();

		final Buffer next;
		final boolean moreAvailable;

		synchronized (receivedBuffers) {
			next = receivedBuffers.poll();
			moreAvailable = !receivedBuffers.isEmpty();
		}

		numBytesIn.inc(next.getSizeUnsafe());
		numBuffersIn.inc();
		return Optional.of(new BufferAndAvailability(next, moreAvailable, getSenderBacklog()));
	}
```

通过ArrayDeque<Buffer> 的poll方法，等待通知