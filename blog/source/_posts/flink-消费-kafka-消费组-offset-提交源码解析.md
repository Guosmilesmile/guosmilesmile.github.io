---
title: flink 消费 kafka 消费组 offset 提交源码解析
date: 2021-06-19 17:17:42
tags:
categories:
	- Flink
---


flink 消费 kafka 数据，提交消费组 offset 有三种类型

* 开启 checkpoint ：在 checkpoint 完成后提交
* 开启 checkpoint，禁用 checkpoint 提交： 不提交消费组 offset
* 不开启 checkpoint：依赖kafka client 的自动提交


一个简单的 flink 程序： 读取kafka topic 数据，写到另一个 topic


```java
val env: StreamExecutionEnvironment = StreamExecutionEnvironment.getExecutionEnvironment
env.setParallelism(1)
// enable checkpoint
val stateBackend = new FsStateBackend("file:///out/checkpoint")
env.setStateBackend(stateBackend)
env.enableCheckpointing(1 * 60 * 1000, CheckpointingMode.EXACTLY_ONCE)

val prop = Common.getProp
//        prop.setProperty("enable.auto.commit", "true")
//        prop.setProperty("auto.commit.interval.ms", "15000")
val kafkaSource = new FlinkKafkaConsumer[String]("kafka_offset", new SimpleStringSchema(), prop)
//        kafkaSource.setCommitOffsetsOnCheckpoints(false)

val kafkaProducer = new FlinkKafkaProducer[String]("kafka_offset_out", new SimpleStringSchema(), prop)
//        kafkaProducer.setWriteTimestampToKafka(true)

env.addSource(kafkaSource)
  .setParallelism(1)
  .map(node => {
    node.toString + ",flinkx"
  })
  .addSink(kafkaProducer)

// execute job
env.execute("KafkaToKafka")
```

### 1 启动 checkpoint

开启checkpoint 默认值就是 消费组 offset 的提交方式是： ON_CHECKPOINTS

offsetCommitMode 提交方法在 FlinkKafkaConsumerBase open 的时候会设置：

FlinkKafkaConsumer 提交消费者的 offset 的行为在 FlinkKafkaConsumerBase open 的时候会设置：

```
@Override
public void open(Configuration configuration) throws Exception {
  // determine the offset commit mode
  this.offsetCommitMode = OffsetCommitModes.fromConfiguration(
      getIsAutoCommitEnabled(),
      enableCommitOnCheckpoints,  // 默认值 true
      ((StreamingRuntimeContext) getRuntimeContext()).isCheckpointingEnabled());
```

OffsetCommitModes.java
```java
public static OffsetCommitMode fromConfiguration(
            boolean enableAutoCommit,
            boolean enableCommitOnCheckpoint,
            boolean enableCheckpointing) {

        if (enableCheckpointing) {
            // if checkpointing is enabled, the mode depends only on whether committing on
            // checkpoints is enabled
            return (enableCommitOnCheckpoint)
                    ? OffsetCommitMode.ON_CHECKPOINTS
                    : OffsetCommitMode.DISABLED;
        } else {
            // else, the mode depends only on whether auto committing is enabled in the provided
            // Kafka properties
            return (enableAutoCommit) ? OffsetCommitMode.KAFKA_PERIODIC : OffsetCommitMode.DISABLED;
        }
    }
```
可以看出，如果开启了checkpoint但是没开启enableCommitOnCheckpoint就会选择OffsetCommitMode.DISABLED（不提交offset）



当 flink 触发一次 checkpoint 的时候，会依次调用所有算子的 notifyCheckpointComplete 方法，kafka source 会调用到 FlinkKafkaConsumerBase.notifyCheckpointComplete

注：FlinkKafkaConsumerBase 是 FlinkKafkaConsumer 的父类

```java
@Override
public final void notifyCheckpointComplete(long checkpointId) throws Exception {
  ....

  if (offsetCommitMode == OffsetCommitMode.ON_CHECKPOINTS) {
    // only one commit operation must be in progress
    ...

    try {
      // 获取当前checkpoint id 对应的待提交的 offset index
      final int posInMap = pendingOffsetsToCommit.indexOf(checkpointId);
      if (posInMap == -1) {
        LOG.warn("Consumer subtask {} received confirmation for unknown checkpoint id {}",
          getRuntimeContext().getIndexOfThisSubtask(), checkpointId);
        return;
      }
      // 根据 offset index 获取 offset 值，待提交的就直接删除了
      @SuppressWarnings("unchecked")
      Map<KafkaTopicPartition, Long> offsets =
        (Map<KafkaTopicPartition, Long>) pendingOffsetsToCommit.remove(posInMap);
      
      ....

      // 调用 KafkaFetcher的 commitInternalOffsetsToKafka 方法 提交 offset
      fetcher.commitInternalOffsetsToKafka(offsets, offsetCommitCallback);
    
    ....
```

最后调用了 AbstractFetcher.commitInternalOffsetsToKafka 

```java
public final void commitInternalOffsetsToKafka(
    Map<KafkaTopicPartition, Long> offsets,
    @Nonnull KafkaCommitCallback commitCallback) throws Exception {
  // Ignore sentinels. They might appear here if snapshot has started before actual offsets values
  // replaced sentinels
  doCommitInternalOffsetsToKafka(filterOutSentinels(offsets), commitCallback);
}

protected abstract void doCommitInternalOffsetsToKafka(
    Map<KafkaTopicPartition, Long> offsets,
    @Nonnull KafkaCommitCallback commitCallback) throws Exception;
```

