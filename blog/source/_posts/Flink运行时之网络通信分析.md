---
title: Flink运行时之网络通信分析
date: 2019-10-11 22:06:04
tags:
categories:
	- Flink
---



### NetworkEnvironment


网络环境（NetworkEnvironment）是TaskManager进行网络通信的主对象，主要用于跟踪中间结果并负责所有的数据交换。每个TaskManager的实例都包含一个网络环境对象，在TaskManager启动时创建。NetworkEnvironment管理着多个协助通信的关键部件，它们是：



 part| 解释
---|---
NetworkBufferPool | 网络缓冲池，负责申请一个TaskManager的所有的内存段用作缓冲池；每个ResultPartition(等价于一个task一个)都有一个localBufferPool，与全局的NetworkBufferPool进行交互申请和释放内存段。
ConnectionManager|连接管理器，用于管理本地（远程）通信连接；
ResultPartitionManager|结果分区管理器，用于跟踪一个TaskManager上所有生产/消费相关的ResultPartition；主要就是用于track所有的result partitions。
TaskEventDispatcher|任务事件分发器，从消费者任务分发事件给生产者任务；
 ResultPartitionConsumableNotifier|结果分区可消费通知器，用于通知消费者生产者生产的结果分区可消费；

当NetworkEnvironment被初始化时，它首先根据配置创建网络缓冲池（NetworkBufferPool）。


