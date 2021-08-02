---
title: FlinkKafkaSink、两阶段提交协议和Semantic三种类型源码解析
date: 2021-06-09 22:26:22
tags:
categories:
	- Flink
---



源码基于1.12.4


### 初始化

通常添加一个 kafka sink 的代码如下：

```java

input.addSink(
   new FlinkKafkaProducer<>(
      "testTopic",
      new KafkaSerializationSchemaImpl(),
         properties,
      FlinkKafkaProducer.Semantic.AT_LEAST_ONCE)).name("Example Sink");
      

public class KafkaSerializationSchemaImpl implements KafkaSerializationSchema<String> {
    private String topic;
    private static final Charset CHARSET = StandardCharsets.UTF_8;

    public KafkaSerializationSchemaImpl(String topic) {
        this.topic = topic;
    }

    @Override
    public void open(SerializationSchema.InitializationContext context) throws Exception {

    }

    @Override
    public ProducerRecord<byte[], byte[]> serialize(String element, @Nullable Long timestamp) {
        byte[] bytes = element.getBytes(CHARSET);
        return new ProducerRecord<>(topic,bytes);
    }
}

```

初始化执行 env.addSink 的时候会创建 StreamSink 对象，即 StreamSink sinkOperator = new StreamSink<>(clean(sinkFunction));这里的 sinkFunction 就是传入的 FlinkKafkaProducer 对象，StreamSink 构造函数中将这个对象传给父类 AbstractUdfStreamOperator 的 userFunction 变量


### Task运行

StreamSink 会调用下面的方法发送数据

```java
@Override
public void processElement(StreamRecord<IN> element) throws Exception {
   sinkContext.element = element;
   userFunction.invoke(element.getValue(), sinkContext);
}
```

也就是实际调用的是 FlinkKafkaProducer#invoke 方法。在 FlinkKafkaProducer 的构造函数中需要指 FlinkKafkaProducer.Semantic.

```java
public enum Semantic {
EXACTLY_ONCE,
AT_LEAST_ONCE,
NONE
}
```

####  Semantic.NONE

这种方式不会做任何额外的操作，完全依靠 kafka producer 自身的特性，也就是FlinkKafkaProducer#invoke 里面发送数据之后，Flink 不会再考虑 kafka 是否已经正确的收到数据。

transaction.producer.send(record, callback);

####  Semantic.AT_LEAST_ONCE


这种语义下，除了会走上面说到的发送数据的流程外，如果开启了 checkpoint 功能，在 FlinkKafkaProducer#snapshotState 中会首先执行父类的 snapshotState方法，里面最终会执行 FlinkKafkaProducer#preCommit。


```java
@Override
    protected void preCommit(FlinkKafkaProducer.KafkaTransactionState transaction)
            throws FlinkKafkaException {
        switch (semantic) {
            case EXACTLY_ONCE:
            case AT_LEAST_ONCE:
                flush(transaction);
                break;
            case NONE:
                break;
            default:
                throw new UnsupportedOperationException("Not implemented semantic");
        }
        checkErroneous();
    }
```
AT_LEAST_ONCE 会执行了 flush 方法，里面执行了：

```java
    /**
     * Flush pending records.
     *
     * @param transaction
     */
    private void flush(FlinkKafkaProducer.KafkaTransactionState transaction)
            throws FlinkKafkaException {
        if (transaction.producer != null) {
            transaction.producer.flush();
        }
        long pendingRecordsCount = pendingRecords.get();
        if (pendingRecordsCount != 0) {
            throw new IllegalStateException(
                    "Pending record count must be zero at this point: " + pendingRecordsCount);
        }

        // if the flushed requests has errors, we should propagate it also and fail the checkpoint
        checkErroneous();
    }
```
这个函数主要做的事就是transaction.producer.flush();

就是将 send 的数据立即发送给 kafka 服务端，详细含义可以参考 KafkaProducer api：http://kafka.apache.org/23/javadoc/index.html?org/apache/kafka/clients/producer/KafkaProducer.html

> flush()  
> Invoking this method makes all buffered records immediately available to send (even if linger.ms is greater than 0) and blocks on the completion of the requests associated with these records.  


EXACTLY_ONCE 语义也会执行 send 和 flush 方法，但是同时会开启 kafka producer 的事务机制。FlinkKafkaProducer 中 beginTransaction 的源码如下，可以看到只有是 EXACTLY_ONCE 模式才会真正开始一个事务。

