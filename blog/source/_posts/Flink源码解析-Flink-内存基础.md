---
title: Flink源码解析 Flink 内存基础
date: 2019-09-11 22:34:06
tags:
categories:
	- Flink
---

### Flink抽象出的内存类型
* HEAP：JVM堆内存
* OFF_HEAP：非堆内存

这在Flink中被定义为一个枚举类型：MemoryType。

```java
@Internal
public enum MemoryType {

	/**
	 * Denotes memory that is part of the Java heap.
	 */
	HEAP,

	/**
	 * Denotes memory that is outside the Java heap (but still part of tha Java process).
	 */
	OFF_HEAP
}
```

### MemorySegment

Flink所管理的内存被抽象为数据结构：MemorySegment。内存管理的最小模块。

HeapMemorySegment(弃用)和HybridMemorySegment是对MemorySegment的实现。

![image](https://note.youdao.com/yws/api/personal/file/19A35F484C1943729D5838E65CC5D1A2?method=download&shareKey=93d97caa3cdba310ff5608d651bda411)

这两个的差别在HybridMemorySegment包含HeapMemorySegment的功能，
但对单个字节的操作效率稍差。


MemorySegment有两个构造函数,分别针对堆内内存和堆外内存。

```java
MemorySegment(byte[] buffer, Object owner) {
		if (buffer == null) {
			throw new NullPointerException("buffer");
		}

		this.heapMemory = buffer;
		this.address = BYTE_ARRAY_BASE_OFFSET;
		this.size = buffer.length;
		this.addressLimit = this.address + this.size;
		this.owner = owner;
	}
```

```java
MemorySegment(long offHeapAddress, int size, Object owner) {
		if (offHeapAddress <= 0) {
			throw new IllegalArgumentException("negative pointer or size");
		}
		if (offHeapAddress >= Long.MAX_VALUE - Integer.MAX_VALUE) {
			// this is necessary to make sure the collapsed checks are safe against numeric overflows
			throw new IllegalArgumentException("Segment initialized with too large address: " + offHeapAddress
					+ " ; Max allowed address is " + (Long.MAX_VALUE - Integer.MAX_VALUE - 1));
		}

		this.heapMemory = null;
		this.address = offHeapAddress;
		this.addressLimit = this.address + size;
		this.size = size;
		this.owner = owner;
	}
```

![image](https://note.youdao.com/yws/api/personal/file/011D7C8167754FD6ABCE456E14027D6E?method=download&shareKey=e6ed309d42f4e524243d1198119d3ebc)

* UNSAFE : 用来对堆/非堆内存进行操作，是JVM的非安全的API
* BYTE_ARRAY_BASE_OFFSET : 二进制字节数组的起始索引，相对于字节数组对象
* LITTLE_ENDIAN ： 布尔值，是否为小端对齐（涉及到字节序的问题）
* heapMemory : 如果为堆内存，则指向访问的内存的引用，否则若内存为非堆内存，则为null
* address : 字节数组对应的相对地址（若heapMemory为null，即可能为off-heap内存的绝对地址，后续会详解）
* addressLimit : 标识地址结束位置（address+size）
* size : 内存段的字节数


提供了一大堆get/put方法，这些getXXX/putXXX大都直接或者间接调用了unsafe.getXXX/unsafe.putXXX。



MemorySegment的下面几个方法需要关注一下：

```java

/**
	 * Bulk copy method. Copies {@code numBytes} bytes from this memory segment, starting at position
	 * {@code offset} to the target memory segment. The bytes will be put into the target segment
	 * starting at position {@code targetOffset}.
	 *
	 * @param offset The position where the bytes are started to be read from in this memory segment.
	 * @param target The memory segment to copy the bytes to.
	 * @param targetOffset The position in the target memory segment to copy the chunk to.
	 * @param numBytes The number of bytes to copy.
	 *
	 * @throws IndexOutOfBoundsException If either of the offsets is invalid, or the source segment does not
	 *           contain the given number of bytes (starting from offset), or the target segment does
	 *           not have enough space for the bytes (counting from targetOffset).
	 */
	public final void copyTo(int offset, MemorySegment target, int targetOffset, int numBytes) {
		final byte[] thisHeapRef = this.heapMemory;
		final byte[] otherHeapRef = target.heapMemory;
		final long thisPointer = this.address + offset;
		final long otherPointer = target.address + targetOffset;

		if ((numBytes | offset | targetOffset) >= 0 &&
				thisPointer <= this.addressLimit - numBytes && otherPointer <= target.addressLimit - numBytes) {
			UNSAFE.copyMemory(thisHeapRef, thisPointer, otherHeapRef, otherPointer, numBytes);
		}
		
		..................................
		一堆异常检查
	}
```

这是一个批量拷贝方法，用于从当前memory segment的offset偏移量开始拷贝numBytes长度的字节到target memory segment中从targetOffset起始的地方。


比较
```java
public final int compare(MemorySegment seg2, int offset1, int offset2, int len) {
		while (len >= 8) {
			long l1 = this.getLongBigEndian(offset1);
			long l2 = seg2.getLongBigEndian(offset2);

			if (l1 != l2) {
				return (l1 < l2) ^ (l1 < 0) ^ (l2 < 0) ? -1 : 1;
			}

			offset1 += 8;
			offset2 += 8;
			len -= 8;
		}
		while (len > 0) {
			int b1 = this.get(offset1) & 0xff;
			int b2 = seg2.get(offset2) & 0xff;
			int cmp = b1 - b2;
			if (cmp != 0) {
				return cmp;
			}
			offset1++;
			offset2++;
			len--;
		}
		return 0;
	}
```

自实现的比较方法，用于对当前memory segment偏移offset1长度为len的数据与seg2偏移起始位offset2长度为len的数据进行比较。

1. 第一个while是逐字节比较，如果len的长度大于8就从各自的起始偏移量开始获取其数据的长整形表示进行对比，如果相等则各自后移8位(一个字节)，并且长度减8，以此循环往复。

getLongBigEndian获取一个长整形,判断是否是大端序，如果是小端序，就进行反转
```java
public final long getLongBigEndian(int index) {
		if (LITTLE_ENDIAN) {
			return Long.reverseBytes(getLong(index));
		} else {
			return getLong(index);
		}
	}
```

0x1234567的大端字节序和小端字节序的写法如下图。

![image](https://note.youdao.com/yws/api/personal/file/E691216E78614EDE94E60DBCCEDAD627?method=download&shareKey=ead114f4fa539399c0eade3135d0ab6e)

2. 第二个循环比较的是最后剩余不到一个字节(八个比特位)，因此是按位比较




#### HybridMemorySegment

它既支持on-heap内存也支持off-heap内存，通过如下实现区分
```java
 unsafe.XXX(Object o, int offset/position, ...)
 ```
这些方法有如下特点：
1. 如果对象o不为null，并且后面的地址或者位置是相对位置，那么会直接对当前对象（比如数组）的相对位置进行操作，既然这里对象不为null，那么这种情况自然满足on-heap的场景；
2. 如果对象o为null，并且后面的地址是某个内存块的绝对地址，那么这些方法的调用也相当于对该内存块进行操作。这里对象o为null，所操作的内存块不是JVM堆内存，这种情况满足了off-heap的场景。

针对堆内内存和堆外内存的构造函数也不一样

堆内内存
```java
	HybridMemorySegment(byte[] buffer, Object owner) {
		super(buffer, owner);
		this.offHeapBuffer = null;
	}
```

堆外内存,使用ByteBuffer，拥有这个实现DirectByteBuffer（直接内存）。
```java

HybridMemorySegment(ByteBuffer buffer, Object owner) {
		super(checkBufferAndGetAddress(buffer), buffer.capacity(), owner);
		this.offHeapBuffer = buffer;
	}
```

获取特定位置的数据
```java
/**
	 * Bulk get method. Copies length memory from the specified position to the
	 * destination memory, beginning at the given offset.
	 *
	 * @param index The position at which the first byte will be read.
	 * @param dst The memory into which the memory will be copied.
	 * @param offset The copying offset in the destination memory.
	 * @param length The number of bytes to be copied.
	 *
	 * @throws IndexOutOfBoundsException Thrown, if the index is negative, or too large that the requested number of
	 *                                   bytes exceed the amount of memory between the index and the memory
	 *                                   segment's end.
	 */
	public abstract void get(int index, byte[] dst, int offset, int length);
```

从第index位置开始读取，获取长度为length的数据，copy到dst中，

```java
public final void get(int index, byte[] dst, int offset, int length) {
		// check the byte array offset and length and the status
		if ((offset | length | (offset + length) | (dst.length - (offset + length))) < 0) {
			throw new IndexOutOfBoundsException();
		}

		final long pos = address + index;
		if (index >= 0 && pos <= addressLimit - length) {
			final long arrayAddress = BYTE_ARRAY_BASE_OFFSET + offset;
			UNSAFE.copyMemory(heapMemory, pos, dst, arrayAddress, length);
		}
		else if (address > addressLimit) {
			throw new IllegalStateException("segment has been freed");
		}
		else {
			// index is in fact invalid
			throw new IndexOutOfBoundsException();
		}
	}
```

unsafe中copyMemory的解释，从scr中srcOffset位置，复制长度length的内容到dest中的destOffset开始。新数据的offset是由BYTE_ARRAY_BASE_OFFSET + offset; 二进制数组的起止索引加上offset，为新数据的offset。
```java
public native void copyMemory(Object srcBase, long srcOffset,
                                  Object destBase, long destOffset,
                                  long bytes);
```


##### 如何获得某个off-heap数据的内存地址

off-heap使用的类是ByteBuffer，继承于Buffer，获取buffer类中的address需要使用反射,因为是一个私有变量
```java
private static final Field ADDRESS_FIELD;

	static {
		try {
			ADDRESS_FIELD = java.nio.Buffer.class.getDeclaredField("address");
			ADDRESS_FIELD.setAccessible(true);
		}
		catch (Throwable t) {
			throw new RuntimeException(
					"Cannot initialize HybridMemorySegment: off-heap memory is incompatible with this JVM.", t);
		}
	}
	
private static long getAddress(ByteBuffer buffer) {
		if (buffer == null) {
			throw new NullPointerException("buffer is null");
		}
		try {
			return (Long) ADDRESS_FIELD.get(buffer);
		}
		catch (Throwable t) {
			throw new RuntimeException("Could not access direct byte buffer address.", t);
		}
	}
```

### MemorySegmentFactory

MemorySegmentFactory是用来创建MemorySegment，而且Flink严重推荐使用它来创建MemorySegment的实例，而不是手动实例化。**为了让运行时只存在某一种MemorySegment的子类实现的实例，而不是MemorySegment的两个子类的实例都同时存在，因为这会让JIT有加载和选择上的开销，导致大幅降低性能**


通过allocateUnpooledOffHeapMemory和allocateUnpooledSegment等多个方法来申请和分配堆内内存还是堆外内存。

从源码上来看，Memory Manager Pool 主要在Batch模式下使用。在Steaming模式下，该池子不会预分配内存，也不会向该池子请求内存块。也就是说该部分的内存都是可以给用户代码使用的。



### MemoryManager

![image](https://note.youdao.com/yws/api/personal/file/F1F06793CD8F408B9CCE6D471CE7AA30?method=download&shareKey=6484343ac7362ab189ddf0e589077d6f)


MemoryManager提供了两个内部类HybridHeapMemoryPool和HybridOffHeapMemoryPool，代表堆内内存池和堆外内存池


为了提升memory segment操作效率，MemoryManager鼓励长度相等的memory segment。由此引入了page的概念。其实page跟memory segment没有本质上的区别，只不过是为了体现memory segment被分配为均等大小的内存空间而引入的。可以将这个类比于操作系统的页式内存分配，page这里看着同等大小的block即可。MemoryManager提供的默认page size为32KB，并提供了自定义page size的下界值不得小于4KB。


```java
	/** The default memory page size. Currently set to 32 KiBytes. */
	public static final int DEFAULT_PAGE_SIZE = 32 * 1024;

	/** The minimal memory page size. Currently set to 4 KiBytes. */
	public static final int MIN_PAGE_SIZE = 4 * 1024;
```


构造函数有两个

```java
public MemoryManager(long memorySize, int numberOfSlots) {
		this(memorySize, numberOfSlots, DEFAULT_PAGE_SIZE, MemoryType.HEAP, true);
	}
```

```java
public MemoryManager(long memorySize, int numberOfSlots, int pageSize,
							MemoryType memoryType, boolean preAllocateMemory)

```
第二个构造器的另一个参数preAllocateMemory，指定memory manager的内存分配策略是预分配还是按需分配。我们后面会看到，对于这两种策略，相关的内存申请和释放操作是不同的。

第二个构造器内就已经根据memory type将特定的memory pool对象初始化好了：

```java
switch (memoryType) {
			case HEAP:
				this.memoryPool = new HybridHeapMemoryPool(memToAllocate, pageSize);
				break;
			case OFF_HEAP:
				if (!preAllocateMemory) {
					LOG.warn("It is advisable to set 'taskmanager.memory.preallocate' to true when" +
						" the memory type 'taskmanager.memory.off-heap' is set to true.");
				}
				this.memoryPool = new HybridOffHeapMemoryPool(memToAllocate, pageSize);
				break;
			default:
				throw new IllegalArgumentException("unrecognized memory type: " + memoryType);
		}

```
通过定位到两个pool对象的构造器，可以看到在实例化构造器的时候就已经将需要预分配的内存分配到位了（当然，这里是针对preAllocateMemory为true的调用情景而言），因为如果该参数为false，那么pool构造器的memToAllocate将会被置为0。

```java
HybridHeapMemoryPool(int numInitialSegments, int segmentSize) {
			this.availableMemory = new ArrayDeque<>(numInitialSegments);
			this.segmentSize = segmentSize;

			for (int i = 0; i < numInitialSegments; i++) {
				this.availableMemory.add(new byte[segmentSize]);
			}
		}
```

```java
HybridOffHeapMemoryPool(int numInitialSegments, int segmentSize) {
			this.availableMemory = new ArrayDeque<>(numInitialSegments);
			this.segmentSize = segmentSize;

			for (int i = 0; i < numInitialSegments; i++) {
				this.availableMemory.add(ByteBuffer.allocateDirect(segmentSize));
			}
		}

```

两种模式的差别在于堆内内存是直接new byte，堆外内存ByteBuffer.allocateDirect。

allocatePages以及release方法为分配和释放内存

```java
/**
	 * Allocates a set of memory segments from this memory manager. If the memory manager pre-allocated the
	 * segments, they will be taken from the pool of memory segments. Otherwise, they will be allocated
	 * as part of this call.
	 *
	 * @param owner The owner to associate with the memory segment, for the fallback release.
	 * @param target The list into which to put the allocated memory pages.
	 * @param numPages The number of pages to allocate.
	 * @throws MemoryAllocationException Thrown, if this memory manager does not have the requested amount
	 *                                   of memory pages any more.
	 */
	public void allocatePages(Object owner, List<MemorySegment> target, int numPages)
			throws MemoryAllocationException {
		// sanity check
		if (owner == null) {
			throw new IllegalArgumentException("The memory owner must not be null.");
		}

		// reserve array space, if applicable
		if (target instanceof ArrayList) {
			((ArrayList<MemorySegment>) target).ensureCapacity(numPages);
		}

		// -------------------- BEGIN CRITICAL SECTION -------------------
		synchronized (lock) {
			if (isShutDown) {
				throw new IllegalStateException("Memory manager has been shut down.");
			}

			// in the case of pre-allocated memory, the 'numNonAllocatedPages' is zero, in the
			// lazy case, the 'freeSegments.size()' is zero.
			if (numPages > (memoryPool.getNumberOfAvailableMemorySegments() + numNonAllocatedPages)) {
				throw new MemoryAllocationException("Could not allocate " + numPages + " pages. Only " +
						(memoryPool.getNumberOfAvailableMemorySegments() + numNonAllocatedPages)
						+ " pages are remaining.");
			}

			Set<MemorySegment> segmentsForOwner = allocatedSegments.get(owner);
			if (segmentsForOwner == null) {
				segmentsForOwner = new HashSet<MemorySegment>(numPages);
				allocatedSegments.put(owner, segmentsForOwner);
			}

			if (isPreAllocated) {
				for (int i = numPages; i > 0; i--) {
					MemorySegment segment = memoryPool.requestSegmentFromPool(owner);
					target.add(segment);
					segmentsForOwner.add(segment);
				}
			}
			else {
				for (int i = numPages; i > 0; i--) {
					MemorySegment segment = memoryPool.allocateNewSegment(owner);
					target.add(segment);
					segmentsForOwner.add(segment);
				}
				numNonAllocatedPages -= numPages;
			}
		}
		// -------------------- END CRITICAL SECTION -------------------
	}
```
这两个方法都共同拥有一个参数owner，一个映射关系，谁申请的memory segment，将会挂到谁的名下，释放的时候也从谁的名下删除.

allocatePages中pagenumber代表了要申请多少个segment，如果是预分配模式，调用requestSegmentFromPool方法，如果不是用的是allocateNewSegment方法，差别在于requestSegmentFromPool是从pool中的双端队列ArrayDeque中获取预先分配的，否则直接new出来.

memory segment释放

```java
public void release(MemorySegment segment) {
		// check if segment is null or has already been freed
		if (segment == null || segment.getOwner() == null) {
			return;
		}

		final Object owner = segment.getOwner();

		// -------------------- BEGIN CRITICAL SECTION -------------------
		synchronized (lock) {
			// prevent double return to this memory manager
			if (segment.isFreed()) {
				return;
			}
			if (isShutDown) {
				throw new IllegalStateException("Memory manager has been shut down.");
			}

			// remove the reference in the map for the owner
			try {
				Set<MemorySegment> segsForOwner = this.allocatedSegments.get(owner);

				if (segsForOwner != null) {
					segsForOwner.remove(segment);
					if (segsForOwner.isEmpty()) {
						this.allocatedSegments.remove(owner);
					}
				}

				if (isPreAllocated) {
					// release the memory in any case
					memoryPool.returnSegmentToPool(segment);
				}
				else {
					segment.free();
					numNonAllocatedPages++;
				}
			}
			catch (Throwable t) {
				throw new RuntimeException("Error removing book-keeping reference to allocated memory segment.", t);
			}
		}
		// -------------------- END CRITICAL SECTION -------------------
	}
```
基本和allocate是相反的逻辑，如果当前释放的segment是segsForOwner集合中的最后一个，那么将segsForOwner也从allocatedSegments中移除。

#### DataInput 数据视图

提供了基于page的对view的进一步实现，说得更直白一点就是，它提供了跨越多个memory page的数据访问(input/output)视图。它包含了从page中读取/写入数据的解码/编码方法以及跨越page的边界检查（边界检查主要由实现类来完成）。

![image](https://note.youdao.com/yws/api/personal/file/6096E819C1E84DE3985771AEBD0DC4FE?method=download&shareKey=cae73308266945a7fb6b1847cefc84d9)



AbstractPagedInputView中advance获取下一个memory segment

```java
/**
	 * Advances the view to the next memory segment. The reading will continue after the header of the next
	 * segment. This method uses {@link #nextSegment(MemorySegment)} and {@link #getLimitForSegment(MemorySegment)}
	 * to get the next segment and set its limit.
	 *
	 * @throws IOException Thrown, if the next segment could not be obtained.
	 *
	 * @see #nextSegment(MemorySegment)
	 * @see #getLimitForSegment(MemorySegment)
	 */
	protected final void advance() throws IOException {
		// note: this code ensures that in case of EOF, we stay at the same position such that
		// EOF is reproducible (if nextSegment throws a reproducible EOFException)
		this.currentSegment = nextSegment(this.currentSegment);
		this.limitInSegment = getLimitForSegment(this.currentSegment);
		this.positionInSegment = this.headerLength;
	}
```
nextSegment、getLimitForSegment都是由具体子类自行实现。


读取长度为len的内容，将内容填充到byte[]里头，从offset的位置开始。

如果读取的长度比当前segment的可读长度（int remaining = this.limitInSegment - this.positionInSegment;）小，那么直接读取。
如果要读取的长度比当前segment长，那么会出现读取下一个page的操作。
```java

/**
	 * Reads up to {@code len} bytes of memory and stores it into {@code b} starting at offset {@code off}.
	 * It returns the number of read bytes or -1 if there is no more data left.
	 *
	 * @param b byte array to store the data to
	 * @param off offset into byte array
	 * @param len byte length to read
	 * @return the number of actually read bytes of -1 if there is no more data left
	 * @throws IOException
	 */
@Override
	public int read(byte[] b, int off, int len) throws IOException{
		if (off < 0 || len < 0 || off + len > b.length) {
			throw new IndexOutOfBoundsException();
		}

		int remaining = this.limitInSegment - this.positionInSegment;
		if (remaining >= len) {
			this.currentSegment.get(this.positionInSegment, b, off, len);
			this.positionInSegment += len;
			return len;
		}
		else {
			if (remaining == 0) {
				try {
					advance();
				}
				catch (EOFException eof) {
					return -1;
				}
				remaining = this.limitInSegment - this.positionInSegment;
			}

			int bytesRead = 0;
			while (true) {
				int toRead = Math.min(remaining, len - bytesRead);
				this.currentSegment.get(this.positionInSegment, b, off, toRead);
				off += toRead;
				bytesRead += toRead;

				if (len > bytesRead) {
					try {
						advance();
					}
					catch (EOFException eof) {
						this.positionInSegment += toRead;
						return bytesRead;
					}
					remaining = this.limitInSegment - this.positionInSegment;
				}
				else {
					this.positionInSegment += toRead;
					break;
				}
			}
			return len;
		}
	}
```


AbstractPagedOutputView写和读的方法其实差不多，在当前页就直接写，跨页就遍历每一个页，写

```java
@Override
	public void write(byte[] b, int off, int len) throws IOException {
		int remaining = this.segmentSize - this.positionInSegment;
		if (remaining >= len) {
			this.currentSegment.put(this.positionInSegment, b, off, len);
			this.positionInSegment += len;
		}
		else {
			if (remaining == 0) {
				advance();
				remaining = this.segmentSize - this.positionInSegment;
			}
			while (true) {
				int toPut = Math.min(remaining, len);
				this.currentSegment.put(this.positionInSegment, b, off, toPut);
				off += toPut;
				len -= toPut;

				if (len > 0) {
					this.positionInSegment = this.segmentSize;
					advance();
					remaining = this.segmentSize - this.positionInSegment;
				}
				else {
					this.positionInSegment += toPut;
					break;
				}
			}
		}
	}
```

### Reference 

https://blog.csdn.net/yanghua_kobe/article/details/50976124

https://blog.csdn.net/yanghua_kobe/article/details/51079524

http://blog.jrwang.me/2019/flink-source-code-memory-management/


https://www.baidu.com/link?url=HSshYdkRO5A-6ttSe9xPlNpkp8Wa-PdGjqarh5s7KG8xnWmyYJfn_QIIYAwXjQKN&wd=&eqid=85713bec000167b3000000065d7218e2

https://www.jianshu.com/p/644d430aaa39

http://wuchong.me/blog/2016/04/29/flink-internals-memory-manage/