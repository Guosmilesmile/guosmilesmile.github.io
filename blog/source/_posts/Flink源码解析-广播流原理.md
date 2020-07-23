---
title: Flink源码解析-广播流原理
date: 2020-06-18 20:16:49
tags:
categories:
	- Flink
---



### 使用

详情见https://ci.apache.org/projects/flink/flink-docs-release-1.10/dev/stream/state/broadcast_state.html



```java

// a map descriptor to store the name of the rule (string) and the rule itself.
MapStateDescriptor<String, Rule> ruleStateDescriptor = new MapStateDescriptor<>(
			"RulesBroadcastState",
			BasicTypeInfo.STRING_TYPE_INFO,
			TypeInformation.of(new TypeHint<Rule>() {}));
		
// broadcast the rules and create the broadcast state
BroadcastStream<Rule> ruleBroadcastStream = ruleStream
                        .broadcast(ruleStateDescriptor);


DataStream<String> output = colorPartitionedStream
                 .connect(ruleBroadcastStream)
                 .process(
                     
                     // type arguments in our KeyedBroadcastProcessFunction represent: 
                     //   1. the key of the keyed stream
                     //   2. the type of elements in the non-broadcast side
                     //   3. the type of elements in the broadcast side
                     //   4. the type of the result, here a string
                     
                     new KeyedBroadcastProcessFunction<Color, Item, Rule, String>() {
                         // my matching logic
                     }
                 );

```





如果colorPartitionedStream是一个KeyedStream，那么process使用的是KeyedBroadcastProcessFunction，如果是一个普通的DataStream，那么用的是BroadcastProcessFunction。



```java

public abstract class BroadcastProcessFunction<IN1, IN2, OUT> extends BaseBroadcastProcessFunction {

    public abstract void processElement(IN1 value, ReadOnlyContext ctx, Collector<OUT> out) throws Exception;

    public abstract void processBroadcastElement(IN2 value, Context ctx, Collector<OUT> out) throws Exception;
}

```



```java

public abstract class KeyedBroadcastProcessFunction<KS, IN1, IN2, OUT> {

    public abstract void processElement(IN1 value, ReadOnlyContext ctx, Collector<OUT> out) throws Exception;

    public abstract void processBroadcastElement(IN2 value, Context ctx, Collector<OUT> out) throws Exception;

    public void onTimer(long timestamp, OnTimerContext ctx, Collector<OUT> out) throws Exception;
}
```



官网提供的demo

```java
new KeyedBroadcastProcessFunction<Color, Item, Rule, String>() {

    // store partial matches, i.e. first elements of the pair waiting for their second element
    // we keep a list as we may have many first elements waiting
    private final MapStateDescriptor<String, List<Item>> mapStateDesc =
        new MapStateDescriptor<>(
            "items",
            BasicTypeInfo.STRING_TYPE_INFO,
            new ListTypeInfo<>(Item.class));

    // identical to our ruleStateDescriptor above
    private final MapStateDescriptor<String, Rule> ruleStateDescriptor = 
        new MapStateDescriptor<>(
            "RulesBroadcastState",
            BasicTypeInfo.STRING_TYPE_INFO,
            TypeInformation.of(new TypeHint<Rule>() {}));

    @Override
    public void processBroadcastElement(Rule value,
                                        Context ctx,
                                        Collector<String> out) throws Exception {
        ctx.getBroadcastState(ruleStateDescriptor).put(value.name, value);
    }

    @Override
    public void processElement(Item value,
                               ReadOnlyContext ctx,
                               Collector<String> out) throws Exception {

        final MapState<String, List<Item>> state = getRuntimeContext().getMapState(mapStateDesc);
        final Shape shape = value.getShape();
    
        for (Map.Entry<String, Rule> entry :
                ctx.getBroadcastState(ruleStateDescriptor).immutableEntries()) {
            final String ruleName = entry.getKey();
            final Rule rule = entry.getValue();
    
            List<Item> stored = state.get(ruleName);
            if (stored == null) {
                stored = new ArrayList<>();
            }
    
            if (shape == rule.second && !stored.isEmpty()) {
                for (Item i : stored) {
                    out.collect("MATCH: " + i + " - " + value);
                }
                stored.clear();
            }
    
            // there is no else{} to cover if rule.first == rule.second
            if (shape.equals(rule.first)) {
                stored.add(value);
            }
    
            if (stored.isEmpty()) {
                state.remove(ruleName);
            } else {
                state.put(ruleName, stored);
            }
        }
    }
}
```



