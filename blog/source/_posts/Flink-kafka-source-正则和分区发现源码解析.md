---
title: Flink kafka source 正则和分区发现源码解析
date: 2021-06-19 17:15:02
tags:
categories:
	- Flink
---



## 前置

使用方法

https://ci.apache.org/projects/flink/flink-docs-release-1.13/zh/docs/connectors/datastream/kafka/


## 总结

先上总结，kafka的分区发现基本还是基于另起线程，在另外的线程内，通过kafka的client的基础功能，获取集群所有的topic，与正则进行匹配（居然是框架的功能不是kafka的功能）过滤出符合的topic，通过kafka client获取所有的partition。

flink会保存已经发现过的partition，将新发现的partition和topic名字的hascode还有子任务的id进行一个计算，保证任务可以均衡的分布在每个子任务（等于partition是自我认领的方式）


## 源码解析


FlinkKafkaConsumerBase.java

```java

        if (discoveryIntervalMillis == PARTITION_DISCOVERY_DISABLED) {
            kafkaFetcher.runFetchLoop();
        } else {
            runWithPartitionDiscovery();
        }
```


如果这个discoverIntervalMillis设置为Long.MIN_VALUE就走kafkaFetcher.runFetchLoop();的方法。
如果设置了这个值，并且不是最小值，那么就会走runWithPartitionDiscovery方法。



discoverIntervalMillis是一个设置了的必须大于等于0或者等于Long.min_value。


如果启用了，最终会调用createAndStartDiscoveryLoop()方法，启动一个单独的线程，负责以discoveryIntervalMillis为周期发现新的topic/partition，并传递给KafkaFetcher。



核心代码在FlinkKafkaConsumerBase的createAndStartDiscoveryLoop

```java
private void createAndStartDiscoveryLoop(AtomicReference<Exception> discoveryLoopErrorRef) {
        discoveryLoopThread =
                new Thread(
                        () -> {
                            try {
                                // --------------------- partition discovery loop
                                // ---------------------

                                // throughout the loop, we always eagerly check if we are still
                                // running before
                                // performing the next operation, so that we can escape the loop as
                                // soon as possible

                                while (running) {
                                    if (LOG.isDebugEnabled()) {
                                        LOG.debug(
                                                "Consumer subtask {} is trying to discover new partitions ...",
                                                getRuntimeContext().getIndexOfThisSubtask());
                                    }

                                    final List<KafkaTopicPartition> discoveredPartitions;
                                    try {
                                        discoveredPartitions =
                                                partitionDiscoverer.discoverPartitions();
                                    } catch (AbstractPartitionDiscoverer.WakeupException
                                            | AbstractPartitionDiscoverer.ClosedException e) {
                                        // the partition discoverer may have been closed or woken up
                                        // before or during the discovery;
                                        // this would only happen if the consumer was canceled;
                                        // simply escape the loop
                                        break;
                                    }

                                    // no need to add the discovered partitions if we were closed
                                    // during the meantime
                                    if (running && !discoveredPartitions.isEmpty()) {
                                        kafkaFetcher.addDiscoveredPartitions(discoveredPartitions);
                                    }

                                    // do not waste any time sleeping if we're not running anymore
                                    if (running && discoveryIntervalMillis != 0) {
                                        try {
                                            Thread.sleep(discoveryIntervalMillis);
                                        } catch (InterruptedException iex) {
                                            // may be interrupted if the consumer was canceled
                                            // midway; simply escape the loop
                                            break;
                                        }
                                    }
                                }
                            } catch (Exception e) {
                                discoveryLoopErrorRef.set(e);
                            } finally {
                                // calling cancel will also let the fetcher loop escape
                                // (if not running, cancel() was already called)
                                if (running) {
                                    cancel();
                                }
                            }
                        },
                        "Kafka Partition Discovery for "
                                + getRuntimeContext().getTaskNameWithSubtasks());

        discoveryLoopThread.start();
    }

```

关键代码在这边：

```java
                                try {
                                        discoveredPartitions =
                                                partitionDiscoverer.discoverPartitions();
                                    } catch (AbstractPartitionDiscoverer.WakeupException
                                            | AbstractPartitionDiscoverer.ClosedException e) {
                                        // the partition discoverer may have been closed or woken up
                                        // before or during the discovery;
                                        // this would only happen if the consumer was canceled;
                                        // simply escape the loop
                                        break;
                                    }

                                    // no need to add the discovered partitions if we were closed
                                    // during the meantime
                                    if (running && !discoveredPartitions.isEmpty()) {
                                        kafkaFetcher.addDiscoveredPartitions(discoveredPartitions);
                                    }

```

先关注下AbstractPartitionDiscoverer.discoverPartitions，
主要做了下面这几件事

