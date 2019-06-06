---
title: Flink operattion算子源码解析operattion算子源码解析
date: 2019-06-06 21:14:14
tags:
categories:
    - 流式计算 
    - Flink
---


#### FlatMap为例

```java
public <R> SingleOutputStreamOperator<R> flatMap(FlatMapFunction<T, R> flatMapper) {
   TypeInformation<R> outType = TypeExtractor.getFlatMapReturnTypes(clean(flatMapper),
         getType(), Utils.getCallLocationName(), true);
   /** 根据传入的flatMapper这个Function，构建StreamFlatMap这个StreamOperator的具体子类实例 */
   return transform("Flat Map", outType, new StreamFlatMap<>(clean(flatMapper)));
}
```

```java
/**
	 * Method for passing user defined operators along with the type
	 * information that will transform the DataStream.
	 *
	 * @param operatorName
	 *            name of the operator, for logging purposes
	 * @param outTypeInfo
	 *            the output type of the operator
	 * @param operator
	 *            the object containing the transformation logic
	 * @param <R>
	 *            type of the return stream
	 * @return the data stream constructed
	 */
	@PublicEvolving
	public <R> SingleOutputStreamOperator<R> transform(String operatorName, TypeInformation<R> outTypeInfo, OneInputStreamOperator<T, R> operator) {

		// read the output type of the input Transform to coax out errors about MissingTypeInfo
		/** 读取输入转换的输出类型, 如果是MissingTypeInfo, 则及时抛出异常, 终止操作 */
		transformation.getOutputType();

		OneInputTransformation<T, R> resultTransform = new OneInputTransformation<>(
				this.transformation,
				operatorName,
				operator,
				outTypeInfo,
				environment.getParallelism());

		@SuppressWarnings({ "unchecked", "rawtypes" })
		SingleOutputStreamOperator<R> returnStream = new SingleOutputStreamOperator(environment, resultTransform);

		getExecutionEnvironment().addOperator(resultTransform);

		return returnStream;
	}
```

上述逻辑中，除了构建出了SingleOutputStreamOperator这个实例为并返回外，还有一句代码：
```java
getExecutionEnvironment().addOperator(resultTransform);
```
就是将上述构建的OneInputTransFormation的实例，添加到了StreamExecutionEnvironment的属性transformations这个类型为List.

### keyBy转换

```java
	/**
	 * It creates a new {@link KeyedStream} that uses the provided key for partitioning
	 * its operator states.
	 *
	 * @param key
	 *            The KeySelector to be used for extracting the key for partitioning
	 * @return The {@link DataStream} with partitioned state (i.e. KeyedStream)
	 */
	public <K> KeyedStream<T, K> keyBy(KeySelector<T, K> key) {
		Preconditions.checkNotNull(key);
		return new KeyedStream<>(this, clean(key));
	}
```
KeyedStream的构造函数,先基于keySelector构造了一个KeyGroupStreamPartitioner的实例，再进一步构造了一个PartitionTransformation实例。
```java
	/**
	 * Creates a new {@link KeyedStream} using the given {@link KeySelector}
	 * to partition operator state by key.
	 *
	 * @param dataStream
	 *            Base stream of data
	 * @param keySelector
	 *            Function for determining state partitions
	 */
	public KeyedStream(DataStream<T> dataStream, KeySelector<T, KEY> keySelector, TypeInformation<KEY> keyType) {
		this(
			dataStream,
			new PartitionTransformation<>(
				dataStream.getTransformation(),
				new KeyGroupStreamPartitioner<>(keySelector, StreamGraphGenerator.DEFAULT_LOWER_BOUND_MAX_PARALLELISM)),
			keySelector,
			keyType);
	}
```