![image](https://note.youdao.com/yws/api/personal/file/9F9468B08EAE48DEA8992F82719C7DD1?method=download&shareKey=733a66b12c5e31f28d063b7f815b444d)


```java
public NetworkEnvironment(
		int numBuffers,
		int memorySegmentSize,
		int partitionRequestInitialBackoff,
		int partitionRequestMaxBackoff,
		int networkBuffersPerChannel,
		int extraNetworkBuffersPerGate,
		boolean enableCreditBased) {
		this(
			new NetworkBufferPool(numBuffers, memorySegmentSize),
			new LocalConnectionManager(),
			new ResultPartitionManager(),
			new TaskEventDispatcher(),
			new KvStateRegistry(),
			null,
			null,
			IOManager.IOMode.SYNC,
			partitionRequestInitialBackoff,
			partitionRequestMaxBackoff,
			networkBuffersPerChannel,
			extraNetworkBuffersPerGate,
			enableCreditBased);
	}
```

初始化创建NetworkBufferPool时需要指定Buffer数目、单个Buffer的大小，并且**申请的网络buffer为OffHeapMemory**。使用array阻塞队列。
```java
ArrayBlockingQueue<MemorySegment> availableMemorySegments;
```
申请流程如下
```java
	try {
			for (int i = 0; i < numberOfSegmentsToAllocate; i++) {
				availableMemorySegments.add(MemorySegmentFactory.allocateUnpooledOffHeapMemory(segmentSize, null));
			}
		}
		catch (OutOfMemoryError err) {
```
关于NetworkBufferPool相关的在本章后面介绍。



在任务执行的核心逻辑中，有一个步骤是需要将自身（Task）注册到网络栈（也就是这里的NetworkEnvironment）。

该步骤会调用NetworkEnvironment的实例方法registerTask进行注册，注册之后NetworkEnvironment会对任务的通信进行管理：

```java
public void registerTask(Task task) throws IOException {
		//获得当前任务对象所生产的结果分区集合
		final ResultPartition[] producedPartitions = task.getProducedPartitions();

		synchronized (lock) {
			if (isShutdown) {
				throw new IllegalStateException("NetworkEnvironment is shut down");
			}

			for (final ResultPartition partition : producedPartitions) {
				setupPartition(partition);
			}

			//同时获得所有的数据的输入分区
			// Setup the buffer pool for each buffer reader
			final SingleInputGate[] inputGates = task.getAllInputGates();
			for (SingleInputGate gate : inputGates) {
				setupInputGate(gate);
			}
		}
	}
```

初始化结果分区和初始化输入分区，是两个重要的核心。

##### 初始化结果分区
核心操作是初始化localBufferPool，然后将localBufferPool注册到partition中，再将partition注册到结果分区管理器ResultPartitionManager中

```java
public void setupPartition(ResultPartition partition) throws IOException {
		BufferPool bufferPool = null;

		try {
			//此分区是否使用有限数量的（网络）缓冲区,如果是最大的MemorySegments为
			// 当前分区的消费者数量*每个传出/传入通道使用的网络缓冲区数+每个传出/传入门使用的额外网络缓冲区数,否则为int的最大值
			// 默认是 消费者数量*2+8
			int maxNumberOfMemorySegments = partition.getPartitionType().isBounded() ?
				partition.getNumberOfSubpartitions() * networkBuffersPerChannel +
					extraNetworkBuffersPerGate : Integer.MAX_VALUE;
			// If the partition type is back pressure-free, we register with the buffer pool for
			// callbacks to release memory.
			//如果分区类型是无压力的，我们在缓冲池中注册回调以释放内存。
			bufferPool = networkBufferPool.createBufferPool(partition.getNumberOfSubpartitions(),
				maxNumberOfMemorySegments,
				partition.getPartitionType().hasBackPressure() ? Optional.empty() : Optional.of(partition));
			//将本地缓冲池注册到结果分区
			partition.registerBufferPool(bufferPool);
			//结果分区会被注册到结果分区管理器
			resultPartitionManager.registerResultPartition(partition);
		} catch (Throwable t) {
			if (bufferPool != null) {
				bufferPool.lazyDestroy();
			}

			if (t instanceof IOException) {
				throw (IOException) t;
			} else {
				throw new IOException(t.getMessage(), t);
			}
		}

		taskEventDispatcher.registerPartition(partition.getPartitionId());
	}
```

##### 初始化输入分区

主要是判断是否是基于信道的通信方式，决定要申请多少的buffer，然后将创建好的localBufferPool注册到对应的inputGate中。
```java

@VisibleForTesting
	public void setupInputGate(SingleInputGate gate) throws IOException {
		BufferPool bufferPool = null;
		int maxNumberOfMemorySegments;
		try {
			if (enableCreditBased) {
				maxNumberOfMemorySegments = gate.getConsumedPartitionType().isBounded() ?
					extraNetworkBuffersPerGate : Integer.MAX_VALUE;

				// assign exclusive buffers to input channels directly and use the rest for floating buffers
				gate.assignExclusiveSegments(networkBufferPool, networkBuffersPerChannel);
				bufferPool = networkBufferPool.createBufferPool(0, maxNumberOfMemorySegments);
			} else {
				maxNumberOfMemorySegments = gate.getConsumedPartitionType().isBounded() ?
					gate.getNumberOfInputChannels() * networkBuffersPerChannel +
						extraNetworkBuffersPerGate : Integer.MAX_VALUE;

				bufferPool = networkBufferPool.createBufferPool(gate.getNumberOfInputChannels(),
					maxNumberOfMemorySegments);
			}
			gate.setBufferPool(bufferPool);
		} catch (Throwable t) {
			if (bufferPool != null) {
				bufferPool.lazyDestroy();
			}

			ExceptionUtils.rethrowIOException(t);
		}
	}
```


## 统一的数据交换对象

在Flink的执行引擎中，流动的元素主要有两种：缓冲（Buffer）和事件（Event）。Buffer主要针对用户数据交换，而Event则用于一些特殊的控制标识。但在实现时，为了在通信层统一数据交换，Flink提供了数据交换对象——BufferOrEvent。它是一个既可以表示Buffer又可以表示Event的类。上层使用者只需调用isBuffer和isEvent方法即可判断当前收到的这条数据是Buffer还是Event。

```java
/**
 * Either type for {@link Buffer} or {@link AbstractEvent} instances tagged with the channel index,
 * from which they were received.
 */
public class BufferOrEvent {

	private final Buffer buffer;

	private final AbstractEvent event;
	
}
```


#### 缓冲 Buffer

缓冲（Buffer）是数据交换的载体，几乎所有的数据（当然事件是特殊的）交换都需要经过Buffer。Buffer底层依赖于Flink自管理内存的内存段（MemorySegment）作为数据的容器。Buffer在内存段上做了一层封装，这一层封装是为了对基于引用计数的Buffer回收机制提供支持。

具体实现NetworkBuffer.java,这个类继承了netty中的AbstractReferenceCountedByteBuf并且实现了自定义的Buffer。

AbstractReferenceCountedByteBuf是netty中已经实现的引用计数的功能。通过这个类，可以判断这个buffer是否需要回收。

```java
public class NetworkBuffer extends AbstractReferenceCountedByteBuf implements Buffer {


    /**
	 * 保留此缓冲区以备将来使用，将参考计数器增加1
	 * @return
	 */
	@Override
	public NetworkBuffer retainBuffer() {
		return (NetworkBuffer) super.retain();
	}
	
	/**
	 * 释放一次该缓冲区，即减少参考计数并在参考计数达到0时回收缓冲区
	 */
	@Override
	public void recycleBuffer() {
		release();
	}
	
	/**
	 * 回收申请的内存
	 */
	@Override
	protected void deallocate() {
		recycler.recycle(memorySegment);
	}
	
}
```


它在内部维护着一个计数器referenceCount，初始值为1。内存回收由缓冲回收器（BufferRecycler）来完成，回收的对象就是内存段（MemorySegment）。

```java
	/** The recycler for the backing {@link MemorySegment}. */
	private final BufferRecycler recycler;
```
BufferRecycler接口有一个名为FreeingBufferRecycler的简单实现者，它的做法是直接释放内存段。当然通常为了分配和回收的效率，会对Buffer进行预先分配然后加入到Buffer池中。所以，BufferRecycler的常规实现是基于缓冲池的。






#### BufferRecycler的具体实现

#####  LocalBufferPool

```java
private final NetworkBufferPool networkBufferPool;
```
拥有一个NetworkBufferPool的属性，这个NetworkBufferPool是一个全局网络buffer池，一个TaskManger只有一个。

**NetWork BufferPool 是 TaskManager 内所有 Task 共享的 BufferPool，TaskManager 初始化时就会向堆外内存申请 NetWork BufferPool。LocalBufferPool 是每个 Task 自己的 BufferPool，假如一个 TaskManager 内运行着 5 个 Task，那么就会有 5 个 LocalBufferPool，但 TaskManager 内永远只有一个 NetWork BufferPool。**

```java
ArrayDeque<MemorySegment> availableMemorySegments = new ArrayDeque<MemorySegment>();
```
当前可用的内存段。 这些段是从网络缓冲池中请求的，目前尚未作为缓冲区实例分发。


```java
private final ArrayDeque<BufferListener> registeredListeners = new ArrayDeque<>();
```

监听buffer是否可用，如果可用，通过这个队列通知。

requestBuffer()调用requestMemorySegment()将获取到的MemorySegment封装成NetworkBuffer，以自身为recycler的入参。申请内存这一块是像全局NetWork BufferPool申请。


requestMemorySegment()方法是申请内存块的


```java
private MemorySegment requestMemorySegment(boolean isBlocking) throws InterruptedException, IOException {
		//返回多余的内存段
		// 如果有多余的内存段，recycle，直接add到networkBufferPool的availableMemorySegments中
		returnExcessMemorySegments();

		// fill availableMemorySegments with at least one element, wait if required
		while (true) {
			//请求内存段，如果有直接返回，如果没有，先释放owner中的内存，如果阻塞，那么等待2秒继续申请，如果不阻塞，返回null
			// 申请内存的操作是去通过  networkBufferPool的availableMemorySegments.poll去等待
			Optional<MemorySegment> segment = internalRequestMemorySegment();
			if (segment.isPresent()) {
				return segment.get();
			}

			if (owner.isPresent()) {
				owner.get().releaseMemory(1);
			}

			synchronized (availableMemorySegments) {
				if (isBlocking) {
					availableMemorySegments.wait(2000);
				}
				else {
					return null;
				}
			}
		}
	}
```


deallocate是一个在netty线程执行的函数，调用自定义的recycle函数。

```java

/**
	 * 回收申请的内存
	 */
	@Override
	protected void deallocate() {
		recycler.recycle(memorySegment);
	}
	
	
@Override
	public void recycle(MemorySegment segment) {
		BufferListener listener;
		NotificationResult notificationResult = NotificationResult.BUFFER_NOT_USED;
		while (!notificationResult.isBufferUsed()) {
			synchronized (availableMemorySegments) {
				//如果是多余的内存段或者已经销毁，直接返回该内存段
				if (isDestroyed || numberOfRequestedMemorySegments > currentPoolSize) {
					returnMemorySegment(segment);
					return;
				} else {
					//获取队列中等待的listener，通过这个listener。如果没有在等待中的 ，加入可获取的队列，notify。
					listener = registeredListeners.poll();
					if (listener == null) {
						availableMemorySegments.add(segment);
						availableMemorySegments.notify();
						return;
					}
				}
			}
			notificationResult = fireBufferAvailableNotification(listener, segment);
		}
	}
```


#### NetworkBufferPool

NetworkBufferPool是网络堆栈的 MemorySegment实例的固定大小的池。申请的是堆外内存。（**个人猜想网络是一块需要经常进行替换数据的地方，频繁的替换会发送大量的young gc，在java中，C c = new C，  c= new C；的方式旧的实例需要通过回收才可以消除，而通过堆外内存可以直接通过put byte直接在内存空间操作，不需要回收**。）


```java
/**
	 * Allocates all {@link MemorySegment} instances managed by this pool.
	 */
	public NetworkBufferPool(int numberOfSegmentsToAllocate, int segmentSize) {

		this.totalNumberOfMemorySegments = numberOfSegmentsToAllocate;
		this.memorySegmentSize = segmentSize;

		final long sizeInLong = (long) segmentSize;

		try {
			this.availableMemorySegments = new ArrayBlockingQueue<>(numberOfSegmentsToAllocate);
		}
		catch (OutOfMemoryError err) {
			throw new OutOfMemoryError("Could not allocate buffer queue of length "
					+ numberOfSegmentsToAllocate + " - " + err.getMessage());
		}

		try {
			//网络专用的buffer申请的是堆外内存
			for (int i = 0; i < numberOfSegmentsToAllocate; i++) {
				availableMemorySegments.add(MemorySegmentFactory.allocateUnpooledOffHeapMemory(segmentSize, null));
			}
		}
		catch (OutOfMemoryError err) {
			//如果对外内存不足，那么将信息全部报出来。
			int allocated = availableMemorySegments.size();

			// free some memory
			availableMemorySegments.clear();

			long requiredMb = (sizeInLong * numberOfSegmentsToAllocate) >> 20;
			long allocatedMb = (sizeInLong * allocated) >> 20;
			long missingMb = requiredMb - allocatedMb;

			throw new OutOfMemoryError("Could not allocate enough memory segments for NetworkBufferPool " +
					"(required (Mb): " + requiredMb +
					", allocated (Mb): " + allocatedMb +
					", missing (Mb): " + missingMb + "). Cause: " + err.getMessage());
		}

		long allocatedMb = (sizeInLong * availableMemorySegments.size()) >> 20;

		LOG.info("Allocated {} MB for network buffer pool (number of memory segments: {}, bytes per segment: {}).",
				allocatedMb, availableMemorySegments.size(), segmentSize);
	}
```

NetworkBufferPool中申请内存段和释放内存段，都是加入和放回自身的队列中。
每个task对应的localBufferPool需要buffer则去networkBufferPool的队列中获取。NetworkBufferPool有用的buffer数量，在初始化的时候就限定了。（这个数量是由配置计算出来的，具体见TaskManagerServices.calculateNetworkBufferMemory）



redistributeBuffers这个函数可以动态调节每个localPool的size，让手头空着的内存段，分配给更需要的本地pool中。通过
```
	/需要申请的数量/总需要申请的数量 求出百分比，去掉小数点。      
	现有内存段个数 * 百分比 - 已经分配为可以分配的
```
需要申请的段越多  百分比就越大，优先分配给权重大的
```java
private void redistributeBuffers() throws IOException {
		assert Thread.holdsLock(factoryLock);

		// All buffers, which are not among the required ones
		final int numAvailableMemorySegment = totalNumberOfMemorySegments - numTotalRequiredBuffers;

		//如果可用内存段为0，将所有的localBufferpool中申请的多余的buffer返还给队列
		if (numAvailableMemorySegment == 0) {
			// in this case, we need to redistribute buffers so that every pool gets its minimum
			for (LocalBufferPool bufferPool : allBufferPools) {
				bufferPool.setNumBuffers(bufferPool.getNumberOfRequiredMemorySegments());
			}
			return;
		}

		/*
		 * With buffer pools being potentially limited, let's distribute the available memory
		 * segments based on the capacity of each buffer pool, i.e. the maximum number of segments
		 * an unlimited buffer pool can take is numAvailableMemorySegment, for limited buffer pools
		 * it may be less. Based on this and the sum of all these values (totalCapacity), we build
		 * a ratio that we use to distribute the buffers.
		 */

		long totalCapacity = 0; // long to avoid int overflow

		//计算每个LocalBufferPool的还可以申请内存段之和
		for (LocalBufferPool bufferPool : allBufferPools) {
			int excessMax = bufferPool.getMaxNumberOfMemorySegments() -
				bufferPool.getNumberOfRequiredMemorySegments();
			totalCapacity += Math.min(numAvailableMemorySegment, excessMax);
		}

		// no capacity to receive additional buffers?
		if (totalCapacity == 0) {
			return; // necessary to avoid div by zero when nothing to re-distribute
		}

		// since one of the arguments of 'min(a,b)' is a positive int, this is actually
		// guaranteed to be within the 'int' domain
		// (we use a checked downCast to handle possible bugs more gracefully).
		final int memorySegmentsToDistribute = MathUtils.checkedDownCast(
				Math.min(numAvailableMemorySegment, totalCapacity));

		long totalPartsUsed = 0; // of totalCapacity
		int numDistributedMemorySegment = 0;
		for (LocalBufferPool bufferPool : allBufferPools) {
			int excessMax = bufferPool.getMaxNumberOfMemorySegments() -
				bufferPool.getNumberOfRequiredMemorySegments();

			// shortcut
			if (excessMax == 0) {
				continue;
			}

			totalPartsUsed += Math.min(numAvailableMemorySegment, excessMax);

			// avoid remaining buffers by looking at the total capacity that should have been
			// re-distributed up until here
			// the downcast will always succeed, because both arguments of the subtraction are in the 'int' domain
			//需要申请的数量/总需要申请的数量 求出百分比，去掉小数点。  * 现有内存段个数 * 百分比 - 已经分配为可以分配的
			final int mySize = MathUtils.checkedDownCast(
					memorySegmentsToDistribute * totalPartsUsed / totalCapacity - numDistributedMemorySegment);

			numDistributedMemorySegment += mySize;
			bufferPool.setNumBuffers(bufferPool.getNumberOfRequiredMemorySegments() + mySize);
		}

		assert (totalPartsUsed == totalCapacity);
		assert (numDistributedMemorySegment == memorySegmentsToDistribute);
	}
```

```java
@Nullable
	public MemorySegment requestMemorySegment() {
		return availableMemorySegments.poll();
	}

	public void recycle(MemorySegment segment) {
		// Adds the segment back to the queue, which does not immediately free the memory
		// however, since this happens when references to the global pool are also released,
		// making the availableMemorySegments queue and its contained object reclaimable
		availableMemorySegments.add(checkNotNull(segment));
	}
```


#### 配置天坑

```java
/**
	 * Fraction of JVM memory to use for network buffers.
	 */
	public static final ConfigOption<Float> NETWORK_BUFFERS_MEMORY_FRACTION =
			key("taskmanager.network.memory.fraction")
			.defaultValue(0.1f)
			.withDescription("Fraction of JVM memory to use for network buffers. This determines how many streaming" +
				" data exchange channels a TaskManager can have at the same time and how well buffered the channels" +
				" are. If a job is rejected or you get a warning that the system has not enough buffers available," +
				" increase this value or the min/max values below. Also note, that \"taskmanager.network.memory.min\"" +
				"` and \"taskmanager.network.memory.max\" may override this fraction.");
				
