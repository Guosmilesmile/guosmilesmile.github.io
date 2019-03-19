---
title: Kafka kafkaProducer源码解析
date: 2019-03-17 13:39:57
tags:
categories: kafka
---


### send方法

```
graph TB
A[确保topic对应的metaData是否可得]-->B[序列化key和value]
B-->C[如果没有指定partition,使用partitioner分配partition]
C-->D[生成对应的TopicPartition:String topic,int partition]
D-->E[用key和value生成RecordAppendResult,加入缓存池中]
E-->F[如果创新一个新的batch或者batch满了,唤醒发送线程]
```

<!--more-->
```java
private Future<RecordMetadata> doSend(ProducerRecord<K, V> record, Callback callback) {
        TopicPartition tp = null;
        try {
            // first make sure the metadata for the topic is available
            ClusterAndWaitTime clusterAndWaitTime = waitOnMetadata(record.topic(), record.partition(), maxBlockTimeMs);
            long remainingWaitMs = Math.max(0, maxBlockTimeMs - clusterAndWaitTime.waitedOnMetadataMs);
            Cluster cluster = clusterAndWaitTime.cluster;
            byte[] serializedKey;
            try {
                serializedKey = keySerializer.serialize(record.topic(), record.headers(), record.key());
            } catch (ClassCastException cce) {
                throw new SerializationException("Can't convert key of class " + record.key().getClass().getName() +
                        " to class " + producerConfig.getClass(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG).getName() +
                        " specified in key.serializer", cce);
            }
            byte[] serializedValue;
            try {
                serializedValue = valueSerializer.serialize(record.topic(), record.headers(), record.value());
            } catch (ClassCastException cce) {
                throw new SerializationException("Can't convert value of class " + record.value().getClass().getName() +
                        " to class " + producerConfig.getClass(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG).getName() +
                        " specified in value.serializer", cce);
            }
            int partition = partition(record, serializedKey, serializedValue, cluster);
            tp = new TopicPartition(record.topic(), partition);

            setReadOnly(record.headers());
            Header[] headers = record.headers().toArray();

            int serializedSize = AbstractRecords.estimateSizeInBytesUpperBound(apiVersions.maxUsableProduceMagic(),
                    compressionType, serializedKey, serializedValue, headers);
            ensureValidRecordSize(serializedSize);
            long timestamp = record.timestamp() == null ? time.milliseconds() : record.timestamp();
            log.trace("Sending record {} with callback {} to topic {} partition {}", record, callback, record.topic(), partition);
            // producer callback will make sure to call both 'callback' and interceptor callback
            Callback interceptCallback = new InterceptorCallback<>(callback, this.interceptors, tp);

            if (transactionManager != null && transactionManager.isTransactional())
                transactionManager.maybeAddPartitionToTransaction(tp);

            RecordAccumulator.RecordAppendResult result = accumulator.append(tp, timestamp, serializedKey,
                    serializedValue, headers, interceptCallback, remainingWaitMs);
            if (result.batchIsFull || result.newBatchCreated) {
                log.trace("Waking up the sender since topic {} partition {} is either full or getting a new batch", record.topic(), partition);
                this.sender.wakeup();
            }
            return result.future;
            // handling exceptions and record the errors;
            // for API exceptions return them in the future,
            // for other exceptions throw directly
        } catch (ApiException e) {
            log.debug("Exception occurred during message send:", e);
            if (callback != null)
                callback.onCompletion(null, e);
            this.errors.record();
            this.interceptors.onSendError(record, tp, e);
            return new FutureFailure(e);
        } catch (InterruptedException e) {
            this.errors.record();
            this.interceptors.onSendError(record, tp, e);
            throw new InterruptException(e);
        } catch (BufferExhaustedException e) {
            this.errors.record();
            this.metrics.sensor("buffer-exhausted-records").record();
            this.interceptors.onSendError(record, tp, e);
            throw e;
        } catch (KafkaException e) {
            this.errors.record();
            this.interceptors.onSendError(record, tp, e);
            throw e;
        } catch (Exception e) {
            // we notify interceptor about all exceptions, since onSend is called before anything else in this method
            this.interceptors.onSendError(record, tp, e);
            throw e;
        }
    }
```


#### waitOnMetadata
等待获取集群的元数据，包括对应topic所有可得的partition



#### DefaultPartitioner.partition
对要发送的数据分配对应的partition

* 获取对应topic的所有partition信息和partition总数
* 如果key存在，对key进行hash，然后取模partition总数
* 如果key不存在，获取当前topic的对应的随机数，然后获取当前topic存活的partition，如果存在取模当前存活partition总数，返回对应partition的int类型。如果不存在存活的partition，直接取模partition总数，返回对应partition的int类型。


