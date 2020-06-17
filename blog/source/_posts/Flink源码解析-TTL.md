---
title: Flink源码解析-TTL
date: 2020-06-17 19:54:37
tags:
categories:
	- Flink
---
## 前言



从Flink 1.6版本开始，社区为状态引入了TTL（time-to-live，生存时间）机制，支持Keyed State的自动过期，有效解决了状态数据在无干预情况下无限增长导致OOM的问题。



```java
StateTtlConfig ttlConfig = StateTtlConfig
    .newBuilder(Time.seconds(1))
    .setUpdateType(StateTtlConfig.UpdateType.OnCreateAndWrite)
    .setStateVisibility(StateTtlConfig.StateVisibility.NeverReturnExpired)
    .build();
    
ValueStateDescriptor<String> stateDescriptor = new ValueStateDescriptor<>("text state", String.class);
stateDescriptor.enableTimeToLive(ttlConfig);
```



那么State TTL的背后又隐藏着什么样的思路呢？下面就从设置类StateTtlConfig入手开始研究



### State

Keyed State和Operator State
Flink中包含两种基础的状态：Keyed State和Operator State。

#### Keyed State
顾名思义，就是基于KeyedStream上的状态。这个状态是跟特定的key绑定的，对KeyedStream流上的每一个key，可能都对应一个state。

#### Operator State
与Keyed State不同，Operator State跟一个特定operator的一个并发实例绑定，整个operator只对应一个state。相比较而言，在一个operator上，可能会有很多个key，从而对应多个keyed state。

举例来说，Flink中的Kafka Connector，就使用了operator state。它会在每个connector实例中，保存该实例中消费topic的所有(partition, offset)映射。



### StateTtlConfig

该类中有5个成员属性，它们就是用户需要指定的全部参数了。

```java
	private final UpdateType updateType;
	private final StateVisibility stateVisibility;
	private final TtlTimeCharacteristic ttlTimeCharacteristic;
	private final Time ttl;
	private final CleanupStrategies cleanupStrategies;
```



其中，ttl参数表示用户设定的状态生存时间。而UpdateType、StateVisibility和TtlTimeCharacteristic都是枚举，分别代表状态时间戳的更新方式、过期状态数据的可见性，以及对应的时间特征。它们的含义在注释中已经解释得很清楚了。





```java
/**
 * This option value configures when to update last access timestamp which prolongs state TTL.
 */
public enum UpdateType {
   /** TTL is disabled. State does not expire. */
   Disabled,
   /** Last access timestamp is initialised when state is created and updated on every write operation. */
    // 以创建和写数据的时间为最新更新时间
   OnCreateAndWrite,
   /** The same as <code>OnCreateAndWrite</code> but also updated on read. */
    // 读和写都会作为最新的时间
   OnReadAndWrite
}
```





```java
	/**
	 * This option configures whether expired user value can be returned or not.
	 */
	public enum StateVisibility {
		/** Return expired user value if it is not cleaned up yet. */
        // 如果还没清理，返回过期数据
		ReturnExpiredIfNotCleanedUp,
		/** Never return expired user value. */
        // 不返回过期的数据
		NeverReturnExpired
	}
```



CleanupStrategies内部类则用来规定过期状态的特殊清理策略，用户在构造StateTtlConfig时，可以通过调用以下方法之一指定。

 

* cleanupFullSnapshot
```java
strategies.put(CleanupStrategies.Strategies.FULL_STATE_SCAN_SNAPSHOT, EMPTY_STRATEGY);
```

当对状态做全量快照时清理过期数据，对开启了增量检查点（incremental checkpoint）的RocksDB状态后端无效，对应源码中的EmptyCleanupStrategy。



为什么叫做“空的”清理策略呢？因为该选项只能保证状态持久化时不包含过期数据，但TaskManager本地的过期状态则不作任何处理，所以无法从根本上解决OOM的问题，需要定期重启作业。



* cleanupIncrementally(int cleanupSize, boolean runCleanupForEveryRecord)

