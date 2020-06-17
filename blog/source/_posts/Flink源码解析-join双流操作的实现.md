---
title: Flink源码解析-join双流操作的实现
date: 2020-05-31 20:24:38
tags:
categories:
	- Flink
---



## Window Join and CoGroup

Window Join 操作，顾名思义，是基于时间窗口对两个流进行关联操作。相比于 Join 操作， CoGroup 提供了一个更为通用的方式来处理两个流在相同的窗口内匹配的元素。 Join 复用了 CoGroup 的实现逻辑。它们的使用方式如下：

```java
stream.join(otherStream)
    .where(<KeySelector>)
    .equalTo(<KeySelector>)
    .window(<WindowAssigner>)
    .apply(<JoinFunction>)

stream.coGroup(otherStream)
    .where(<KeySelector>)
    .equalTo(<KeySelector>)
    .window(<WindowAssigner>)
    .apply(<CoGroupFunction>)
```


从 JoinFunction 和 CogroupFunction 接口的定义中可以大致看出它们的区别：

```java

public interface JoinFunction<IN1, IN2, OUT> extends Function, Serializable {

	/**
	 * The join method, called once per joined pair of elements.
	 *
	 * @param first The element from first input.
	 * @param second The element from second input.
	 * @return The resulting element.
	 *
	 * @throws Exception This method may throw exceptions. Throwing an exception will cause the operation
	 *                   to fail and may trigger recovery.
	 */
	OUT join(IN1 first, IN2 second) throws Exception;
}

public interface CoGroupFunction<IN1, IN2, O> extends Function, Serializable {

	/**
	 * This method must be implemented to provide a user implementation of a
	 * coGroup. It is called for each pair of element groups where the elements share the
	 * same key.
	 *
	 * @param first The records from the first input.
	 * @param second The records from the second.
	 * @param out A collector to return elements.
	 *
	 * @throws Exception The function may throw Exceptions, which will cause the program to cancel,
	 *                   and may trigger the recovery logic.
	 */
	void coGroup(Iterable<IN1> first, Iterable<IN2> second, Collector<O> out) throws Exception;
}

```


可以看出来，JoinFunction 主要关注的是两个流中按照 key 匹配的每一对元素，而 CoGroupFunction 的参数则是两个中 key 相同的所有元素。JoinFunction 的逻辑更类似于 INNER JOIN，而 CoGroupFunction 除了可以实现 INNER JOIN，也可以实现 OUTER JOIN。


Window Join 的是被转换成 CoGroup 进行处理的：


```java
public class JoinedStreams<T1, T2> {
	public static class WithWindow<T1, T2, KEY, W extends Window> {
		public <T> DataStream<T> apply(JoinFunction<T1, T2, T> function, TypeInformation<T> resultType) {
			//clean the closure
			function = input1.getExecutionEnvironment().clean(function);

			//Join 操作被转换为 CoGroup
			coGroupedWindowedStream = input1.coGroup(input2)
				.where(keySelector1)
				.equalTo(keySelector2)
				.window(windowAssigner)
				.trigger(trigger)
				.evictor(evictor)
				.allowedLateness(allowedLateness);
			//JoinFunction 被包装为 CoGroupFunction
			return coGroupedWindowedStream
					.apply(new JoinCoGroupFunction<>(function), resultType);
		}
	}

	/**
	 * CoGroup function that does a nested-loop join to get the join result.
	 */
	private static class JoinCoGroupFunction<T1, T2, T>
			extends WrappingFunction<JoinFunction<T1, T2, T>>
			implements CoGroupFunction<T1, T2, T> {
		private static final long serialVersionUID = 1L;

		public JoinCoGroupFunction(JoinFunction<T1, T2, T> wrappedFunction) {
			super(wrappedFunction);
		}

		@Override
		public void coGroup(Iterable<T1> first, Iterable<T2> second, Collector<T> out) throws Exception {
			for (T1 val1: first) {
				for (T2 val2: second) {
					//每一个匹配的元素对
					out.collect(wrappedFunction.join(val1, val2));
				}
			}
		}
	}
}
```

那么 CoGroup 又是怎么实现两个流的操作的呢？

union + map 组合完成。

