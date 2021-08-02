---
title: Kafka压缩的思考和性能对比
date: 2021-07-20 00:35:26
categories:
	- Kafka
---


## 背景

kafka的压缩可以提升性能，可是kafka的链路有producer、server、consumer这三个环节，那么是哪里做的呢？
压缩格式有GZIP、Snappy、LZ4、ZStandard性能上又有什么差别呢？


## 总结


kafka的压缩一般是发生在客户端，可以发生在服务端，因为两个都可以压缩，会出现压缩冲突。如果是正常的客户端压缩，那么消息在客户端压缩，服务端是不会做解压的，对server没有损耗，还可以减少带宽

目前的压缩性能对比

压缩比：LZ4 > GZIP > Snappy
吞吐量：LZ4 > Snappy > GZIP


## 压缩是在哪发生的

在Kafka中，压缩可能发生在两个地方：生产者端和Broker端


### 生产者压缩
生产者程序中配置compression.type参数即表示启用指定类型的压缩算法

```java
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("acks", "all");
props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
// 开启GZIP压缩
props.put("compression.type", "gzip");
Producer<String, String> producer = new KafkaProducer<>(props);
```

这里比较关键的代码行是props.put(“compression.type”, “gzip”)，它表明该Producer的压缩算法使用的是GZIP

这样Producer启动后生产的每个消息集合都是经GZIP压缩过的，故而能很好地节省网络传输带宽以及Kafka Broker端的磁盘占用。

既然kafka在client压缩，那么comsumer也有对应的解压才是，不然解压可能出现在kafka server。

### kafka producer 压缩源码

详细源码可以看RecordAccumulator.java和CompressionType.java和MemoryRecordsBuilder.java


MemoryRecordsBuilder.java
```java
public MemoryRecordsBuilder(ByteBufferOutputStream bufferStream,
                                byte magic,
                                CompressionType compressionType,
                                TimestampType timestampType,
                                long baseOffset,
                                long logAppendTime,
                                long producerId,
                                short producerEpoch,
                                int baseSequence,
                                boolean isTransactional,
                                boolean isControlBatch,
                                int partitionLeaderEpoch,
                                int writeLimit) {
        .....

        bufferStream.position(initialPosition + batchHeaderSizeInBytes);
        this.bufferStream = bufferStream;
        this.appendStream = new DataOutputStream(compressionType.wrapForOutput(this.bufferStream, magic));
    }

```
compressionType内部是由wrapForOutput,wrapForInput这两个方法组成
```java
LZ4(3, "lz4", 1.0f) {
        @Override
        public OutputStream wrapForOutput(ByteBufferOutputStream buffer, byte messageVersion) {
            try {
                return new KafkaLZ4BlockOutputStream(buffer, messageVersion == RecordBatch.MAGIC_VALUE_V0);
            } catch (Throwable e) {
                throw new KafkaException(e);
            }
        }

        @Override
        public InputStream wrapForInput(ByteBuffer inputBuffer, byte messageVersion, BufferSupplier decompressionBufferSupplier) {
            try {
                return new KafkaLZ4BlockInputStream(inputBuffer, decompressionBufferSupplier,
                                                    messageVersion == RecordBatch.MAGIC_VALUE_V0);
            } catch (Throwable e) {
                throw new KafkaException(e);
            }
        }
```
### consumer解压源码
AbstractLegacyRecordBatch.java
```java

 private CloseableIterator<Record> iterator(BufferSupplier bufferSupplier) {
        if (isCompressed())
            return new DeepRecordsIterator(this, false, Integer.MAX_VALUE, bufferSupplier);
            
        .....            
            
            
    }


private DeepRecordsIterator(AbstractLegacyRecordBatch wrapperEntry,
                                    boolean ensureMatchingMagic,
                                    int maxMessageSize,
                                    BufferSupplier bufferSupplier) {
            LegacyRecord wrapperRecord = wrapperEntry.outerRecord();
            this.wrapperMagic = wrapperRecord.magic();
            if (wrapperMagic != RecordBatch.MAGIC_VALUE_V0 && wrapperMagic != RecordBatch.MAGIC_VALUE_V1)
                throw new InvalidRecordException("Invalid wrapper magic found in legacy deep record iterator " + wrapperMagic);

            CompressionType compressionType = wrapperRecord.compressionType();
            ByteBuffer wrapperValue = wrapperRecord.value();
            if (wrapperValue == null)
                throw new InvalidRecordException("Found invalid compressed record set with null value (magic = " +
                        wrapperMagic + ")");

            InputStream stream = compressionType.wrapForInput(wrapperValue, wrapperRecord.magic(), bufferSupplier);
            LogInputStream<AbstractLegacyRecordBatch> logStream = new DataLogInputStream(stream, maxMessageSize);
            
            .......
            }


```


### 服务端压缩

服务端配置compression.type


## 压缩冲突

大部分情况下，Broker 从 Producer 接收到消息后，仅仅只是原封不动地保存，而不会对其进行任何修改。什么情况下会出现重新压缩？


### Broker端指定了和Producer端不同的压缩算法

Producer端指定了压缩算法为GZIP，Broker端指定了压缩算法为Snappy，在这种情况下Broker接收到GZIP压缩的消息后，只能先解压缩然后使用Snappy重新压缩一遍

可一旦在Broker端设置了不同的compression.type值，就要小心了，因为可能会发生预料之外的压缩/解压缩操作，通常表现为Broker端CPU使用率飙升


### Broker端发生了消息格式转换

消息格式转换主要是为了兼容老版本的消费者程序，在一个 Kafka 集群中通常同时保存多种版本的消息格式（V1/V2）。    
Broker 端会对新版本消息执行向老版本格式的转换，该过程中会涉及消息的解压缩和重新压缩。   
消息格式转换对性能的影响很大，除了增加额外的压缩和解压缩操作之外，还会让 Kafka 丧失其优秀的 Zero Copy特性。因此，一定要保证消息格式的统一。    
Zero Copy：数据在磁盘和网络进行传输时，避免昂贵的内核态数据拷贝，从而实现快速的数据传输。    




## 压缩性能比较呢

下面这张表是Facebook Zstandard官网提供的一份压缩算法基准测试比较结果


![image](https://note.youdao.com/yws/api/personal/file/CEF9B00C83E348DCB3351219EB7AC980?method=download&shareKey=c89b8a76cf002f96f5299988d9899e36)


还有一个是kafka 开发大佬自己压测的结果
https://www.cnblogs.com/huxi2b/p/10330607.html

从情况来看如下：

压缩比：LZ4 > GZIP > Snappy
吞吐量：LZ4 > Snappy > GZIP




### Reference

https://www.cnblogs.com/huxi2b/p/10330607.html
http://www.louisvv.com/archives/2436.html
https://intl.cloud.tencent.com/zh/document/product/597/34004?lang=zh&pg=
https://www.jianshu.com/p/22e0d862149f