##### KeyGroupStreamPartitioner
```java
	@Override
	public int selectChannel(SerializationDelegate<StreamRecord<T>> record) {
		K key;
		try {
			key = keySelector.getKey(record.getInstance().getValue());
		} catch (Exception e) {
			throw new RuntimeException("Could not extract key from " + record.getInstance().getValue(), e);
		}
		return KeyGroupRangeAssignment.assignKeyToParallelOperator(key, maxParallelism, numberOfChannels);
	}
```
在这个partitioner中会选择record下游的channel id。
```java
	/** 
	 * 计算下游选择选择的channel id
	 * Assigns the given key to a parallel operator index.
	 *
	 * @param key the key to assign
	 * @param maxParallelism the maximum supported parallelism, aka the number of key-groups.
	 * @param parallelism the current parallelism of the operator
	 * @return the index of the parallel operator to which the given key should be routed.
	 */
	public static int assignKeyToParallelOperator(Object key, int maxParallelism, int parallelism) {
		return computeOperatorIndexForKeyGroup(maxParallelism, parallelism, assignToKeyGroup(key, maxParallelism));
	}
	
	//通过key 和 最大并行度，给出一个id，通过key的hashcode与最大并行度取模，获取一个分组id
	public static int assignToKeyGroup(Object key, int maxParallelism) {
		return computeKeyGroupForKeyHash(key.hashCode(), maxParallelism);
	}
	
	public static int computeKeyGroupForKeyHash(int keyHash, int maxParallelism) {
		return MathUtils.murmurHash(keyHash) % maxParallelism;
	}
	
	//通过分组id获取下游的channel id
	public static int computeOperatorIndexForKeyGroup(int maxParallelism, int parallelism, int keyGroupId) {
		return keyGroupId * parallelism / maxParallelism;
	}
```
* 先通过key的hashCode，算出maxParallelism的余数，也就是可以得到一个[0, maxParallelism)的整数，为分组id； 
* 在通过公式 keyGroupId * parallelism / maxParallelism ，计算出一个[0, parallelism)区间的整数，为下游的channel id，从而实现分区功能。


### keyby与flatmap的区别
* flatMap中，根据传入的flatMapper这个Function构建的是StreamOperator这个接口的子类的实例，而keyBy中，则是根据keySelector构建了ChannelSelector接口的子类实例； 
* keyBy中构建的StreamTransformation实例，并没有添加到StreamExecutionEnvironment的属性transformations这个列表中。


### timeWindow转换

* KeyedStream中存在这么一个调用window的方法
```java
@PublicEvolving
	public <W extends Window> WindowedStream<T, KEY, W> window(WindowAssigner<? super T, W> assigner) {
		return new WindowedStream<>(this, assigner);
	}
```


##### WindowAssigner

抽象类WindowAssigner中最主要的方法,针对数据进行窗口的分配
```java
/**
	 * Returns a {@code Collection} of windows that should be assigned to the element.
	 *
	 * @param element The element to which windows should be assigned.
	 * @param timestamp The timestamp of the element.
	 * @param context The {@link WindowAssignerContext} in which the assigner operates.
	 */
	public abstract Collection<W> assignWindows(T element, long timestamp, WindowAssignerContext context);
```

* SlidingProcessingTimeWindows对这个函数的实现
```java
    @Override
	public Collection<TimeWindow> assignWindows(Object element, long timestamp, WindowAssignerContext context) {
	    //获取当前的处理时间
		timestamp = context.getCurrentProcessingTime();
		//初始化窗口
		List<TimeWindow> windows = new ArrayList<>((int) (size / slide));
		//获取这个元素对应对应窗口的window_start（窗口开始时间）
		long lastStart = TimeWindow.getWindowStartWithOffset(timestamp, offset, slide);
		//从开始时间开始，根据slide创建窗口
		for (long start = lastStart;
			start > timestamp - size;
			start -= slide) {
			windows.add(new TimeWindow(start, start + size));
		}
		return windows;
	}
```
```java

	/**
	 * Method to get the window start for a timestamp.
	 *
	 * @param timestamp epoch millisecond to get the window start.
	 * @param offset The offset which window start would be shifted by.
	 * @param windowSize The size of the generated windows.
	 * @return window start
	 */
	public static long getWindowStartWithOffset(long timestamp, long offset, long windowSize) {
		return timestamp - (timestamp - offset + windowSize) % windowSize;
	}
```
```
a、timestamp = 1520406257000 // 2018-03-07 15:04:17 
b、offset = 0 
c、windowSize = 60000 
d、(timestamp - offset + windowSize) % windowSize = 17000 
e、说明在时间戳 1520406257000 之前最近的窗口是在 17000 毫秒的地方 
f、timestamp - (timestamp - offset + windowSize) % windowSize = 1520406240000 // 2018-03-07 15:04:00 
g、这样就可以保证每个时间窗口都是从整点开始, 而offset则是由于时区等原因需要时间调整而设置

```


#### Reference
https://blog.csdn.net/qq_21653785/article/details/79488249