增量清理过期数据，默认在每次访问状态时进行清理，将runCleanupForEveryRecord设为true可以附加在每次写入/删除时清理。cleanupSize指定每次触发清理时检查的状态条数。
仅对基于堆的状态后端有效，对应源码中的IncrementalCleanupStrategy。



* cleanupInRocksdbCompactFilter(long queryTimeAfterNumEntries)

当RocksDB做compaction操作时，通过Flink定制的过滤器（FlinkCompactionFilter）过滤掉过期状态数据。参数queryTimeAfterNumEntries用于指定在写入多少条状态数据后，通过状态时间戳来判断是否过期。当RocksDB做compaction操作时，通过Flink定制的过滤器（FlinkCompactionFilter）过滤掉过期状态数据。参数queryTimeAfterNumEntries用于指定在写入多少条状态数据后，通过状态时间戳来判断是否过期。

该策略仅对RocksDB状态后端有效，对应源码中的RocksdbCompactFilterCleanupStrategy。CompactionFilter是RocksDB原生提供的机制。



如果不调用上述方法，则采用默认的后台清理策略，下文有讲。





### TtlStateFactory、TtlStateContext

在所有Keyed State状态后端的抽象基类AbstractKeyedStateBackend中，创建并记录一个状态实例的方法如下



```java
/**
	 * @see KeyedStateBackend
	 */
	@Override
	@SuppressWarnings("unchecked")
	public <N, S extends State, V> S getOrCreateKeyedState(
			final TypeSerializer<N> namespaceSerializer,
			StateDescriptor<S, V> stateDescriptor) throws Exception {
		checkNotNull(namespaceSerializer, "Namespace serializer");
		checkNotNull(keySerializer, "State key serializer has not been configured in the config. " +
				"This operation cannot use partitioned state.");

		InternalKvState<K, ?, ?> kvState = keyValueStatesByName.get(stateDescriptor.getName());
		if (kvState == null) {
			if (!stateDescriptor.isSerializerInitialized()) {
				stateDescriptor.initializeSerializerUnlessSet(executionConfig);
			}
			kvState = TtlStateFactory.createStateAndWrapWithTtlIfEnabled(
				namespaceSerializer, stateDescriptor, this, ttlTimeProvider);
			keyValueStatesByName.put(stateDescriptor.getName(), kvState);
			publishQueryableStateIfEnabled(stateDescriptor, kvState);
		}
		return (S) kvState;
	}

```

可见是调用了TtlStateFactory.createStateAndWrapWithTtlIfEnabled()方法来真正创建。顾名思义，TtlStateFactory是产生TTL状态的工厂类。



```java
public static <K, N, SV, TTLSV, S extends State, IS extends S> IS createStateAndWrapWithTtlIfEnabled(
		TypeSerializer<N> namespaceSerializer,
		StateDescriptor<S, SV> stateDesc,
		KeyedStateBackend<K> stateBackend,
		TtlTimeProvider timeProvider) throws Exception {
		Preconditions.checkNotNull(namespaceSerializer);
		Preconditions.checkNotNull(stateDesc);
		Preconditions.checkNotNull(stateBackend);
		Preconditions.checkNotNull(timeProvider);
		return  stateDesc.getTtlConfig().isEnabled() ?
			new TtlStateFactory<K, N, SV, TTLSV, S, IS>(
				namespaceSerializer, stateDesc, stateBackend, timeProvider)
				.createState() :
			stateBackend.createInternalState(namespaceSerializer, stateDesc);
	}
```

由上可知，如果我们为状态描述符StateDescriptor加入了TTL，那么就会调用TtlStateFactory.createState()方法创建一个带有TTL的状态实例；否则，就调用StateBackend.createInternalState()创建一个普通的状态实例。



TtlStateFactory.createState()的代码如下。