/**
	 * Minimum memory size for network buffers.
	 */
	public static final ConfigOption<String> NETWORK_BUFFERS_MEMORY_MIN =
			key("taskmanager.network.memory.min")
			.defaultValue("64mb")
			.withDescription("Minimum memory size for network buffers.");

	/**
	 * Maximum memory size for network buffers.
	 */
	public static final ConfigOption<String> NETWORK_BUFFERS_MEMORY_MAX =
			key("taskmanager.network.memory.max")
			.defaultValue("1gb")
			.withDescription("Maximum memory size for network buffers.");
```

通过配比*内存大小与 最小大小比，取最大值，再与最大值比取较小值。。结果就是哪怕你的配比配的再大，网络buffer也是1gb。。。转为Count为 1024*1024/32K= 32,768。。。如果需要调整网络参数，一定要把最大值和最小值一起配上。
```java
	final long networkBufBytes = Math.min(networkBufMax, Math.max(networkBufMin,
			(long) (jvmHeapNoNet / (1.0 - networkBufFraction) * networkBufFraction)));
```


#### 事件  Event

Flink的数据流中不仅仅只有用户的数据，还包含了一些特殊的事件，这些事件都是由算子注入到数据流中的。它们在每个流分区里伴随着其他的数据元素而被有序地分发。接收到这些事件的算子会对这些事件给出响应，典型的事件类型有：

* 检查点屏障：用于隔离多个检查点之间的数据，保障快照数据的一致性；
* 迭代屏障：标识流分区已到达了一个超级步的结尾；
* 子分区数据结束标记：当消费任务获取到该事件时，表示其所消费的对应的分区中的数据已被全部消费完成；

所有事件的最终基类都是AbstractEvent。AbstractEvent这一抽象类又派生出另一个抽象类RuntimeEvent，几乎所有预先内置的事件都直接派生于此。除了预定义的事件外，Flink还支持自定义的扩展事件，所有自定义的事件都继承自派生于AbstractEvent的TaskEvent。
    

#### ResultPartitionManager

ResultPartitionManager：结果分区管理器，用于跟踪一个TaskManager上所有生产/消费相关的ResultPartition；主要就是用于track所有的result partitions，核心结构为
```
Table<ExecutionAttemptID, IntermediateResultPartitionID, ResultPartition> registeredPartitions =HashBasedTable.create();
```

通过createSubpartitionView创建消费ResultSubpartition的视图ResultSubpartitionView，入参之一为BufferAvailabilityListener，是用来notify这个listener有数据到来，可以消费。

所以ResultSubpartition就是消费分区，ResultSubpartitionView是消费分区与消费者绑定在一起的视图。


#### TaskEventDispatcher

任务事件分发器，从消费者任务分发事件给生产者任务。

#### ConnectionManager

连接管理器，用于管理本地（远程）通信连接.存在两种模式，一种是本地，一种是netty远程。线上环境都是netty模式。

```java
NettyConfig nettyConfig = networkEnvironmentConfiguration.nettyConfig();
		if (nettyConfig != null) {
			connectionManager = new NettyConnectionManager(nettyConfig);
			enableCreditBased = nettyConfig.isCreditBasedEnabled();
		} else {
			connectionManager = new LocalConnectionManager();
		}