需要注意的是，目前广播流的状态都保存在内存中，RocksDB 状态后端目前还不支持广播状态。



### 分析源码

#### MapStateDescriptor

首先要说明一些概念：

- Flink中包含两种基础的状态：Keyed State和Operator State。
- Keyed State和Operator State又可以 以两种形式存在：原始状态和托管状态。
- 托管状态是由Flink框架管理的状态，如ValueState, ListState, MapState等。
- raw state即原始状态，由用户自行管理状态具体的数据结构，框架在做checkpoint的时候，使用byte[]来读写状态内容，对其内部数据结构一无所知。
- MapState是托管状态的一种：即状态值为一个map。用户通过`put`或`putAll`方法添加元素。



根据dataStream的不同决定了选用不同的state。



---------

DataStream.broadcast

```java
@PublicEvolving
public BroadcastStream<T> broadcast(final MapStateDescriptor<?, ?>... broadcastStateDescriptors) {
   Preconditions.checkNotNull(broadcastStateDescriptors);
   final DataStream<T> broadcastStream = setConnectionType(new BroadcastPartitioner<>());
   return new BroadcastStream<>(environment, broadcastStream, broadcastStateDescriptors);
}
```





在初始构造的时候，做了如下的事情



* 将partitioning设置为BroadcastPartitioner，这个分区策略是将数据发送到所有的下游，如果有n个channel，那就是每个channel都发一份数据。
* 用dataStream和状态描述创建一个BroadcastStream



BroadcastStream没什么特别的，属性如下



```java
public class BroadcastStream<T> {

	private final StreamExecutionEnvironment environment;

	private final DataStream<T> inputStream;

	/**
	 * The {@link org.apache.flink.api.common.state.StateDescriptor state descriptors} of the
	 * registered {@link org.apache.flink.api.common.state.BroadcastState broadcast states}. These
	 * states have {@code key-value} format.
	 */
	private final List<MapStateDescriptor<?, ?>> broadcastStateDescriptors;
}
```



在DataStream.connect(BroadcastStream)的时候构建出关键类BroadcastConnectedStream



```java
	@PublicEvolving
	public <R> BroadcastConnectedStream<T, R> connect(BroadcastStream<R> broadcastStream) {
		return new BroadcastConnectedStream<>(
				environment,
				this,
				Preconditions.checkNotNull(broadcastStream),
				broadcastStream.getBroadcastStateDescriptor());
	}
```



在构造函数上，将dataStream和broadcastStream复制给input1和input2，作为TwoInput。



```java
protected BroadcastConnectedStream(
			final StreamExecutionEnvironment env,
			final DataStream<IN1> input1,
			final BroadcastStream<IN2> input2,
			final List<MapStateDescriptor<?, ?>> broadcastStateDescriptors) {
		this.environment = requireNonNull(env);
		this.inputStream1 = requireNonNull(input1);
		this.inputStream2 = requireNonNull(input2);
		this.broadcastStateDescriptors = requireNonNull(broadcastStateDescriptors);
	}
```



用户代码DataStream.connect(BroadcastStream).process()的时候调用process方法将真正的计算操作交给了CoBroadcastWithKeyedOperator



```java
@PublicEvolving
public <KS, OUT> SingleOutputStreamOperator<OUT> process(
      final KeyedBroadcastProcessFunction<KS, IN1, IN2, OUT> function,
      final TypeInformation<OUT> outTypeInfo) {

   Preconditions.checkNotNull(function);
   Preconditions.checkArgument(inputStream1 instanceof KeyedStream,
         "A KeyedBroadcastProcessFunction can only be used on a keyed stream.");

   TwoInputStreamOperator<IN1, IN2, OUT> operator =
         new CoBroadcastWithKeyedOperator<>(clean(function), broadcastStateDescriptors);
   return transform("Co-Process-Broadcast-Keyed", outTypeInfo, operator);
}
```



CoBroadcastWithKeyedOperator



```java
@Override
	public void processElement1(StreamRecord<IN1> element) throws Exception {
		collector.setTimestamp(element);
		rContext.setElement(element);
		userFunction.processElement(element.getValue(), rContext, collector);
		rContext.setElement(null);
	}

	@Override
	public void processElement2(StreamRecord<IN2> element) throws Exception {
		collector.setTimestamp(element);
		rwContext.setElement(element);
		userFunction.processBroadcastElement(element.getValue(), rwContext, collector);
		rwContext.setElement(null);
	}
```