```java
	private IS createState() throws Exception {
		SupplierWithException<IS, Exception> stateFactory = stateFactories.get(stateDesc.getClass());
		if (stateFactory == null) {
			String message = String.format("State %s is not supported by %s",
				stateDesc.getClass(), TtlStateFactory.class);
			throw new FlinkRuntimeException(message);
		}
		IS state = stateFactory.get();
		if (incrementalCleanup != null) {
			incrementalCleanup.setTtlState((AbstractTtlState<K, N, ?, TTLSV, ?>) state);
		}
		return state;
	}
```



stateFactories是一个Map结构，维护了各种状态描述符与对应产生该种状态对象的工厂方法映射。所有的工厂方法都被包装成了Supplier（Java 8提供的函数式接口），所以在上述createState()方法中，可以通过Supplier.get()方法来实际执行createTtl*State()工厂方法，并获得新的状态实例。


```java
Map<Class<? extends StateDescriptor>, SupplierWithException<IS, Exception>> stateFactories;

	private Map<Class<? extends StateDescriptor>, SupplierWithException<IS, Exception>> createStateFactories() {
		return Stream.of(
			Tuple2.of(ValueStateDescriptor.class, (SupplierWithException<IS, Exception>) this::createValueState),
			Tuple2.of(ListStateDescriptor.class, (SupplierWithException<IS, Exception>) this::createListState),
			Tuple2.of(MapStateDescriptor.class, (SupplierWithException<IS, Exception>) this::createMapState),
			Tuple2.of(ReducingStateDescriptor.class, (SupplierWithException<IS, Exception>) this::createReducingState),
			Tuple2.of(AggregatingStateDescriptor.class, (SupplierWithException<IS, Exception>) this::createAggregatingState),
			Tuple2.of(FoldingStateDescriptor.class, (SupplierWithException<IS, Exception>) this::createFoldingState)
		).collect(Collectors.toMap(t -> t.f0, t -> t.f1));
	}


private IS createValueState() throws Exception {
		ValueStateDescriptor<TtlValue<SV>> ttlDescriptor = new ValueStateDescriptor<>(
			stateDesc.getName(), new TtlSerializer<>(LongSerializer.INSTANCE, stateDesc.getSerializer()));
		return (IS) new TtlValueState<>(createTtlStateContext(ttlDescriptor));
	}
```

带有TTL的状态类名其实就是普通状态类名加上Ttl前缀，只是没有公开给用户而已。并且在生成Ttl*State时，还会通过createTtlStateContext()方法生成TTL状态的上下文。来看下TtlStateFactory.createTtlStateContext方法





```java
private <OIS extends State, TTLS extends State, V, TTLV> TtlStateContext<OIS, V>
		createTtlStateContext(StateDescriptor<TTLS, TTLV> ttlDescriptor) throws Exception {

		ttlDescriptor.enableTimeToLive(stateDesc.getTtlConfig()); // also used by RocksDB backend for TTL compaction filter config
		OIS originalState = (OIS) stateBackend.createInternalState(
			namespaceSerializer, ttlDescriptor, getSnapshotTransformFactory());
		return new TtlStateContext<>(
			originalState, ttlConfig, timeProvider, (TypeSerializer<V>) stateDesc.getSerializer(),
			registerTtlIncrementalCleanupCallback((InternalKvState<?, ?, ?>) originalState));
	}

```



TtlStateContext的本质是对以下几个实例做了封装。

* T original    原始State（通过StateBackend.createInternalState()方法创建）及其序列化器（通过StateDescriptor.getSerializer()方法取得）, 对应不同状态后台的map，如果是rocksdb状态后台，得到的是RocksDBMapState；

* StateTtlConfig，前文已经讲过；

* TtlTimeProvider，用来提供判断状态过期标准的时间戳。当前只是简单地代理了System.currentTimeMillis()，没有任何其他代码；

* 一个Runnable类型的回调方法，通过registerTtlIncrementalCleanupCallback()方法产生，用于状态数据的增量清理，后面会看到它的用途。

  

  

接下来就具体看看TTL状态是如何实现的。