```
netty模式涉及到基于Netty的网络通信部分。后面再讲。





### 结果分区消费端


输入网关（InputGate）用于消费中间结果（IntermediateResult）在并行执行时由子任务生产的一个或多个结果分区（ResultPartition）。

Flink当前提供了两个输入网关的实现，分别是：

* SingleInputGate：常规输入网关；
* UnionInputGate：联合输入网关，它允许将多个输入网关联合起来；

我们主要分析SingleInputGate，因为它是消费ResultPartition的实体，而UnionInputGate主要充当InputGate容器的角色。


作为数据的消费者，InputGate最关键的方法自然是获取生产者所生产的缓冲区，提供该功能的方法为getNextBufferOrEvent，它返回的对象是我们后面谈到的统一的数据交换对象BufferOrEvent。
**
BufferOrEvent的直接消费对象是通信层API中的记录读取器，它会将Buffer中的数据反序列化为记录供上层任务使用。**

```java
private Optional<BufferOrEvent> getNextBufferOrEvent(boolean blocking) throws IOException, InterruptedException {
		//如果已接收到所有EndOfPartitionEvent事件，则说明每个ResultSubpartition中的数据都被消费完成
		if (hasReceivedAllEndOfPartitionEvents) {
			return Optional.empty();
		}

		if (isReleased) {
			throw new IllegalStateException("Released");
		}
		//触发所有的输入通道向ResultSubpartition发起请求
		requestPartitions();

		InputChannel currentChannel;
		boolean moreAvailable;
		Optional<BufferAndAvailability> result = Optional.empty();

		do {
			synchronized (inputChannelsWithData) {
				while (inputChannelsWithData.size() == 0) {
					if (isReleased) {
						throw new IllegalStateException("Released");
					}
					//如果是允许阻塞，等待数据
					if (blocking) {
						inputChannelsWithData.wait();
					}
					else {
						return Optional.empty();
					}
				}

				currentChannel = inputChannelsWithData.remove();
				enqueuedInputChannelsWithData.clear(currentChannel.getChannelIndex());
				moreAvailable = !inputChannelsWithData.isEmpty();
			}
			//从输入通道中获得下一个Buffer
			result = currentChannel.getNextBuffer();
		} while (!result.isPresent());

		// this channel was now removed from the non-empty channels queue
		// we re-add it in case it has more data, because in that case no "non-empty" notification
		// will come for that channel
		if (result.get().moreAvailable()) {
			queueChannel(currentChannel);
			moreAvailable = true;
		}

		final Buffer buffer = result.get().buffer();
		//如果该Buffer是用户数据，则构建BufferOrEvent对象并返回
		if (buffer.isBuffer()) {
			return Optional.of(new BufferOrEvent(buffer, currentChannel.getChannelIndex(), moreAvailable));
		}
		//否则把它当作事件来处理
		else {
			final AbstractEvent event = EventSerializer.fromBuffer(buffer, getClass().getClassLoader());

			//如果获取到的是标识某ResultSubpartition已经生产完数据的事件
			if (event.getClass() == EndOfPartitionEvent.class) {
				//对获取该ResultSubpartition的通道进行标记
				channelsWithEndOfPartitionEvents.set(currentChannel.getChannelIndex());
				//如果所有信道都被标记了，置全部通道获取数据完成
				if (channelsWithEndOfPartitionEvents.cardinality() == numberOfInputChannels) {
					// Because of race condition between:
					// 1. releasing inputChannelsWithData lock in this method and reaching this place
					// 2. empty data notification that re-enqueues a channel
					// we can end up with moreAvailable flag set to true, while we expect no more data.
					checkState(!moreAvailable || !pollNextBufferOrEvent().isPresent());
					moreAvailable = false;
					hasReceivedAllEndOfPartitionEvents = true;
				}
				//对外发出ResultSubpartition已被消费的通知同时释放资源
				currentChannel.notifySubpartitionConsumed();

				currentChannel.releaseAllResources();
			}
			//以事件来构建BufferOrEvent对象
			return Optional.of(new BufferOrEvent(event, currentChannel.getChannelIndex(), moreAvailable));
		}
	}