```java
@Override
    protected FlinkKafkaProducer.KafkaTransactionState beginTransaction()
            throws FlinkKafkaException {
        switch (semantic) {
            case EXACTLY_ONCE:
                FlinkKafkaInternalProducer<byte[], byte[]> producer = createTransactionalProducer();
                producer.beginTransaction();
                return new FlinkKafkaProducer.KafkaTransactionState(
                        producer.getTransactionalId(), producer);
            case AT_LEAST_ONCE:
            case NONE:
                // Do not create new producer on each beginTransaction() if it is not necessary
                final FlinkKafkaProducer.KafkaTransactionState currentTransaction =
                        currentTransaction();
                if (currentTransaction != null && currentTransaction.producer != null) {
                    return new FlinkKafkaProducer.KafkaTransactionState(
                            currentTransaction.producer);
                }
                return new FlinkKafkaProducer.KafkaTransactionState(
                        initNonTransactionalProducer(true));
            default:
                throw new UnsupportedOperationException("Not implemented semantic");
        }
    }
```
和 AT_LEAST_ONCE 另一个不同的地方在于 checkpoint 的时候，会将事务相关信息保存到变量 nextTransactionalIdHintState 中，这个变量存储的信息会作为 checkpoint 中的一部分进行持久化。

```java
@Override
    public void snapshotState(FunctionSnapshotContext context) throws Exception {
        super.snapshotState(context);

        nextTransactionalIdHintState.clear();
        // To avoid duplication only first subtask keeps track of next transactional id hint.
        // Otherwise all of the
        // subtasks would write exactly same information.
        if (getRuntimeContext().getIndexOfThisSubtask() == 0
                && semantic == FlinkKafkaProducer.Semantic.EXACTLY_ONCE) {
            checkState(
                    nextTransactionalIdHint != null,
                    "nextTransactionalIdHint must be set for EXACTLY_ONCE");
            long nextFreeTransactionalId = nextTransactionalIdHint.nextFreeTransactionalId;

            // If we scaled up, some (unknown) subtask must have created new transactional ids from
            // scratch. In that
            // case we adjust nextFreeTransactionalId by the range of transactionalIds that could be
            // used for this
            // scaling up.
            if (getRuntimeContext().getNumberOfParallelSubtasks()
                    > nextTransactionalIdHint.lastParallelism) {
                nextFreeTransactionalId +=
                        getRuntimeContext().getNumberOfParallelSubtasks() * kafkaProducersPoolSize;
            }

            nextTransactionalIdHintState.add(
                    new FlinkKafkaProducer.NextTransactionalIdHint(
                            getRuntimeContext().getNumberOfParallelSubtasks(),
                            nextFreeTransactionalId));
        }
    }
```

### 完整调用流程


- snapshotState（开始checkPoint）
- - preCommit
- - - flush
- - beginTransactionInternal
- - - beginTransaction
- notifyCheckpointComplete （完成checkPoint并且上传到TM后回调）
- - commit
- - - commitTransaction 


![image](https://note.youdao.com/yws/api/personal/file/WEB900ed00110d7308dbe205483c13ac373?method=download&shareKey=5aadbab2a9632e0fe4e87c85e8e3a2a5)

![image](https://note.youdao.com/yws/api/personal/file/WEB584e419c48c80d4a72abb7628d00ef1d?method=download&shareKey=59a5c524d777ed6bb4785672c33d3458)

### 完整性差别

如果Source->map->sink的topology中，如果完成下一次checkpoint前，已经出现了5条数据。

* none模式

5条数据已经在kafkaClient的send队列中了，是否发送取决于 LINGER_MS 和 BATCH_SIZE 两个参数，如果这两个参数过大，程序重启可能会丢数据，丢的数据是上几个checkpoint种还没来得及flush的数据，这次还没checkpoint的数据并没有丢

* AtLeastOnce


5条数据已经在kafkaClient的send队列中了，并且每次checkPoint的时候，都会flush kakfaClient的send队列，保证每次新的checkpoint没有残留上一个checkPoint的数据。如果send这次的数据出现程序重启，那么就会重新发送数据，但是不会出现丢数据的情况。

* exactly once

5条数据都跟着事务走，checkpoint的时候先preCommit，如果checkpoint完成并且tm回掉了，那么就提交事务commit。每个事务都是new一个producter，保存对应的事务id到状态中。







### Reference

https://flink.apache.org/features/2018/03/01/end-to-end-exactly-once-apache-flink.html


https://developer.aliyun.com/article/752225

https://zhuanlan.zhihu.com/p/111304281


https://blog.csdn.net/alex_xfboy/article/details/82988259