Flink 其实是通过一个变换(mapfunction)，将两个流转换成一个流进行处理，转换之后数据流中的每一条消息都有一个标记来记录这个消息是属于左边的流还是右边的流，这样窗口的操作就和单个流的实现一样了。等到窗口被触发的时候，再按照标记将窗口内的元素分为左边的一组和右边的一组，然后交给 CoGroupFunction 进行处理。
```java
public class CoGroupedStreams<T1, T2> {
	public static class WithWindow<T1, T2, KEY, W extends Window> {
		public <T> DataStream<T> apply(CoGroupFunction<T1, T2, T> function, TypeInformation<T> resultType) {
			//clean the closure
			function = input1.getExecutionEnvironment().clean(function);

			UnionTypeInfo<T1, T2> unionType = new UnionTypeInfo<>(input1.getType(), input2.getType());
			UnionKeySelector<T1, T2, KEY> unionKeySelector = new UnionKeySelector<>(keySelector1, keySelector2);

			DataStream<TaggedUnion<T1, T2>> taggedInput1 = input1
					.map(new Input1Tagger<T1, T2>())
					.setParallelism(input1.getParallelism())
					.returns(unionType); //左边流
			DataStream<TaggedUnion<T1, T2>> taggedInput2 = input2
					.map(new Input2Tagger<T1, T2>())
					.setParallelism(input2.getParallelism())
					.returns(unionType); //右边流
			
			//合并成一个数据流
			DataStream<TaggedUnion<T1, T2>> unionStream = taggedInput1.union(taggedInput2);

			// we explicitly create the keyed stream to manually pass the key type information in
			windowedStream =
					new KeyedStream<TaggedUnion<T1, T2>, KEY>(unionStream, unionKeySelector, keyType)
					.window(windowAssigner);

			if (trigger != null) {
				windowedStream.trigger(trigger);
			}
			if (evictor != null) {
				windowedStream.evictor(evictor);
			}
			if (allowedLateness != null) {
				windowedStream.allowedLateness(allowedLateness);
			}

			return windowedStream.apply(new CoGroupWindowFunction<T1, T2, T, KEY, W>(function), resultType);
		}
	}
	
	//将 CoGroupFunction 封装为 WindowFunction
	private static class CoGroupWindowFunction<T1, T2, T, KEY, W extends Window>
			extends WrappingFunction<CoGroupFunction<T1, T2, T>>
			implements WindowFunction<TaggedUnion<T1, T2>, T, KEY, W> {

		private static final long serialVersionUID = 1L;

		public CoGroupWindowFunction(CoGroupFunction<T1, T2, T> userFunction) {
			super(userFunction);
		}

		@Override
		public void apply(KEY key,
				W window,
				Iterable<TaggedUnion<T1, T2>> values,
				Collector<T> out) throws Exception {

			List<T1> oneValues = new ArrayList<>();
			List<T2> twoValues = new ArrayList<>();

			//窗口内的所有元素按标记重新分为左边的一组和右边的一组
			for (TaggedUnion<T1, T2> val: values) {
				if (val.isOne()) {
					oneValues.add(val.getOne());
				} else {
					twoValues.add(val.getTwo());
				}
			}
			//调用 CoGroupFunction
			wrappedFunction.coGroup(oneValues, twoValues, out);
		}
	}
}

```

从上面的源码来看，Iterable<IN1> first, Iterable<IN2> second可以不用判空，因为这部分被初始化了，只会是长度为0，不会为null。

join和cogroup没有什么太大差别，还是需要将数据全部转为list，只是交给function是单个还是多个而已。



## Interval Join