```
由于requestPartitions只是起到触发其内部的InputChannel去请求的作用，这个调用可能并不会阻塞等待远程数据被返回。因为不同的InputChannel其请求的机制并不相同，RemoteChannel就是利用Netty异步请求的

SingleInputGate.requestPartitions
```java

				for (InputChannel inputChannel : inputChannels.values()) {
					inputChannel.requestSubpartition(consumedSubpartitionIndex);
				}
```
所以SingleInputGate采用阻塞等待以及事件回调的方式来等待InputChannel上的数据可用。具体而言，它在while代码块中循环阻塞等待有可获取数据的InputChannel。而可用的InputChannel则由它们自己通过回调SingleInputGate的onAvailableBuffer添加到阻塞队列inputChannelsWithData中来。当有可获取数据的InputChannel之后，即可获取到Buffer。

#### UnionInputGate
UnionInputGate，它更像一个包含SingleInputGate的容器，同时可以这些SingleInputGate拥有的InputChannel联合起来。并且多数InputGate约定的接口方法的实现，都被委托给了每个SingleInputGate。

那么它在实现getNextBufferOrEvent方法的时候，到底从哪个InputGate来获得缓冲区呢。它采用的是事件通知机制，所有加入UnionInputGate的InputGate都会将自己注册到InputGateListener。当某个InputGate上有数据可获取，该InputGate将会被加入一个阻塞队列。接着我们再来看getNextBufferOrEvent方法的实现：

```java
public Optional<BufferOrEvent> getNextBufferOrEvent() throws IOException, InterruptedException {
		if (inputGatesWithRemainingData.isEmpty()) {
			return Optional.empty();
		}

		//遍历每个InputGate，依次调用其requestPartitions方法
		// Make sure to request the partitions, if they have not been requested before.
		requestPartitions();

		//阻塞等待输入网关队列中有可获取数据的输入网关
		InputGateWithData inputGateWithData = waitAndGetNextInputGate();
		//获取对应的输入网关和数据
		InputGate inputGate = inputGateWithData.inputGate;
		BufferOrEvent bufferOrEvent = inputGateWithData.bufferOrEvent;

		if (bufferOrEvent.moreAvailable()) {
			// this buffer or event was now removed from the non-empty gates queue
			// we re-add it in case it has more data, because in that case no "non-empty" notification
			// will come for that gate
			queueInputGate(inputGate);
		}

		//如果获取到的是事件且该事件为EndOfPartitionEvent且输入网关已完成
		if (bufferOrEvent.isEvent()
			&& bufferOrEvent.getEvent().getClass() == EndOfPartitionEvent.class
			&& inputGate.isFinished()) {

			checkState(!bufferOrEvent.moreAvailable());
			//尝试将该输入网关从仍然可消费数据的输入网关集合中删除
			if (!inputGatesWithRemainingData.remove(inputGate)) {
				throw new IllegalStateException("Couldn't find input gate in set of remaining " +
					"input gates.");
			}
		}

		//获得通道索引偏移
		// Set the channel index to identify the input channel (across all unioned input gates)
		final int channelIndexOffset = inputGateToIndexOffsetMap.get(inputGate);

		//计算真实通道索引
		bufferOrEvent.setChannelIndex(channelIndexOffset + bufferOrEvent.getChannelIndex());
		bufferOrEvent.setMoreAvailable(bufferOrEvent.moreAvailable() || inputGateWithData.moreInputGatesAvailable);

		return Optional.of(bufferOrEvent);
	}
