---
title: Flink源码解析-Timer
date: 2020-06-07 16:28:48
tags:
categories:
	- Flink
---


Timer在窗口机制中也有重要的地位。提起窗口自然就能想到Trigger，即触发器。


![image](https://note.youdao.com/yws/api/personal/file/B6314B8813F54E19ACF301EFE6D9194D?method=download&shareKey=8ac8a0de69cbddd4233cd6706889388d)

来看下Flink自带的EventTimeTrigger的部分代码，它是事件时间特征下的默认触发器。

```java
@Override
	public TriggerResult onElement(Object element, long timestamp, TimeWindow window, TriggerContext ctx) throws Exception {
		if (window.maxTimestamp() <= ctx.getCurrentWatermark()) {
			// if the watermark is already past the window fire immediately
			// allowedLateness的存在，会出现来的数据比水位线小，那么就直接出发计算
			return TriggerResult.FIRE;
		} else {
			ctx.registerEventTimeTimer(window.maxTimestamp());
			return TriggerResult.CONTINUE;
		}
	}
	
	@Override
	public TriggerResult onEventTime(long time, TimeWindow window, TriggerContext ctx) {
		return time == window.maxTimestamp() ?
			TriggerResult.FIRE :
			TriggerResult.CONTINUE;
	}
```

ctx.registerEventTimeTimer(window.maxTimestamp());的内部调用了internalTimerService.registerEventTimeTimer(window, time);

internalTimerService是一个接口

```java
public interface InternalTimerService<N> {
    long currentProcessingTime();
    long currentWatermark();
 
    void registerProcessingTimeTimer(N namespace, long time);
    void deleteProcessingTimeTimer(N namespace, long time);
 
    void registerEventTimeTimer(N namespace, long time);
    void deleteEventTimeTimer(N namespace, long time);
 
    // ...

```
对应的实现类InternalTimerServiceImpl

```java
 private final ProcessingTimeService processingTimeService;
    private final KeyGroupedInternalPriorityQueue<TimerHeapInternalTimer<K, N>> processingTimeTimersQueue;
    private final KeyGroupedInternalPriorityQueue<TimerHeapInternalTimer<K, N>> eventTimeTimersQueue;
    private ScheduledFuture<?> nextTimer;
 
    @Override
    public void registerProcessingTimeTimer(N namespace, long time) {
        InternalTimer<K, N> oldHead = processingTimeTimersQueue.peek();
        if (processingTimeTimersQueue.add(new TimerHeapInternalTimer<>(time, (K) keyContext.getCurrentKey(), namespace))) {
            long nextTriggerTime = oldHead != null ? oldHead.getTimestamp() : Long.MAX_VALUE;
            if (time < nextTriggerTime) {
                if (nextTimer != null) {
                    nextTimer.cancel(false);
                }
                nextTimer = processingTimeService.registerTimer(time, this);
            }
        }
    }
 
    @Override
    public void registerEventTimeTimer(N namespace, long time) {
        eventTimeTimersQueue.add(new TimerHeapInternalTimer<>(time, (K) keyContext.getCurrentKey(), namespace));
    }
 
    @Override
    public void deleteProcessingTimeTimer(N namespace, long time) {
        processingTimeTimersQueue.remove(new TimerHeapInternalTimer<>(time, (K) keyContext.getCurrentKey(), namespace));
    }
 
    @Override
    public void deleteEventTimeTimer(N namespace, long time) {
        eventTimeTimersQueue.remove(new TimerHeapInternalTimer<>(time, (K) keyContext.getCurrentKey(), namespace));
    }

```

注册Timer实际上就是为它们赋予对应的时间戳、key和命名空间，并将它们加入对应的优先队列。

特别地，当注册基于处理时间的Timer时，会先检查要注册的Timer时间戳与当前在最小堆堆顶的Timer的时间戳的大小关系。如果前者比后者要早，就会用前者替代掉后者，因为处理时间是永远线性增长的，得先处理时间比较靠前的。


### Timer注册好了之后是如何触发的呢？

#### eventTime

事件时间与内部时间戳无关，而与水印有关.

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

从队列头一个个取，如果获取的时间小于水位线，该任务需要处理，那就poll出来，调用trigger的onEvnetTime函数，执行方法。

向上追溯，回到InternalTimeServiceManager的同名方法。

```java
public void advanceWatermark(Watermark watermark) throws Exception {
        for (InternalTimerServiceImpl<?, ?> service : timerServices.values()) {
            service.advanceWatermark(watermark.getTimestamp());
        }
    }

```

继续向上追溯，到达终点：算子基类AbstractStreamOperator中处理水印的方法processWatermark()。当水印到来时，就会按着上述调用链流转到InternalTimerServiceImpl中，并触发所有早于水印时间戳的Timer了。

```java
public void processWatermark(Watermark mark) throws Exception {
        if (timeServiceManager != null) {
            timeServiceManager.advanceWatermark(mark);
        }
        output.emitWatermark(mark);
    }
```

#### processTime

```java
private void onProcessingTime(long time) throws Exception {
		// null out the timer in case the Triggerable calls registerProcessingTimeTimer()
		// inside the callback.
		nextTimer = null;

		InternalTimer<K, N> timer;

		while ((timer = processingTimeTimersQueue.peek()) != null && timer.getTimestamp() <= time) {
			processingTimeTimersQueue.poll();
			keyContext.setCurrentKey(timer.getKey());
			triggerTarget.onProcessingTime(timer);
		}

		if (timer != null && nextTimer == null) {
			nextTimer = processingTimeService.registerTimer(timer.getTimestamp(), this::onProcessingTime);
		}
	}
```

当onProcessingTime()方法被触发回调时，就会按顺序从队列中获取到比时间戳time小的所有Timer，并挨个执行Triggerable.onProcessingTime()方法

然后从获取的最新的赋值给nextTimer。


这个方法用于构造函数的时候

```java
processingTimeService.registerTimer(headTimer.getTimestamp(), this::onProcessingTime);
```

来到ProcessingTimeService的实现类SystemProcessingTimeService，它是用调度线程池实现回调的。

```java
	@Override
	public ScheduledFuture<?> registerTimer(long timestamp, ProcessingTimeCallback callback) {

		// delay the firing of the timer by 1 ms to align the semantics with watermark. A watermark
		// T says we won't see elements in the future with a timestamp smaller or equal to T.
		// With processing time, we therefore need to delay firing the timer by one ms.
		long delay = Math.max(timestamp - getCurrentProcessingTime(), 0) + 1;

		// we directly try to register the timer and only react to the status on exception
		// that way we save unnecessary volatile accesses for each timer
		try {
			return timerService.schedule(wrapOnTimerCallback(callback, timestamp), delay, TimeUnit.MILLISECONDS);
		}
		catch (RejectedExecutionException e) {
			final int status = this.status.get();
			if (status == STATUS_QUIESCED) {
				return new NeverCompleteFuture(delay);
			}
			else if (status == STATUS_SHUTDOWN) {
				throw new IllegalStateException("Timer service is shut down");
			}
			else {
				// something else happened, so propagate the exception
				throw e;
			}
		}
	}
```
ScheduledTask

```java
        @Override
		public void run() {
			if (serviceStatus.get() != STATUS_ALIVE) {
				return;
			}
			try {
				callback.onProcessingTime(nextTimestamp);
			} catch (Exception ex) {
				exceptionHandler.handleException(ex);
			}
			nextTimestamp += period;
		}

```

onProcessingTime()在TriggerTask线程中被回调，而TriggerTask线程按照Timer的时间戳来调度。到这里，处理时间Timer的情况就讲述完毕了。


两个模式，其实主要是有一个TimersQueue的队列，将对应的window加入队列中，如果是processTime，就会在ScheduledThreadPoolExecutor中调度一个又一个线程，每个线程的以延迟时间作为调度的条件。event不需要调度线程，根据watermark来调度，新的watermark到来就会触发。

如果出现timer的取消，就会删除TimersQueue的队列中的数据，但是ScheduledThreadPoolExecutor不会调出对应的数据，触发了找不到合适的数据而已。


### TimerHeapInternalTimer

在InternalTimerServiceImpl中有这么一个类TimerHeapInternalTimer。

```java
/** The key for which the timer is scoped. */
	@Nonnull
	private final K key;

	/** The namespace for which the timer is scoped. */
	@Nonnull
	private final N namespace;

	/** The expiration timestamp. */
	private final long timestamp;

	/**
	 * This field holds the current physical index of this timer when it is managed by a timer heap so that we can
	 * support fast deletes.
	 */
	private transient int timerHeapIndex;
```



可见，Timer的scope有两个，一是数据的key，二是命名空间。但是用户不会感知到命名空间的存在，所以我们可以简单地认为Timer是以key级别注册的（Timer四大特点之1）。正确估计key的量可以帮助我们控制Timer的量。

timerHeapIndex是这个Timer在优先队列里存储的下标。优先队列通常用二叉堆实现，而二叉堆可以直接用数组存储，所以让Timer持有其对应的下标可以较快地从队列里删除它。

comparePriorityTo()方法则用于确定Timer的优先级，显然Timer的优先队列是一个按Timer时间戳为关键字排序的最小堆。下面粗略看看该最小堆的实现。



### HeapPriorityQueueSet


在InternalTimeServiceManager用于管理各个InternalTimeService。

有一个工厂用于创建各种Timer队列，根据设置的状态后台而定。


```java
private <N> KeyGroupedInternalPriorityQueue<TimerHeapInternalTimer<K, N>> createTimerPriorityQueue(
		String name,
		TimerSerializer<K, N> timerSerializer) {
		return priorityQueueSetFactory.create(
			name,
			timerSerializer);
	}
```

PriorityQueueSetFactory目前有heap和rockersdb，主要看heap的。

要搞懂它，必须解释一下KeyGroup和KeyGroupRange。KeyGroup是Flink内部KeyedState的原子单位，亦即一些key的组合。一个Flink App的KeyGroup数量与最大并行度相同，将key分配到KeyGroup的操作则是经典的取hashCode+取模。而KeyGroupRange则是一些连续KeyGroup的范围，每个Flink sub-task都只包含一个KeyGroupRange。也就是说，KeyGroupRange可以看做当前sub-task在本地维护的所有key。

解释完毕。容易得知，上述代码中的那个HashMap<T, T>[]数组就是在KeyGroup级别对key进行去重的容器，数组中每个元素对应一个KeyGroup。以插入一个Timer的流程为例：

* 从Timer中取出key，计算该key属于哪一个KeyGroup；
* 计算出该KeyGroup在整个KeyGroupRange中的偏移量，按该偏移量定位到HashMap<T, T>[]数组的下标；
* 根据putIfAbsent()方法的语义，只有当对应HashMap不存在该Timer的key时，才将Timer插入最小堆中。


### 二叉堆


二叉堆是完全二元树或者是近似完全二元树，按照数据的排列方式可以分为两种：最大堆和最小堆。
最大堆：父结点的键值总是大于或等于任何一个子节点的键值；最小堆：父结点的键值总是小于或等于任何一个子节点的键值。

![image](https://note.youdao.com/yws/api/personal/file/1FFD942ED9C14290871ADD39747BB686?method=download&shareKey=45f1b15991d2d142485056eff53b5709)

图文解析是以"最大堆"来进行介绍的。

最大堆的核心内容是"添加"和"删除"，理解这两个算法，二叉堆也就基本掌握了

#### 1. 添加

假设在最大堆[90,80,70,60,40,30,20,10,50]种添加85，需要执行的步骤如下：
![image](https://note.youdao.com/yws/api/personal/file/E8EB673BDD764441AE96D762F4C25EB3?method=download&shareKey=e0b6e01649f7b6c264b1d59d67a534b7)


如上图所示，当向最大堆中添加数据时：先将数据加入到最大堆的最后，然后尽可能把这个元素往上挪，直到挪不动为止！
将85添加到[90,80,70,60,40,30,20,10,50]中后，最大堆变成了[90,85,70,60,80,30,20,10,50,40]。

```java

/*
 * 最大堆的向上调整算法(从start开始向上直到0，调整堆)
 *
 * 注：数组实现的堆中，第N个节点的左孩子的索引值是(2N+1)，右孩子的索引是(2N+2)。
 *
 * 参数说明：
 *     start -- 被上调节点的起始位置(一般为数组中最后一个元素的索引)
 */
protected void filterup(int start) {
    int c = start;            // 当前节点(current)的位置
    int p = (c-1)/2;        // 父(parent)结点的位置 
    T tmp = mHeap.get(c);        // 当前节点(current)的大小

    while(c > 0) {
        int cmp = mHeap.get(p).compareTo(tmp);
        if(cmp >= 0)
            break;
        else {
            mHeap.set(c, mHeap.get(p));
            c = p;
            p = (p-1)/2;   
        }       
    }
    mHeap.set(c, tmp);
}
  
/* 
 * 将data插入到二叉堆中
 */
public void insert(T data) {
    int size = mHeap.size();

    mHeap.add(data);    // 将"数组"插在表尾
    filterup(size);        // 向上调整堆
}

```
insert(data)的作用：将数据data添加到最大堆中。mHeap是动态数组ArrayList对象。
当堆已满的时候，添加失败；否则data添加到最大堆的末尾。然后通过上调算法重新调整数组，使之重新成为最大堆。

#### 2. 删除


假设从最大堆[90,85,70,60,80,30,20,10,50,40]中删除90，需要执行的步骤如下：

![image](https://note.youdao.com/yws/api/personal/file/1D2EB7C7AEBA4C4391ED232D245AD43A?method=download&shareKey=2654e662d1730a1ecaffd9c92104cb8c)

如上图所示，当从最大堆中删除数据时：先删除该数据，然后用最大堆中最后一个的元素插入这个空位；接着，把这个“空位”尽量往上挪，直到剩余的数据变成一个最大堆。
从[90,85,70,60,80,30,20,10,50,40]删除90之后，最大堆变成了[85,80,70,60,40,30,20,10,50]。


注意：考虑从最大堆[90,85,70,60,80,30,20,10,50,40]中删除60，执行的步骤不能单纯的用它的字节点来替换；而必须考虑到"替换后的树仍然要是最大堆"！


![image](https://note.youdao.com/yws/api/personal/file/30C5525A0DA74625B75E068B488E36FD?method=download&shareKey=5c53c7709352fa11a7e0a3ec2f27a8bc)


```java
/* 
 * 最大堆的向下调整算法
 *
 * 注：数组实现的堆中，第N个节点的左孩子的索引值是(2N+1)，右孩子的索引是(2N+2)。
 *
 * 参数说明：
 *     start -- 被下调节点的起始位置(一般为0，表示从第1个开始)
 *     end   -- 截至范围(一般为数组中最后一个元素的索引)
 */
protected void filterdown(int start, int end) {
    int c = start;          // 当前(current)节点的位置
    int l = 2*c + 1;     // 左(left)孩子的位置
    T tmp = mHeap.get(c);    // 当前(current)节点的大小

    while(l <= end) {
        int cmp = mHeap.get(l).compareTo(mHeap.get(l+1));
        // "l"是左孩子，"l+1"是右孩子
        if(l < end && cmp<0)
            l++;        // 左右两孩子中选择较大者，即mHeap[l+1]
        cmp = tmp.compareTo(mHeap.get(l));
        if(cmp >= 0)
            break;        //调整结束
        else {
            mHeap.set(c, mHeap.get(l));
            c = l;
            l = 2*l + 1;   
        }       
    }   
    mHeap.set(c, tmp);
}

/*
 * 删除最大堆中的data
 *
 * 返回值：
 *      0，成功
 *     -1，失败
 */
public int remove(T data) {
    // 如果"堆"已空，则返回-1
    if(mHeap.isEmpty() == true)
        return -1;

    // 获取data在数组中的索引
    int index = mHeap.indexOf(data);
    if (index==-1)
        return -1;

    int size = mHeap.size();
    mHeap.set(index, mHeap.get(size-1));// 用最后元素填补
    mHeap.remove(size - 1);                // 删除最后的元素

    if (mHeap.size() > 1)
        filterdown(index, mHeap.size()-1);    // 从index号位置开始自上向下调整为最小堆

    return 0;
}
```

### 最小堆的实现HeapPriorityQueue

在flink中因为要获取的是最小的Timer，用的是小顶堆。

如下是flink中添加数据的实现
```java
    @Override
	protected void addInternal(@Nonnull T element) {
	    // 用数组实现的，需要先扩容
		final int newSize = increaseSizeByOne();
		// 将数据放入新扩容的位置，堆尾
		moveElementToIdx(element, newSize);
		// 开始调整堆
		siftUp(newSize);
	}
	
	private void siftUp(int idx) {
		final T[] heap = this.queue;
		final T currentElement = heap[idx];
		int parentIdx = idx >>> 1;

		while (parentIdx > 0 && isElementPriorityLessThen(currentElement, heap[parentIdx])) {
			moveElementToIdx(heap[parentIdx], idx);
			idx = parentIdx;
			parentIdx >>>= 1;
		}

		moveElementToIdx(currentElement, idx);
	}
	
	
	private boolean isElementPriorityLessThen(T a, T b) {
		return elementPriorityComparator.comparePriority(a, b) < 0;
	}
	
	private int increaseSizeByOne() {
		final int oldArraySize = queue.length;
		final int minRequiredNewSize = ++size;
		if (minRequiredNewSize >= oldArraySize) {
			final int grow = (oldArraySize < 64) ? oldArraySize + 2 : oldArraySize >> 1;
			resizeQueueArray(oldArraySize + grow, minRequiredNewSize);
		}
		// TODO implement shrinking as well?
		return minRequiredNewSize;
	}
```


删除数据

```java
    @Override
	protected T removeInternal(int removeIdx) {
		T[] heap = this.queue;
		T removedValue = heap[removeIdx];

		assert removedValue.getInternalIndex() == removeIdx;

		final int oldSize = size;

		if (removeIdx != oldSize) {
			T element = heap[oldSize];
			moveElementToIdx(element, removeIdx);
			adjustElementAtIndex(element, removeIdx);
		}

		heap[oldSize] = null;

		--size;
		return removedValue;
	}
	
	private void adjustElementAtIndex(T element, int index) {
		siftDown(index);
		if (queue[index] == element) {
			siftUp(index);
		}
	}
	
	
	private void siftDown(int idx) {
		final T[] heap = this.queue;
		final int heapSize = this.size;

		final T currentElement = heap[idx];
		int firstChildIdx = idx << 1;
		int secondChildIdx = firstChildIdx + 1;

		if (isElementIndexValid(secondChildIdx, heapSize) &&
			isElementPriorityLessThen(heap[secondChildIdx], heap[firstChildIdx])) {
			firstChildIdx = secondChildIdx;
		}

		while (isElementIndexValid(firstChildIdx, heapSize) &&
			isElementPriorityLessThen(heap[firstChildIdx], currentElement)) {
			moveElementToIdx(heap[firstChildIdx], idx);
			idx = firstChildIdx;
			firstChildIdx = idx << 1;
			secondChildIdx = firstChildIdx + 1;

			if (isElementIndexValid(secondChildIdx, heapSize) &&
				isElementPriorityLessThen(heap[secondChildIdx], heap[firstChildIdx])) {
				firstChildIdx = secondChildIdx;
			}
		}

		moveElementToIdx(currentElement, idx);
	}
```
### timer on RocksDb


在InternalTimeServiceManager中存在一个工厂PriorityQueueSetFactory，根据选择的状态后端决定Timer是heap还是rocksdb，rocksdb对应的工厂为RocksDBPriorityQueueSetFactory。



对应的队列实现RocksDBCachingPriorityQueueSet中，



```java
/** Cache for the head element in de-serialized form. */
// 关于head元素的缓存。这个是维护的全局变量，只有在堆顶改变后在会置为null
@Nullable
private E peekCache;

// 保存了在rocksdb的头部的元素缓存
/** In memory cache that holds a head-subset of the elements stored in RocksDB. */
@Nonnull
private final OrderedByteArraySetCache orderedCache;
```





```java
@Nullable
@Override
public E peek() {
   
   checkRefillCacheFromStore();

    // 如果全局peekCache有，直接返回，如果没有为null
   if (peekCache != null) {
      return peekCache;
   }

    // 直接获取第一次元素，逆序列化后返回
   byte[] firstBytes = orderedCache.peekFirst();
   if (firstBytes != null) {
      peekCache = deserializeElement(firstBytes);
      return peekCache;
   } else {
      return null;
   }
}
```



```java
@Nullable
@Override
// 如果peekCache有，那么返回的peekCache，并删除rocksdb头部。如果没有peek，将获取的头部逆序列化返回
public E poll() {

   checkRefillCacheFromStore();
	// poll最新的数据
   final byte[] firstBytes = orderedCache.pollFirst();

   if (firstBytes == null) {
      return null;
   }

   // write-through sync
    // 从rocksdb中删除
   removeFromRocksDB(firstBytes);

   if (orderedCache.isEmpty()) {
      seekHint = firstBytes;
   }

   if (peekCache != null) {
      E fromCache = peekCache;
      peekCache = null;
      return fromCache;
   } else {
      return deserializeElement(firstBytes);
   }
}
```



```java
@Override
public boolean add(@Nonnull E toAdd) {

   checkRefillCacheFromStore();

   final byte[] toAddBytes = serializeElement(toAdd);

   final boolean cacheFull = orderedCache.isFull();

    // 如果cache没满并且之前所有元素都在cache中了 
   if ((!cacheFull && allElementsInCache) ||
       // 新加入的元素的优先级通过byte数组的优先级比较发现应该在堆顶
      LEXICOGRAPHIC_BYTE_COMPARATOR.compare(toAddBytes, orderedCache.peekLast()) < 0) {

      if (cacheFull) {
         // we drop the element with lowest priority from the cache
         orderedCache.pollLast();
         // the dropped element is now only in the store
         allElementsInCache = false;
      }

       // 用来判重
      if (orderedCache.add(toAddBytes)) {
         // write-through sync
         addToRocksDB(toAddBytes);
         if (toAddBytes == orderedCache.peekFirst()) {
             // 说明新的写入导致了堆顶变化
            peekCache = null;
            return true;
         }
      }
   } else {
       // 如果cache满了，或者不是所有的元素都在cache中，说明新来的数据一定不是堆顶的数据
      // we only added to the store
      addToRocksDB(toAddBytes);
      allElementsInCache = false;
   }
   return false;
}
```

### Referenece

https://blog.csdn.net/nazeniwaresakini/article/details/104220113


http://aitozi.com/flink-timerservice-based-on-rocksdb.html

https://www.jianshu.com/p/7e8ff8675639

https://www.cnblogs.com/skywang12345/p/3610390.html