![image](https://blog.jrwang.me/img/flink/interval-join.svg)

Window Join 的一个局限是关联的两个数据流必须在同样的时间窗口中。但有些时候，我们希望在一个数据流中的消息到达时，在另一个数据流的一段时间内去查找匹配的元素。更确切地说，如果数据流 b 中消息到达时，我们希望在数据流 a 中匹配的元素的时间范围为 a.timestamp + lowerBound <= b.timestamp <= a.timestamp + upperBound；同样，对数据流 a 中的消息也是如此。在这种情况，就可以使用 Interval Join。具体的用法如下：


```java
stream
    .keyBy(<KeySelector>)
    .intervalJoin(otherStream.keyBy(<KeySelector>))
    .between(<Time>,<Time>)
    .process(<ProcessJoinFunction>)
```

Interval Join 是基于 ConnectedStreams 实现的：

```java
public class KeyedStream<T, KEY> extends DataStream<T> {
	public static class IntervalJoined<IN1, IN2, KEY> {
		public <OUT> SingleOutputStreamOperator<OUT> process(
				ProcessJoinFunction<IN1, IN2, OUT> processJoinFunction,
				TypeInformation<OUT> outputType) {
			Preconditions.checkNotNull(processJoinFunction);
			Preconditions.checkNotNull(outputType);

			final ProcessJoinFunction<IN1, IN2, OUT> cleanedUdf = left.getExecutionEnvironment().clean(processJoinFunction);

			final IntervalJoinOperator<KEY, IN1, IN2, OUT> operator =
				new IntervalJoinOperator<>(
					lowerBound,
					upperBound,
					lowerBoundInclusive,
					upperBoundInclusive,
					left.getType().createSerializer(left.getExecutionConfig()),
					right.getType().createSerializer(right.getExecutionConfig()),
					cleanedUdf
				);

			return left
				.connect(right)
				.keyBy(keySelector1, keySelector2)
				.transform("Interval Join", outputType, operator);
		}
	}
}
```

将两条keyStream通过connect转为ConnectedStreams 。

在 IntervalJoinOperator 中，使用两个 MapState 分别保存两个数据流到达的消息，MapState 的 key 是消息的时间。当一个数据流有新消息到达时，就会去另一个数据流的状态中查找时间落在匹配范围内的消息，然后进行关联处理。每一条消息会注册一个定时器，在时间越过该消息的有效范围后从状态中清除该消息。

```java
public class IntervalJoinOperator<K, T1, T2, OUT>
		extends AbstractUdfStreamOperator<OUT, ProcessJoinFunction<T1, T2, OUT>>
		implements TwoInputStreamOperator<T1, T2, OUT>, Triggerable<K, String> {
	
	private transient MapState<Long, List<BufferEntry<T1>>> leftBuffer;
	private transient MapState<Long, List<BufferEntry<T2>>> rightBuffer;

	@Override
	public void processElement1(StreamRecord<T1> record) throws Exception {
		processElement(record, leftBuffer, rightBuffer, lowerBound, upperBound, true);
	}

	@Override
	public void processElement2(StreamRecord<T2> record) throws Exception {
		processElement(record, rightBuffer, leftBuffer, -upperBound, -lowerBound, false);
	}

	private <THIS, OTHER> void processElement(
			final StreamRecord<THIS> record,
			final MapState<Long, List<IntervalJoinOperator.BufferEntry<THIS>>> ourBuffer,
			final MapState<Long, List<IntervalJoinOperator.BufferEntry<OTHER>>> otherBuffer,
			final long relativeLowerBound,
			final long relativeUpperBound,
			final boolean isLeft) throws Exception {

		final THIS ourValue = record.getValue();
		final long ourTimestamp = record.getTimestamp();

		if (ourTimestamp == Long.MIN_VALUE) {
			throw new FlinkException("Long.MIN_VALUE timestamp: Elements used in " +
					"interval stream joins need to have timestamps meaningful timestamps.");
		}

		if (isLate(ourTimestamp)) {
			return;
		}

		//将消息加入状态中
		addToBuffer(ourBuffer, ourValue, ourTimestamp);

		//从另一个数据流的状态中查找匹配的记录
		for (Map.Entry<Long, List<BufferEntry<OTHER>>> bucket: otherBuffer.entries()) {
			final long timestamp  = bucket.getKey();

			if (timestamp < ourTimestamp + relativeLowerBound ||
					timestamp > ourTimestamp + relativeUpperBound) {
				continue;
			}

			for (BufferEntry<OTHER> entry: bucket.getValue()) {
				if (isLeft) {
					collect((T1) ourValue, (T2) entry.element, ourTimestamp, timestamp);
				} else {
					collect((T1) entry.element, (T2) ourValue, timestamp, ourTimestamp);
				}
			}
		}

		//注册清理状态的timer
		long cleanupTime = (relativeUpperBound > 0L) ? ourTimestamp + relativeUpperBound : ourTimestamp;
		if (isLeft) {
			internalTimerService.registerEventTimeTimer(CLEANUP_NAMESPACE_LEFT, cleanupTime);
		} else {
			internalTimerService.registerEventTimeTimer(CLEANUP_NAMESPACE_RIGHT, cleanupTime);
		}
	}
}
```


### Reference

https://blog.jrwang.me/2019/flink-source-code-two-stream-join/