```


### 输入通道
一个InputGate包含多个输入通道（InputChannel），输入通道用于请求ResultSubpartitionView，并从中消费数据。

```
所谓的ResultSubpartitionView是由ResultSubpartition所创建的用于供消费者任务消费数据的视图对象。
```
对于每个InputChannel，消费的生命周期会经历如下的方法调用过程：

* requestSubpartition：请求ResultSubpartition；
* getNextBuffer：获得下一个Buffer；
* releaseAllResources：释放所有的相关资源；

InputChannel根据ResultPartitionLocation提供了三种实现：

* LocalInputChannel：用于请求同实例中生产者任务所生产的ResultSubpartitionView的输入通道；
* RemoteInputChannel：用于请求远程生产者任务所生产的ResultSubpartitionView的输入通道；
* UnknownInputChannel：一种用于占位目的的输入通道，需要占位通道是因为暂未确定相对于生产者任务位置，但最终要么被替换为RemoteInputChannel，要么被替换为LocalInputChannel。

LocalInputChannel会从相同的JVM实例中消费生产者任务所生产的Buffer。因此，这种模式是直接借助于方法调用和对象共享的机制完成消费，无需跨节点网络通信。具体而言，它是通过ResultPartitionManager来直接创建对应的ResultSubpartitionView的实例，这种通道相对简单。

RemoteInputChannel是我们重点关注的输入通道，因为它涉及到远程请求结果子分区。远程数据交换的通信机制建立在Netty框架的基础之上，因此会有一个主交互对象PartitionRequestClient来衔接通信层跟输入通道。


我们以请求子分区的requestSubpartition为入口来进行分析。首先，通过一个ConnectionManager根据连接编号（对应着目的主机）来创建PartitionRequestClient实例。
```java
partitionRequestClient = connectionManager
				.createPartitionRequestClient(connectionId);