![](https://note.youdao.com/yws/api/personal/file/4EAD8AC8B9A443B8B39E88FD43616BB7?method=download&shareKey=811a37a33b8a1e7c751cb597214b5916)



所有Ttl*State都是AbstractTtlState的子类，而AbstractTtlState又是装饰器AbstractTtlDecorator的子类。



```java

abstract class AbstractTtlDecorator<T> {
	/** Wrapped original state handler. */
	final T original;

	final StateTtlConfig config;

	final TtlTimeProvider timeProvider;

	/** Whether to renew expiration timestamp on state read access. */
	final boolean updateTsOnRead;

	/** Whether to renew expiration timestamp on state read access. */
	final boolean returnExpired;

	/** State value time to live in milliseconds. */
	final long ttl;

	AbstractTtlDecorator(
		T original,
		StateTtlConfig config,
		TtlTimeProvider timeProvider) {
		Preconditions.checkNotNull(original);
		Preconditions.checkNotNull(config);
		Preconditions.checkNotNull(timeProvider);
		this.original = original;
		this.config = config;
		this.timeProvider = timeProvider;
		this.updateTsOnRead = config.getUpdateType() == StateTtlConfig.UpdateType.OnReadAndWrite;
		this.returnExpired = config.getStateVisibility() == StateTtlConfig.StateVisibility.ReturnExpiredIfNotCleanedUp;
		this.ttl = config.getTtl().toMilliseconds();
	}

	<V> V getUnexpired(TtlValue<V> ttlValue) {
		return ttlValue == null || (expired(ttlValue) && !returnExpired) ? null : ttlValue.getUserValue();
	}

	<V> boolean expired(TtlValue<V> ttlValue) {
		return TtlUtils.expired(ttlValue, ttl, timeProvider);
	}

	<V> TtlValue<V> wrapWithTs(V value) {
		return TtlUtils.wrapWithTs(value, timeProvider.currentTimestamp());
	}

	<V> TtlValue<V> rewrapWithNewTs(TtlValue<V> ttlValue) {
		return wrapWithTs(ttlValue.getUserValue());
	}

	<SE extends Throwable, CE extends Throwable, CLE extends Throwable, V> V getWithTtlCheckAndUpdate(
		SupplierWithException<TtlValue<V>, SE> getter,
		ThrowingConsumer<TtlValue<V>, CE> updater,
		ThrowingRunnable<CLE> stateClear) throws SE, CE, CLE {
		TtlValue<V> ttlValue = getWrappedWithTtlCheckAndUpdate(getter, updater, stateClear);
		return ttlValue == null ? null : ttlValue.getUserValue();
	}

	<SE extends Throwable, CE extends Throwable, CLE extends Throwable, V> TtlValue<V> getWrappedWithTtlCheckAndUpdate(
		SupplierWithException<TtlValue<V>, SE> getter,
		ThrowingConsumer<TtlValue<V>, CE> updater,
		ThrowingRunnable<CLE> stateClear) throws SE, CE, CLE {
		TtlValue<V> ttlValue = getter.get();
		if (ttlValue == null) {
			return null;
		} else if (expired(ttlValue)) {
			stateClear.run();
			if (!returnExpired) {
				return null;
			}
		} else if (updateTsOnRead) {
			updater.accept(rewrapWithNewTs(ttlValue));
		}
		return ttlValue;
	}
}

```



它的成员属性比较容易理解，例如，updateTsOnRead表示在读取状态值时也更新时间戳（即UpdateType.OnReadAndWrite），returnExpired表示即使状态过期，在被真正删除之前也返回它的值（即StateVisibility.ReturnExpiredIfNotCleanedUp）。



状态值与TTL的包装（成为TtlValue）以及过期检测都由工具类TtlUtils来负责



```java

public class TtlUtils {
	static <V> boolean expired(@Nullable TtlValue<V> ttlValue, long ttl, TtlTimeProvider timeProvider) {
		return expired(ttlValue, ttl, timeProvider.currentTimestamp());
	}

	static <V> boolean expired(@Nullable TtlValue<V> ttlValue, long ttl, long currentTimestamp) {
		return ttlValue != null && expired(ttlValue.getLastAccessTimestamp(), ttl, currentTimestamp);
	}

	static boolean expired(long ts, long ttl, TtlTimeProvider timeProvider) {
		return expired(ts, ttl, timeProvider.currentTimestamp());
	}

	public static boolean expired(long ts, long ttl, long currentTimestamp) {
		return getExpirationTimestamp(ts, ttl) <= currentTimestamp;
	}

	private static long getExpirationTimestamp(long ts, long ttl) {
		long ttlWithoutOverflow = ts > 0 ? Math.min(Long.MAX_VALUE - ts, ttl) : ttl;
		return ts + ttlWithoutOverflow;
	}

	static <V> TtlValue<V> wrapWithTs(V value, long ts) {
		return new TtlValue<>(value, ts);
	}
}

```





TtlValue的属性只有两个：状态值和时间戳



```java

public class TtlValue<T> implements Serializable {
	private static final long serialVersionUID = 5221129704201125020L;

	@Nullable
	private final T userValue;
	private final long lastAccessTimestamp;

	public TtlValue(@Nullable T userValue, long lastAccessTimestamp) {
		this.userValue = userValue;
		this.lastAccessTimestamp = lastAccessTimestamp;
	}

	@Nullable
	public T getUserValue() {
		return userValue;
	}

	public long getLastAccessTimestamp() {
		return lastAccessTimestamp;
	}
}
```





AbstractTtlDecorator核心方法是获取状态值的getWrappedWithTtlCheckAndUpdate()，它接受三个参数：



- getter：一个可抛出异常的Supplier，用于获取状态值；
- updater：一个可抛出异常的Consumer，用于更新状态的时间戳；
- stateClear：一个可抛出异常的Runnable，用于异步删除过期状态。



可见，在默认情况下的后台清理策略是：**只有状态值被读取时，才会做过期检测，并异步清除过期的状态**。这种**惰性清理**的机制会导致**那些实际已经过期但从未被再次访问过的状态无法被删除**，需要特别注意。官方文档中也已有提示：

```
By default, expired values are explicitly removed on read, such as ValueState#value, and periodically garbage collected in the background if supported by the configured state backend.
```



不过到了1.10版本，这个已经有了解决方法，在官网也有提到



```
heap state backend relies on incremental cleanup and RocksDB backend uses compaction filter for background cleanup.

RocksDB compaction filter will query current timestamp, used to check expiration, from Flink every time after processing certain number of state  entries.
```

每次处理一定数量的 StateEntry 之后，会获取当前的 timestamp 然后在 RocksDB 的 compaction
时对所有的 StateEntry 进行 filter,过期的状态 Key就过滤删除掉。



当确认到状态过期时，会调用stateClear的逻辑进行删除；如果需要在读取时顺便更新状态的时间戳，会调用updater的逻辑重新包装一个TtlValue。





AbstractTtlState的代码



```java


final Runnable accessCallback;
 
<SE extends Throwable, CE extends Throwable, T> T getWithTtlCheckAndUpdate(
    SupplierWithException<TtlValue<T>, SE> getter,
    ThrowingConsumer<TtlValue<T>, CE> updater) throws SE, CE {
    return getWithTtlCheckAndUpdate(getter, updater, original::clear);
}
 
@Override
public void clear() {
    original.clear();
    accessCallback.run();
}

```



其中，accessCallback就是TtlStateContext中注册的增量清理回调。

下面以TtlMapState为例，看看具体的TTL状态如何利用上文所述的这些实现。



#### TtlMapState

```java
class TtlMapState<K, N, UK, UV>
	extends AbstractTtlState<K, N, Map<UK, UV>, Map<UK, TtlValue<UV>>, InternalMapState<K, N, UK, TtlValue<UV>>>
	implements InternalMapState<K, N, UK, UV> {
	TtlMapState(TtlStateContext<InternalMapState<K, N, UK, TtlValue<UV>>, Map<UK, UV>> ttlStateContext) {
		super(ttlStateContext);
	}

	@Override
	public UV get(UK key) throws Exception {
		TtlValue<UV> ttlValue = getWrapped(key);
		return ttlValue == null ? null : ttlValue.getUserValue();
	}

	private TtlValue<UV> getWrapped(UK key) throws Exception {
		accessCallback.run();
		return getWrappedWithTtlCheckAndUpdate(
			() -> original.get(key), v -> original.put(key, v), () -> original.remove(key));
	}

	@Override
	public void put(UK key, UV value) throws Exception {
		accessCallback.run();
		original.put(key, wrapWithTs(value));
	}

	@Override
	public void putAll(Map<UK, UV> map) throws Exception {
		accessCallback.run();
		if (map == null) {
			return;
		}
		Map<UK, TtlValue<UV>> ttlMap = new HashMap<>(map.size());
		long currentTimestamp = timeProvider.currentTimestamp();
		for (Map.Entry<UK, UV> entry : map.entrySet()) {
			UK key = entry.getKey();
			ttlMap.put(key, TtlUtils.wrapWithTs(entry.getValue(), currentTimestamp));
		}
		original.putAll(ttlMap);
	}

	@Override
	public void remove(UK key) throws Exception {
		accessCallback.run();
		original.remove(key);
	}

	@Override
	public boolean contains(UK key) throws Exception {
		TtlValue<UV> ttlValue = getWrapped(key);
		return ttlValue != null;
	}

	@Override
	public Iterable<Map.Entry<UK, UV>> entries() throws Exception {
		return entries(e -> e);
	}

	private <R> Iterable<R> entries(
		Function<Map.Entry<UK, UV>, R> resultMapper) throws Exception {
		accessCallback.run();
		Iterable<Map.Entry<UK, TtlValue<UV>>> withTs = original.entries();
		return () -> new EntriesIterator<>(withTs == null ? Collections.emptyList() : withTs, resultMapper);
	}

	@Override
	public Iterable<UK> keys() throws Exception {
		return entries(Map.Entry::getKey);
	}

	@Override
	public Iterable<UV> values() throws Exception {
		return entries(Map.Entry::getValue);
	}

	@Override
	public Iterator<Map.Entry<UK, UV>> iterator() throws Exception {
		return entries().iterator();
	}

	@Override
	public boolean isEmpty() throws Exception {
		accessCallback.run();
		return original.isEmpty();
	}

	@Nullable
	@Override
	public Map<UK, TtlValue<UV>> getUnexpiredOrNull(@Nonnull Map<UK, TtlValue<UV>> ttlValue) {
		Map<UK, TtlValue<UV>> unexpired = new HashMap<>();
		TypeSerializer<TtlValue<UV>> valueSerializer =
			((MapSerializer<UK, TtlValue<UV>>) original.getValueSerializer()).getValueSerializer();
		for (Map.Entry<UK, TtlValue<UV>> e : ttlValue.entrySet()) {
			if (!expired(e.getValue())) {
				// we have to do the defensive copy to update the value
				unexpired.put(e.getKey(), valueSerializer.copy(e.getValue()));
			}
		}
		return ttlValue.size() == unexpired.size() ? ttlValue : unexpired;
	}

	@Override
	public void clear() {
		original.clear();
	}

	private class EntriesIterator<R> implements Iterator<R> {
		private final Iterator<Map.Entry<UK, TtlValue<UV>>> originalIterator;
		private final Function<Map.Entry<UK, UV>, R> resultMapper;
		private Map.Entry<UK, UV> nextUnexpired = null;
		private boolean rightAfterNextIsCalled = false;

		private EntriesIterator(
			@Nonnull Iterable<Map.Entry<UK, TtlValue<UV>>> withTs,
			@Nonnull Function<Map.Entry<UK, UV>, R> resultMapper) {
			this.originalIterator = withTs.iterator();
			this.resultMapper = resultMapper;
		}

		@Override
		public boolean hasNext() {
			rightAfterNextIsCalled = false;
			while (nextUnexpired == null && originalIterator.hasNext()) {
				nextUnexpired = getUnexpiredAndUpdateOrCleanup(originalIterator.next());
			}
			return nextUnexpired != null;
		}

		@Override
		public R next() {
			if (hasNext()) {
				rightAfterNextIsCalled = true;
				R result = resultMapper.apply(nextUnexpired);
				nextUnexpired = null;
				return result;
			}
			throw new NoSuchElementException();
		}

		@Override
		public void remove() {
			if (rightAfterNextIsCalled) {
				originalIterator.remove();
			} else {
				throw new IllegalStateException("next() has not been called or hasNext() has been called afterwards," +
					" remove() is supported only right after calling next()");
			}
		}

		private Map.Entry<UK, UV> getUnexpiredAndUpdateOrCleanup(Map.Entry<UK, TtlValue<UV>> e) {
			TtlValue<UV> unexpiredValue;
			try {
				unexpiredValue = getWrappedWithTtlCheckAndUpdate(
					e::getValue,
					v -> original.put(e.getKey(), v),
					originalIterator::remove);
			} catch (Exception ex) {
				throw new FlinkRuntimeException(ex);
			}
			return unexpiredValue == null ? null : new AbstractMap.SimpleEntry<>(e.getKey(), unexpiredValue.getUserValue());
		}
	}
}

```



可见，TtlMapState的增删改查操作都是在原MapState上进行，只是加上了TTL相关的逻辑，这也是装饰器模式的特点。例如，TtlMapState.get()方法调用了上述AbstractTtlDecorator.getWrappedWithTtlCheckAndUpdate()方法，传入的获取（getter）、插入（updater）和删除（stateClear）的逻辑就是原MapState的get()、put()和remove()方法。而TtlMapState.put()只是在调用原MapState的put()方法之前，将状态包装为TtlValue而已。




#### 增量清理策略





另外需要注意，所有增删改查操作之前都需要执行accessCallback.run()方法。如果启用了增量清理策略，该Runnable会通过在状态数据上维护一个全局迭代器向前清理过期数据。如果未启用增量清理策略，accessCallback为空。前文提到过的TtlStateFactory.registerTtlIncrementalCleanupCallback()方法如下



TtlStateFactory.registerTtlIncrementalCleanupCallback



```java

	private Runnable registerTtlIncrementalCleanupCallback(InternalKvState<?, ?, ?> originalState) {
		StateTtlConfig.IncrementalCleanupStrategy config =
			ttlConfig.getCleanupStrategies().getIncrementalCleanupStrategy();
		boolean cleanupConfigured = config != null && incrementalCleanup != null;
		boolean isCleanupActive = cleanupConfigured &&
			isStateIteratorSupported(originalState, incrementalCleanup.getCleanupSize());
		Runnable callback = isCleanupActive ? incrementalCleanup::stateAccessed : () -> { };
		if (isCleanupActive && config.runCleanupForEveryRecord()) {
			stateBackend.registerKeySelectionListener(stub -> callback.run());
		}
		return callback;
	}

```

实际清理的代码则位于TtlIncrementalCleanup类中，stateIterator就是状态数据的迭代器。



```java
private void runCleanup() {
		int entryNum = 0;
		Collection<StateEntry<K, N, S>> nextEntries;
		while (
			entryNum < cleanupSize &&
			stateIterator.hasNext() &&
			!(nextEntries = stateIterator.nextEntries()).isEmpty()) {

			for (StateEntry<K, N, S> state : nextEntries) {
				S cleanState = ttlState.getUnexpiredOrNull(state.getState());
				if (cleanState == null) {
					stateIterator.remove(state);
				} else if (cleanState != state.getState()) {
					stateIterator.update(state, cleanState);
				}
			}

			entryNum += nextEntries.size();
		}
	}
```



### Reference

https://blog.csdn.net/nazeniwaresakini/article/details/106094778