1. 会根据传入的是单个固定的topic还是由正则表达式指定的多个topics获取所有的可能的partition。
2. 排除旧的partition和不需要这个subtask需要订阅的partition


如果是固定topic调用getAllPartitionsForTopics(topicsDescriptor.getFixedTopics());

如果是基于正则，需要先获取这个集群上的所有的topic，进行正则的匹配（原来这个功能是在框架进行的，不是kafka的功能）。然后调用和固定topic一样的函数getAllPartitionsForTopics

```java
        List<String> matchedTopics = getAllTopics();

        // retain topics that match the pattern
        Iterator<String> iter = matchedTopics.iterator();
        while (iter.hasNext()) {
        if (!topicsDescriptor.isMatchingTopic(iter.next())) {
                     iter.remove();
            }
        }

        if (matchedTopics.size() != 0) {
            // get partitions only for matched topics
            newDiscoveredPartitions = getAllPartitionsForTopics(matchedTopics);
         } else {
            newDiscoveredPartitions = null;
        }
```

KafkaPartitionDiscoverer.getAllPartitionsForTopics主要是用于发现kafka对应topic的partition。这个函数主要是使用kafka自带的client获取这个topic所有的partition

```java
final List<PartitionInfo> kafkaPartitions = kafkaConsumer.partitionsFor(topic);
```

获取所有的topic也是一样调用kafka client自带的方法

```java
kafkaConsumer.listTopics()
```


回归主线，看第二步，如何排除无关的partition。关键是AbstractPartitionDiscoverer.setAndCheckDiscoveredPartition.

AbstractPartitionDiscoverer这个类中有一个变量discoveredPartitions存储已经被发现过的partition.非该存储中的均为新发现的partition。


如何判断这个partition是否是这个subtask需要消费的。通过getRuntimeContext().getIndexOfThisSubtask()获取该任务的index，通过getRuntimeContext().getNumberOfParallelSubtasks()获取总任务的并行度。调用assign进行判断。

assign方法进行了比较多的考虑，产生的单个主题的分区分布具有以下特点
1. 均匀地分布在子任务中
2. 通过使用分区ID作为起始索引的偏移量

topicA 可能是从subtask 10 ，11 ， 12 ... 开始分布   
topicB 可能是从subtask 2 ，3 ，4 ...  开始分布

为了错开，进行讲topic名字的hashCode作为计算分部的一部分。

```java
/**
     * Sets a partition as discovered. Partitions are considered as new if its partition id is
     * larger than all partition ids previously seen for the topic it belongs to. Therefore, for a
     * set of discovered partitions, the order that this method is invoked with each partition is
     * important.
     *
     * <p>If the partition is indeed newly discovered, this method also returns whether the new
     * partition should be subscribed by this subtask.
     *
     * @param partition the partition to set and check
     * @return {@code true}, if the partition wasn't seen before and should be subscribed by this
     *     subtask; {@code false} otherwise
     */
    public boolean setAndCheckDiscoveredPartition(KafkaTopicPartition partition) {
        if (isUndiscoveredPartition(partition)) {
            discoveredPartitions.add(partition);

            return KafkaTopicPartitionAssigner.assign(partition, numParallelSubtasks)
                    == indexOfThisSubtask;
        }

        return false;
    }
    
    
    /**
     * Returns the index of the target subtask that a specific Kafka partition should be assigned
     * to.
     *
     * <p>The resulting distribution of partitions of a single topic has the following contract:
     *
     * <ul>
     *   <li>1. Uniformly distributed across subtasks
     *   <li>2. Partitions are round-robin distributed (strictly clockwise w.r.t. ascending subtask
     *       indices) by using the partition id as the offset from a starting index (i.e., the index
     *       of the subtask which partition 0 of the topic will be assigned to, determined using the
     *       topic name).
     * </ul>
     *
     * <p>The above contract is crucial and cannot be broken. Consumer subtasks rely on this
     * contract to locally filter out partitions that it should not subscribe to, guaranteeing that
     * all partitions of a single topic will always be assigned to some subtask in a uniformly
     * distributed manner.
     *
     * @param partition the Kafka partition
     * @param numParallelSubtasks total number of parallel subtasks
     * @return index of the target subtask that the Kafka partition should be assigned to.
     */
    public static int assign(KafkaTopicPartition partition, int numParallelSubtasks) {
        int startIndex =
                ((partition.getTopic().hashCode() * 31) & 0x7FFFFFFF) % numParallelSubtasks;

        // here, the assumption is that the id of Kafka partitions are always ascending
        // starting from 0, and therefore can be used directly as the offset clockwise from the
        // start index
        return (startIndex + partition.getPartition()) % numParallelSubtasks;
    }
```




### Reference


https://cloud.tencent.com/developer/article/1677272      
https://www.cnblogs.com/Springmoon-venn/p/12023350.html      
https://juejin.cn/post/6844903972835344391    