```


接着具体的请求工作被委托给PartitionRequestClient的实例：

```java
partitionRequestClient.requestSubpartition(partitionId, subpartitionIndex, this, 0);

```
Netty以异步的方式处理请求。因此，上面的代码段中会看到将代表当前RemoteChannel实例的this对象作为参数注入到Netty的特定的ChannelHandler中去，在处理时根据特定的处理逻辑会触发RemoteChannel中相应的回调方法。

在RemoteChannel中定义了多个“onXXX”回调方法来衔接Netty的事件回调。其中，较为关键的自然是接收到数据的onBuffer方法：

RemoteInputChannel.java
```java
public void onBuffer(Buffer buffer, int sequenceNumber, int backlog) throws IOException {
		boolean recycleBuffer = true;

		try {

			final boolean wasEmpty;
			synchronized (receivedBuffers) {
				// Similar to notifyBufferAvailable(), make sure that we never add a buffer
				// after releaseAllResources() released all buffers from receivedBuffers
				// (see above for details).
				if (isReleased.get()) {
					return;
				}

				//如果实际序列号跟所期待的序列号不一致，则会触发onError回调，并相应以一个特定的异常对象
				//该方法调用在成功设置完错误原因后，同样会触发notifyAvailableBuffer方法调用
				if (expectedSequenceNumber != sequenceNumber) {
					onError(new BufferReorderingException(expectedSequenceNumber, sequenceNumber));
					return;
				}

				wasEmpty = receivedBuffers.isEmpty();
				//将数据加入接收队列同时将预期序列号计数器加一
				receivedBuffers.add(buffer);
				recycleBuffer = false;
			}

			++expectedSequenceNumber;

			if (wasEmpty) {
				notifyChannelNonEmpty();
			}

			if (backlog >= 0) {
				onSenderBacklog(backlog);
			}
		} finally {
			if (recycleBuffer) {
				buffer.recycleBuffer();
			}
		}
	}
```

onBuffer方法的执行处于Netty的I/O线程上，但RemoteInputChannel中getNextBuffer却不会在Netty的I/O线程上被调用，所以必须有一个数据共享的容器，这个容器就是receivedBuffers队列。getNextBuffer就是直接从receivedBuffers队列中出队一条数据然后返回。



#### Reference

https://zhuanlan.zhihu.com/p/35008079

https://blog.csdn.net/yanghua_kobe/article/details/53648748

https://blog.csdn.net/yanghua_kobe/article/details/53946640

https://blog.csdn.net/yanghua_kobe/article/details/54089128

https://www.jianshu.com/p/2779e73abcb8


https://www.jianshu.com/p/c261307757c4

基于netty通信

https://blog.csdn.net/yanghua_kobe/article/details/54233945