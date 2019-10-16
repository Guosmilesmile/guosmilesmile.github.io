---
title: Flink源码解析  序列化
date: 2019-09-17 22:23:50
tags:
categories:
	- Flink
---

TypeInformation 类是描述一切类型的公共基类，它和它的所有子类必须可序列化（Serializable），因为类型信息将会伴随 Flink 的作业提交，被传递给每个执行节点。

类型信息由 TypeInformation 类表示，TypeInformation 支持以下几种类型：

* BasicTypeInfo: 任意Java 基本类型（装箱的）或 String 类型。
* BasicArrayTypeInfo: 任意Java基本类型数组（装箱的）或 String 数组。
* WritableTypeInfo: 任意 Hadoop Writable 接口的实现类。
* TupleTypeInfo: 任意的 Flink Tuple 类型(支持Tuple1 to Tuple25)。Flink tuples 是固定长度固定类型的Java Tuple实现。
* CaseClassTypeInfo: 任意的 Scala CaseClass(包括 Scala tuples)。
* PojoTypeInfo: 任意的 POJO (Java or Scala)，例如，Java对象的所有成员变量，要么是 public 修饰符定义，要么有 getter/setter 方法。
* GenericTypeInfo: 任意无法匹配之前几种类型的类。

![image](https://note.youdao.com/yws/api/personal/file/9B6167B879D44D0897292590B37E9D61?method=download&shareKey=475b5c6e1e1984afcbc727f42d246e3c)

### TypSerializer

所有序列化的基础类。此接口描述Flink运行时处理数据类型所需的方法。 具体来说，此接口包含序列化和复制方法。

其中有一个深度拷贝的方法duplicate，因为这个序列化器是非线程安全的，如果序列化器不具备状态，直接返回自身，如果有状态，调用这个方法返回一个深度拷贝（抽象发发，具体实现子类自行实现。）

```java
/**
	 * Creates a deep copy of this serializer if it is necessary, i.e. if it is stateful. This
	 * can return itself if the serializer is not stateful.
	 *
	 * We need this because Serializers might be used in several threads. Stateless serializers
	 * are inherently thread-safe while stateful serializers might not be thread-safe.
	 */
	public abstract TypeSerializer<T> duplicate();
```

还有序列化和逆序列化的定义，将数据与DataView进行转换，dataView作为MemorySegment 的抽象显示，序列化与逆序列化就是讲自定义的内存结构与实体进行转换的过程。

```java
/**
	 * Serializes the given record to the given target output view.
	 * 
	 * @param record The record to serialize.
	 * @param target The output view to write the serialized data to.
	 * 
	 * @throws IOException Thrown, if the serialization encountered an I/O related error. Typically raised by the
	 *                     output view, which may have an underlying I/O channel to which it delegates.
	 */
	public abstract void serialize(T record, DataOutputView target) throws IOException;

	/**
	 * De-serializes a record from the given source input view.
	 * 
	 * @param source The input view from which to read the data.
	 * @return The deserialized element.
	 * 
	 * @throws IOException Thrown, if the de-serialization encountered an I/O related error. Typically raised by the
	 *                     input view, which may have an underlying I/O channel from which it reads.
	 */
	public abstract T deserialize(DataInputView source) throws IOException;
```

选几个具体的序列化器分析一下

分为定长(LongSerializer)和变长(StringSerializer)
##### LongSerializer

```java
@Override
	public void serialize(Long record, DataOutputView target) throws IOException {
		target.writeLong(record);
	}

	@Override
	public Long deserialize(DataInputView source) throws IOException {
		return source.readLong();
	}
```
直接调用DataView的方法

AbstractPagedOutputView.writeLong

```java
public void writeLong(long v) throws IOException {
		if (this.positionInSegment < this.segmentSize - 7) {
			this.currentSegment.putLongBigEndian(this.positionInSegment, v);
			this.positionInSegment += 8;
		}
		else if (this.positionInSegment == this.segmentSize) {
			advance();
			writeLong(v);
		}
		else {
			writeByte((int) (v >> 56));
			writeByte((int) (v >> 48));
			writeByte((int) (v >> 40));
			writeByte((int) (v >> 32));
			writeByte((int) (v >> 24));
			writeByte((int) (v >> 16));
			writeByte((int) (v >>  8));
			writeByte((int) v);
		}
	}
```

通过this.positionInSegment < this.segmentSize - 7判断剩余的长度，如果剩余的长度还够8位，那么直接put进去

```java
this.currentSegment.putLongBigEndian(this.positionInSegment, v);
```
这里会判断是大端序还是小端序，如果是小端，需要做一次转换
```java
public final void putLongBigEndian(int index, long value) {
		if (LITTLE_ENDIAN) {
			putLong(index, Long.reverseBytes(value));
		} else {
			putLong(index, value);
		}
	}
```
如果剩余的不足8位，且刚好剩余为0，通过advance进行切换页操作，然后写。如果剩余的还有一些，因为long是8个byte，所以每次写一个byte进去，不够再切页。切页操作是由具体的子类实现的。


##### StringSerializer 

重点在于序列化和逆序列化
```java
@Override
	public void serialize(String record, DataOutputView target) throws IOException {
		StringValue.writeString(record, target);
	}

	@Override
	public String deserialize(DataInputView source) throws IOException {
		return StringValue.readString(source);
	}
```
重点的放在StringValue中。

##### StringValue

变长的string，是比较复杂的。
* 以一个byte(8位)为一格，小于128的都好办，大于128的需要循环写入
* **大于128的字符读取结束的条件是遇到小于128的byte，所以变长字符的写入要大于128，通过每次写入7位，第八位强制设为1实现。**


在这个类中，定义了一个高bit  0x1 << 7(128，ASCII 定义了128个字符)

```java
	private static final int HIGH_BIT = 0x1 << 7;
```

先研究写入，写入的时候，是通过out.write(byte)写入数据，一次最多只能写入8位数据。flink的做法是每次获取7位的数据，然后第八位通过（ | HIGH_BIT[1000 0000]） 强制置为1，在二进制中，置为第八位代表正负，1位负，0位正。然后右移7位，获取下一个高7位的数据，以此类推，写入到流中。

假设写入一个汉字 ：测

测 这个字 对应的 (int)s.charAt(0)为27979，
写入的时候，先获取长度+1，第一位为2，将27979与HIGH_BIT进行或操作将第八位置为1，得到28107，其实真实写入的是28107的后八位11001011，右移7位后进行计算得到218，再一次得到1，序列化后的是  2 28107 218 1 . 写入byte中得到2 -53 -38 1. 

因为有符号范围在 -128  ---- 127之间，无符号是在0-256，由于写是默认按照有符号写，所以写11001011会变成-53.


```java
public static final void writeString(CharSequence cs, DataOutput out) throws IOException {
		if (cs != null) {
			// the length we write is offset by one, because a length of zero indicates a null value
			int lenToWrite = cs.length()+1;
			if (lenToWrite < 0) {
				throw new IllegalArgumentException("CharSequence is too long.");
			}
	
			// write the length, variable-length encoded
			while (lenToWrite >= HIGH_BIT) {
				out.write(lenToWrite | HIGH_BIT);
				lenToWrite >>>= 7;
			}
			out.write(lenToWrite);
	
			// write the char data, variable length encoded
			for (int i = 0; i < cs.length(); i++) {
				int c = cs.charAt(i);
	
				while (c >= HIGH_BIT) {
					out.write(c | HIGH_BIT);
					c >>>= 7;
				}
				out.write(c);
			}
		} else {
			out.write(0);
		}
	}
```

逆序列化的时候，逆序列化 2 -53 -38 1. 

```java
        if (c < HIGH_BIT) {
				data[i] = (char) c;
			}
```
默认如果小于128，就会当做是结束条件，直接转换成char，所以在变长的string中，还没结束的部分，一定要大于128，这就是为什么写入的时候，一定要强制将第八位置为1.读取的2-1=1获取数据长度为1， -53 通过in.readUnsignedByte()得到203,对应11001011，取七位1001011，作为低七位，
继续获取218[11011010]，取七位[1011010]，再进行左移7位操作作为高一阶的七位，拼上低七位，得到10110101001011,最后一个1，进行左移14位操作作为再高一阶的七位，拼上低十四位得到110110101001011对应的十进制为27979，转为ascII为测这个字。


```java
public static String readString(DataInput in) throws IOException {
		// the length we read is offset by one, because a length of zero indicates a null value
		int len = in.readUnsignedByte();
		
		if (len == 0) {
			return null;
		}

		if (len >= HIGH_BIT) {
			int shift = 7;
			int curr;
			len = len & 0x7f;
			while ((curr = in.readUnsignedByte()) >= HIGH_BIT) {
				len |= (curr & 0x7f) << shift;
				shift += 7;
			}
			len |= curr << shift;
		}
		
		// subtract one for the null length
		len -= 1;
		
		final char[] data = new char[len];

		for (int i = 0; i < len; i++) {
			int c = in.readUnsignedByte();
			if (c < HIGH_BIT) {
				data[i] = (char) c;
			} else {
				int shift = 7;
				int curr;
				c = c & 0x7f;
				while ((curr = in.readUnsignedByte()) >= HIGH_BIT) {
					c |= (curr & 0x7f) << shift;
					shift += 7;
				}
				c |= curr << shift;
				data[i] = (char) c;
			}
		}
		
		return new String(data, 0, len);
	}
```

##### PojoSerializer

POJO多个一个字节的header，PojoSerializer只负责将header序列化进去，并委托每个字段对应的serializer对字段进行序列化。

```java
	// Flags for the header
	private static byte IS_NULL = 1;
	private static byte NO_SUBCLASS = 2;
	private static byte IS_SUBCLASS = 4;
	private static byte IS_TAGGED_SUBCLASS = 8;
```

序列化的时候，会先判断是否是null，是的话，置为IS_NULL.

接着进行header的判断。
如果Class<?> actualClass = value.getClass();等于构造函数的class，说明这个class不是一个subClass，置为IS_SUBCLASS。如果在registeredClasses可以获取到子类，说明这个类自身存在子类，他是一个父类，置为IS_TAGGED_SUBCLASS
先将flag写入，target.writeByte(flags);
如果是subclass，将类的全名写入序列化中，如果



如果是NO_SUBCLASS，直接

```
header{
    flag,
    subFlag,(class tag id  or the full classname)
}
```
写完header，再委托每个字段对应的serializer对字段进行序列化。
```java
public void serialize(T value, DataOutputView target) throws IOException {
		int flags = 0;
		// handle null values
		if (value == null) {
			flags |= IS_NULL;
			target.writeByte(flags);
			return;
		}

		Integer subclassTag = -1;
		Class<?> actualClass = value.getClass();
		TypeSerializer subclassSerializer = null;
		if (clazz != actualClass) {
			subclassTag = registeredClasses.get(actualClass);
			if (subclassTag != null) {
				flags |= IS_TAGGED_SUBCLASS;
				subclassSerializer = registeredSerializers[subclassTag];
			} else {
				flags |= IS_SUBCLASS;
				subclassSerializer = getSubclassSerializer(actualClass);
			}
		} else {
			flags |= NO_SUBCLASS;
		}

		target.writeByte(flags);

		// if its a registered subclass, write the class tag id, otherwise write the full classname
		if ((flags & IS_SUBCLASS) != 0) {
			target.writeUTF(actualClass.getName());
		} else if ((flags & IS_TAGGED_SUBCLASS) != 0) {
			target.writeByte(subclassTag);
		}

		// if its a subclass, use the corresponding subclass serializer,
		// otherwise serialize each field with our field serializers
		if ((flags & NO_SUBCLASS) != 0) {
			try {
				for (int i = 0; i < numFields; i++) {
					Object o = (fields[i] != null) ? fields[i].get(value) : null;
					if (o == null) {
						target.writeBoolean(true); // null field handling
					} else {
						target.writeBoolean(false);
						fieldSerializers[i].serialize(o, target);
					}
				}
			} catch (IllegalAccessException e) {
				throw new RuntimeException("Error during POJO copy, this should not happen since we check the fields before.", e);
			}
		} else {
			// subclass
			if (subclassSerializer != null) {
				subclassSerializer.serialize(value, target);
			}
		}
	}
```


![image](https://note.youdao.com/yws/api/personal/file/4FE22225B71B474A834F13F69B7C789E?method=download&shareKey=54f98b410bf649be112f69db79b4afc7)




## Flink 如何直接操作二进制数据

### 排序

Flink 提供了如 group、sort、join 等操作，这些操作都需要访问海量数据。这里，我们以sort为例，这是一个在 Flink 中使用非常频繁的操作。

首先，Flink 会从 MemoryManager 中申请一批 MemorySegment，我们把这批 MemorySegment 称作 sort buffer，用来存放排序的数据。

将实际的数据和指针加定长key分开存放有两个目的。第一，交换定长块（key+pointer）更高效，不用交换真实的数据也不用移动其他key和pointer。第二，这样做是缓存友好的，因为key都是连续存储在内存中的，可以大大减少 cache miss（后面会详细解释）。

排序的关键是比大小和交换。Flink 中，会先用 key 比大小，这样就可以直接用二进制的key比较而不需要反序列化出整个对象。因为key是定长的，所以如果key相同（或者没有提供二进制key），那就必须将真实的二进制数据反序列化出来，然后再做比较。之后，只需要交换key+pointer就可以达到排序的效果，真实的数据不用移动。

最后，访问排序后的数据，可以沿着排好序的key+pointer区域顺序访问，通过pointer找到对应的真实数据，并写到内存或外部

![image](https://note.youdao.com/yws/api/personal/file/11B1637FC25C4C3686FAD944DCA964FF?method=download&shareKey=d3c087ab46b1888a2002fc238598ef44)

![image](https://note.youdao.com/yws/api/personal/file/86CB5AB7CCAE4DB8AB9F185D0860CA82?method=download&shareKey=a3527963a8da3fadb9da6cab519a985a)

可见NormalizedKeySorter.java




### Reference
https://segmentfault.com/a/1190000016350098

http://wuchong.me/blog/2016/04/29/flink-internals-memory-manage/

https://www.cnblogs.com/fxjwind/p/5334271.html

http://aitozi.com/BinaryRow-implement.html