AbstractFetcher.doCommitInternalOffsetsToKafka 的实现 KafkaFetcher.doCommitInternalOffsetsToKafka

使用 Map<KafkaTopicPartition, Long> offsets 构造提交 kafka offset 的 Map<TopicPartition, OffsetAndMetadata> offsetsToCommit

注：offset + 1 表示下一次消费的位置

```java
@Override
protected void doCommitInternalOffsetsToKafka(
  Map<KafkaTopicPartition, Long> offsets,
  @Nonnull KafkaCommitCallback commitCallback) throws Exception {

  @SuppressWarnings("unchecked")
  List<KafkaTopicPartitionState<T, TopicPartition>> partitions = subscribedPartitionStates();

  Map<TopicPartition, OffsetAndMetadata> offsetsToCommit = new HashMap<>(partitions.size());

  for (KafkaTopicPartitionState<T, TopicPartition> partition : partitions) {
    Long lastProcessedOffset = offsets.get(partition.getKafkaTopicPartition());
    if (lastProcessedOffset != null) {
      checkState(lastProcessedOffset >= 0, "Illegal offset value to commit");

      // committed offsets through the KafkaConsumer need to be 1 more than the last processed offset.
      // This does not affect Flink's checkpoints/saved state.
      long offsetToCommit = lastProcessedOffset + 1;

      offsetsToCommit.put(partition.getKafkaPartitionHandle(), new OffsetAndMetadata(offsetToCommit));
      partition.setCommittedOffset(offsetToCommit);
    }
  }

  // record the work to be committed by the main consumer thread and make sure the consumer notices that
  consumerThread.setOffsetsToCommit(offsetsToCommit, commitCallback);
}
```

然后调用 KafkaConsumerThread.setOffsetsToCommit:  将待提交的 offset 放到 kafka 的消费线程对于的属性 nextOffsetsToCommit 中，等待下一个消费循环提交

```java
void setOffsetsToCommit(
      Map<TopicPartition, OffsetAndMetadata> offsetsToCommit,
      @Nonnull KafkaCommitCallback commitCallback) {

    // 把待提交的 offsetsToCommit 放到 nextOffsetsToCommit 中，供 kafka 的消费线程来取
    // 返回值不为 null，说明上次的没提交完成
    // record the work to be committed by the main consumer thread and make sure the consumer notices that
    if (nextOffsetsToCommit.getAndSet(Tuple2.of(offsetsToCommit, commitCallback)) != null) {
      log.warn("Committing offsets to Kafka takes longer than the checkpoint interval. " +
          "Skipping commit of previous offsets because newer complete checkpoint offsets are available. " +
          "This does not compromise Flink's checkpoint integrity.");
    }

    // if the consumer is blocked in a poll() or handover operation, wake it up to commit soon
    handover.wakeupProducer();

    synchronized (consumerReassignmentLock) {
      if (consumer != null) {
        consumer.wakeup();
      } else {
        // the consumer is currently isolated for partition reassignment;
        // set this flag so that the wakeup state is restored once the reassignment is complete
        hasBufferedWakeup = true;
      }
    }
  }
```

![image](https://note.youdao.com/yws/api/personal/file/08E43DE1C182414AADD39CE68E31ABE8?method=download&shareKey=3f2d716974f9a7c688985ab5ec61fbbe)

然后就到了kafka 消费的线程，KafkaConsumerThread.run 方法中：  这里是消费 kafka 数据的地方，也提交对应消费组的offset

```java
@Override
  public void run() {
    ...

      this.consumer = getConsumer(kafkaProperties);

    ....
      // 循环从kafka poll 数据
      // main fetch loop 
      while (running) {
        // 这里就是提交 offset 的地方了
        // check if there is something to commit
        if (!commitInProgress) {

          // nextOffsetsToCommit 就是 那边线程放入 offset 的对象了

          // get and reset the work-to-be committed, so we don't repeatedly commit the same
          final Tuple2<Map<TopicPartition, OffsetAndMetadata>, KafkaCommitCallback> commitOffsetsAndCallback =
              nextOffsetsToCommit.getAndSet(null);

          // 如果取出commitOffsetsAndCallback 不为空，就异步提交 offset 到kafka
          if (commitOffsetsAndCallback != null) {
            log.debug("Sending async offset commit request to Kafka broker");

            // also record that a commit is already in progress
            // the order here matters! first set the flag, then send the commit command.
            commitInProgress = true;
            consumer.commitAsync(commitOffsetsAndCallback.f0, new CommitCallback(commitOffsetsAndCallback.f1));
          }
        }

       ... 
        // get the next batch of records, unless we did not manage to hand the old batch over
        if (records == null) {
          try {
            records = consumer.poll(pollTimeout);
          }
          catch (WakeupException we) {
            continue;
          }
        }

        ...
      }
```

![image](https://note.youdao.com/yws/api/personal/file/2C48356580BF4DB9A229DBABD5B369B7?method=download&shareKey=93c37ab3eacdfd2dfa4673e58f272387)

到这里就能看到 flink 的offset 提交到了 kafka 中


### kafka如果不提交offset，还能正常消费吗？

kafka client的offset是保存在自身的内存中的，启动的时候发现为空，就会去broker中获取，有多种获取策略。 然后后续的消费，就跟自身内存打交道了，只是无脑提交offset到kafka。后续重启才会再次和broker打交道拿offset。





### Reference

https://www.cnblogs.com/Springmoon-venn/p/13405140.html



