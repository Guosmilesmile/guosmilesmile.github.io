---
title: Flink Kafka Sink 埋坑历程
date: 2021-08-07 12:32:57
tags:
categories:
	- Flink
---


## 背景

由于历史原因，flink版本停留在1.7版本，kafka sink使用的是FlinkKafkaProducer011. 该版本在flink 1.10后就不在维护，使用的是通用的kafka connection包

## 先上结论

1. kafka produce 在低版本，会指定partition为fix
2. 修改fix的方法为传递null，011有bug，不能直接传null
3. kafka produce的partition * batch.size < buffer memory，不然会有性能问题
4. kafka produce 在0.10版本，针对snapp有硬编码，增大batch.size会导致吞吐上不去

## 问题一

Flink sink数据到kafka中,程序并行度是100，下游kafka的topic partition为200，现象是只有100个partition有数据。



### 问题分析

下游的partition与上游并行度的绑定，会导致kafka失去partition提高并行度的优势，下游和上游绑定会有很大的问题


### 源码分析


FlinkKafkaProducer011 使用的是默认的构造函数

```java

public FlinkKafkaProducer011(
            String brokerList, 
            String topicId, 
            SerializationSchema<IN> serializationSchema);

```

默认构造函数在底层调用了如下

```java
public FlinkKafkaProducer011(
			String topicId,
			KeyedSerializationSchema<IN> serializationSchema,
			Properties producerConfig,
			Semantic semantic) {
		this(topicId,
			serializationSchema,
			producerConfig,
			Optional.of(new FlinkFixedPartitioner<IN>()),
			semantic,
			DEFAULT_KAFKA_PRODUCERS_POOL_SIZE);
	}
```
FlinkFixedPartitioner是何许东西呢。

```java
        if (flinkKafkaPartitioner != null) {
			record = new ProducerRecord<>(
				targetTopic,
				flinkKafkaPartitioner.partition(next, serializedKey, serializedValue, targetTopic, partitions),
				timestamp,
				serializedKey,
				serializedValue);
		} else {
			record = new ProducerRecord<>(targetTopic, null, timestamp, serializedKey, serializedValue);
		}
```
如果有设置flinkKafkaPartitioner，那么发送数据的时候就会设定为partition，如果设置为null就可以发送到全部partition。


### 初步解决方案
调用构造函数，在partitioner的入参设置为null


```java
public FlinkKafkaProducer011(
			String topicId,
			SerializationSchema<IN> serializationSchema,
			Properties producerConfig,
			Optional<FlinkKafkaPartitioner<IN>> customPartitioner) {

		this(topicId, new KeyedSerializationSchemaWrapper<>(serializationSchema), producerConfig, customPartitioner);
	}
```

### 初步方案带来的bug

在FlinkKafkaProducer010 FlinkKafkaProducer09 都是正常的，在FlinkKafkaProducer011直接抛出异常了...


再看下最底层的构造函数

011

```java
this.flinkKafkaPartitioner = checkNotNull(customPartitioner, "customPartitioner is null").orElse(null);

public static <T> T checkNotNull(T reference, @Nullable String errorMessage) {
		if (reference == null) {
			throw new NullPointerException(String.valueOf(errorMessage));
		}
		return reference;
	}
	
	
```

如此传入的null，就会抛异常，那还orElse(null)想干嘛。。。看来是个bug


在看下010,	@Nullable...

```java
public FlinkKafkaProducer010(
			String topicId,
			KeyedSerializationSchema<T> serializationSchema,
			Properties producerConfig,
			@Nullable FlinkKafkaPartitioner<T> customPartitioner) {

		super(topicId, serializationSchema, producerConfig, customPartitioner);
	}
```



### 最后解决方案

使用最全的构造函数，就可以跳过这个bug
```java
new FlinkKafkaProducer011<String>(
sinkTopic, new StringKeyedSerializationSchema,producerConfig,Optional.ofNullable(null), sinkSemantic,5)

```


## 问题二


上了问题一的解决方案，数据可以shuffer到所有的分区了，可是吞吐上不去了从原来的 4并发 90k/s降低到6k/s

![image](https://note.youdao.com/yws/api/personal/file/E41109B019684731BFC1FE42CB62A7E1?method=download&shareKey=925d6a4e418a01ec97d771b52abc701b)


### 问题猜测

配置如下

batch.size=512k   
linger.ms=200ms    
下游partition数量100.


改动最大的变化是client原来是一个sink对 1-2个partition，到现在是1对100个partition，每个partition都需要一个batch.size。 512k*100=51m>32m了，
是不是buffer memory没设置，32m不够用了？


### 尝试

```
properties.setProperty(ProducerConfig.BUFFER_MEMORY_CONFIG, String.valueOf(100 * 1024 * 1024))
```

增加配置。吞吐上去了

### 尝试2 

batch.size改为51k

吞吐也上去了


### 结论

batch.szie * partition  < buffer memory

## 效果图

![image](https://note.youdao.com/yws/api/personal/file/5133A85514A44308A5FF53F9774D6B62?method=download&shareKey=5de48de3b3feb707e26f138efeda094a)

![image](https://note.youdao.com/yws/api/personal/file/CCAD30289860473EB8223CED300166DC?method=download&shareKey=759c3dcc0900a5347c5445eaf7bef263)

## 外传


在使用0.10发送数据到kafka中，压缩使用snapp，增大batch size理论会让压缩率变高，性能更好，结果相反，性能更差了。

从官方的 0.11的RELEASE NOTES可以看到这么一段话

>> When compressing data with snappy, the producer and broker will use the compression scheme's default block size (2 x 32 KB) instead of 1 KB in order to improve the compression ratio. There have been reports of data compressed with the smaller block size being 50% larger than when compressed with the larger block size. For the snappy case, a producer with 5000 partitions will require an additional 315 MB of JVM heap.

https://kafka.apache.org/0110/documentation.html


可以看出0.10把数据1k压缩一次，32k的数据这还怎么玩。。

后续把kafka client的版本提升上去就吞吐上去了，符合三观