```java
 public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
        List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
        int numPartitions = partitions.size();
        if (keyBytes == null) {
            int nextValue = nextValue(topic);
            List<PartitionInfo> availablePartitions = cluster.availablePartitionsForTopic(topic);
            if (availablePartitions.size() > 0) {
                int part = Utils.toPositive(nextValue) % availablePartitions.size();
                return availablePartitions.get(part).partition();
            } else {
                // no partitions are available, give a non-available partition
                return Utils.toPositive(nextValue) % numPartitions;
            }
        } else {
            // hash the keyBytes to choose a partition
            return Utils.toPositive(Utils.murmur2(keyBytes)) % numPartitions;
        }
    }

    //如果当前topic不存在对应的counter，随机生成一个，然后+1，如果存在，直接+1
    private int nextValue(String topic) {
        AtomicInteger counter = topicCounterMap.get(topic);
        if (null == counter) {
            counter = new AtomicInteger(ThreadLocalRandom.current().nextInt());
            AtomicInteger currentCounter = topicCounterMap.putIfAbsent(topic, counter);
            if (currentCounter != null) {
                counter = currentCounter;
            }
        }
        return counter.getAndIncrement();
    }
```


#### producter启动的时候如果更新metadata
在kafkaProducter中，producter负责读取metadata，sender中的MetadataUpdater负责更新metadata，在sender中有一个默认的NetworkClient负责获取网络上获取个各种信息。默认的metadata为DefaultMetadataUpdater，这个update调用了client中的leastLoadedNode随机获取一个node去连接对应的broker获取整个集群的拓扑信息。**[选择有最少的未发送请求的node，要求这些node至少是可以连接的。这个方法会优先选择有可用的连接的节点，但是如果所有的已连接的节点都在使用，它就会选择还没有建立连接的节点。这个方法绝对不会选择忆经断开连接的节点或者正在reconnect backoff阶段的连接。]**

* 获取所有node信息，并且随机一个0至node个数的随机数offset
* for循环节点个数，以offset为起点，获取对应位置的node
* 获取对该node所有正在请求中的request的个数，如果为0而且已经建立连接，对该node没有请求在飞行中，返回该node（然后刚刚启动，并没有建立连接）
* 如果不满足上面的条件，那么如果我们与给定节点断开连接并且无法重新建立连接，也不会选择该节点，直到遇到有连接过的节点或者没连接过的节点。
```java
 public Node leastLoadedNode(long now) {
        List<Node> nodes = this.metadataUpdater.fetchNodes();
        int inflight = Integer.MAX_VALUE;
        Node found = null;

        int offset = this.randOffset.nextInt(nodes.size());
        for (int i = 0; i < nodes.size(); i++) {
            int idx = (offset + i) % nodes.size();
            Node node = nodes.get(idx);
            int currInflight = this.inFlightRequests.count(node.idString());
            if (currInflight == 0 && isReady(node, now)) {
                // if we find an established connection with no in-flight requests we can stop right away
                log.trace("Found least loaded node {} connected with no in-flight requests", node);
                return node;
            } else if (!this.connectionStates.isBlackedOut(node.idString(), now) && currInflight < inflight) {
                // otherwise if this is the best we have found so far, record that
                inflight = currInflight;
                found = node;
            } else if (log.isTraceEnabled()) {
                log.trace("Removing node {} from least loaded node selection: is-blacked-out: {}, in-flight-requests: {}",
                        node, this.connectionStates.isBlackedOut(node.idString(), now), currInflight);
            }
        }

        if (found != null)
            log.trace("Found least loaded node {}", found);
        else
            log.trace("Least loaded node selection failed to find an available node");

        return found;
    }
```

在0.8.2.2的kafkaProducter中存在一个bug，如果kafka的地址中存在一台没有kafka，新旧两个版本都有有一个random。新版的random的offset是每次调用函数都重新生成，而旧版的random在初始化的生成，调用函数不会更新，那么就会出现一个bug，如果获取的是错误的那台服务器，每次重新调用就都会去拿这个服务器，就会一直出错。


```java
public Node leastLoadedNode(long now) {
        List<Node> nodes = this.metadata.fetch().nodes();
        int inflight = Integer.MAX_VALUE;
        Node found = null;
        for (int i = 0; i < nodes.size(); i++) {
            //this.nodeIndexOffset初始化的时候生成，再不更新，会出现一开始初始化的时候都没有已经连接的node，如果存在错误的服务器，会一直连接这台。
            int idx = Utils.abs((this.nodeIndexOffset + i) % nodes.size());
            Node node = nodes.get(idx);
            int currInflight = this.inFlightRequests.inFlightRequestCount(node.id());
            if (currInflight == 0 && this.connectionStates.isConnected(node.id())) {
                // if we find an established connection with no in-flight requests we can stop right away
                return node;
            } else if (!this.connectionStates.isBlackedOut(node.id(), now) && currInflight < inflight) {
                // otherwise if this is the best we have found so far, record that
                inflight = currInflight;
                found = node;
            }
        }

        return found;
    }
```

#### RecordAccumulator 缓冲池

* 添加一条记录，计数器加一
* 获取对应topic的partition的双端队列，队列里头存储的是一个个batch，如果不存在，直接创建新的双端队列
* 添加对应记录，如果batch有剩余空间，则添加完成
* 如果没有剩余空间，创建新的batch，分配新的空间，如果在bolck默认内存不够会锁住等待一定时间
* 再次尝试一次在原来的空间添加记录，（可能会出现垃圾回收后有空间），如果成功则成功，如果不成功，在新分配的空间添加batch。

#### Sender 发送线程



https://www.jianshu.com/p/4b4e6d2455bc   
https://www.cnblogs.com/benfly/p/9360563.html             
https://blog.csdn.net/chunlongyu/article/details/52622422