本质上，其实还是双流join，只是没有数据late或者其他watermark之类的判断，广播流的数据存储在用户定义的state中，另一条从state中获取，形成广播流的join效果。



-------------------------



状态的创建主要是在CoBroadcastWithKeyedOperator.open方法



```java
for (MapStateDescriptor<?, ?> descriptor: broadcastStateDescriptors) {
			broadcastStates.put(descriptor, getOperatorStateBackend().getBroadcastState(descriptor));
		}
```



#### OperatorStateBackEnd

OperatorStateBackEnd 主要管理OperatorState. 目前只有一种实现: DefaultOperatorStateBackend。


BroadcastState的实现也只有一个HeapBroadcastState，所以广播流的状态都保存在内存中。




```java

public <K, V> BroadcastState<K, V> getBroadcastState(final MapStateDescriptor<K, V> stateDescriptor) throws StateMigrationException {

		Preconditions.checkNotNull(stateDescriptor);
		String name = Preconditions.checkNotNull(stateDescriptor.getName());

		BackendWritableBroadcastState<K, V> previous =
			(BackendWritableBroadcastState<K, V>) accessedBroadcastStatesByName.get(name);

		if (previous != null) {
			checkStateNameAndMode(
					previous.getStateMetaInfo().getName(),
					name,
					previous.getStateMetaInfo().getAssignmentMode(),
					OperatorStateHandle.Mode.BROADCAST);
			return previous;
		}

		stateDescriptor.initializeSerializerUnlessSet(getExecutionConfig());
		TypeSerializer<K> broadcastStateKeySerializer = Preconditions.checkNotNull(stateDescriptor.getKeySerializer());
		TypeSerializer<V> broadcastStateValueSerializer = Preconditions.checkNotNull(stateDescriptor.getValueSerializer());

		BackendWritableBroadcastState<K, V> broadcastState =
			(BackendWritableBroadcastState<K, V>) registeredBroadcastStates.get(name);

		if (broadcastState == null) {
			broadcastState = new HeapBroadcastState<>(
					new RegisteredBroadcastStateBackendMetaInfo<>(
							name,
							OperatorStateHandle.Mode.BROADCAST,
							broadcastStateKeySerializer,
							broadcastStateValueSerializer));
			registeredBroadcastStates.put(name, broadcastState);
		} else {
			// has restored state; check compatibility of new state access

			checkStateNameAndMode(
					broadcastState.getStateMetaInfo().getName(),
					name,
					broadcastState.getStateMetaInfo().getAssignmentMode(),
					OperatorStateHandle.Mode.BROADCAST);

			RegisteredBroadcastStateBackendMetaInfo<K, V> restoredBroadcastStateMetaInfo = broadcastState.getStateMetaInfo();

			// check whether new serializers are incompatible
			TypeSerializerSchemaCompatibility<K> keyCompatibility =
				restoredBroadcastStateMetaInfo.updateKeySerializer(broadcastStateKeySerializer);
			if (keyCompatibility.isIncompatible()) {
				throw new StateMigrationException("The new key typeSerializer for broadcast state must not be incompatible.");
			}

			TypeSerializerSchemaCompatibility<V> valueCompatibility =
				restoredBroadcastStateMetaInfo.updateValueSerializer(broadcastStateValueSerializer);
			if (valueCompatibility.isIncompatible()) {
				throw new StateMigrationException("The new value typeSerializer for broadcast state must not be incompatible.");
			}

			broadcastState.setStateMetaInfo(restoredBroadcastStateMetaInfo);
		}

		accessedBroadcastStatesByName.put(name, broadcastState);
		return broadcastState;
	}
    
    
```



```java
broadcastState = new HeapBroadcastState<>(
      new RegisteredBroadcastStateBackendMetaInfo<>(
            name,
            OperatorStateHandle.Mode.BROADCAST,
            broadcastStateKeySerializer,
            broadcastStateValueSerializer));
registeredBroadcastStates.put(name, broadcastState);
```

可以看出这部分是放在内存中的





### Reference

https://www.cnblogs.com/rossiXYZ/p/12594315.html



https://ci.apache.org/projects/flink/flink-docs-release-1.10/dev/stream/state/broadcast_state.html