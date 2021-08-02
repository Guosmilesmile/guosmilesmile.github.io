---
title: FlinkKafkaConsumer同groupId消费问题深入分析
date: 2021-05-16 14:25:56
tags:
categories:
	- Flink
---





### 问题
这有两个相同代码的程序：

```

val bsEnv = StreamExecutionEnvironment.getExecutionEnvironment
 Env.setRestartStrategy(RestartStrategies.noRestart())
 val consumerProps = new Properties()
 consumerProps.put("bootstrap.servers", brokers)
 consumerProps.put("group.id", "test1234")

 val consumer = new FlinkKafkaConsumer[String](topic,new KafkaStringSchema,consumerProps).setStartFromLatest()
 Env.addSource(consumer).print()
 Env.execute()
```

同时启动这两个程序，他们连接相同的集群的topic，group.id也一样，然后向topic发送一些数据，发现这两个程序都能消费到发送的所有分区的消息，kafka 的consumer group组内应该是有消费隔离的，为什么这里两个程序都能同时消费到全部数据呢?

而用KafkaConsumer写两个相同的程序去消费这个topic就可以看到两边程序是没有重复消费同一分区的

### 解答


在 Flink 消费 Kafka 的过程中， 由 FlinkKafkaConsumer 会从 Kafka 中拿到当前 topic 的所有 partition 信息并分配并发消费，这里的 group id 只是用于将当前 partition 的消费 offset commit 到 Kafka，并用这个消费组标识。而使用 KafkaConsumer 消费数据则应用到了 Kafka 的消费组管理, 这是 Kafka 服务端的一个角色。


为了保证 Flink 程序的 exactly-once，必须由各个 Kafka source  算子维护当前算子所消费的 partition 消费 offset 信息，并在每次checkpoint 时将这些信息写入到 state 中， 在从 checkpoint 恢复中从上次 commit 的位点开始消费，保证 exactly-once.  如果用 Kafka 消费组管理，那么 FlinkKafkaConsumer 内各个并发实例所分配的 partition 将由 Kafka 的消费组管理，且 offset 也由 Kafka 消费组管理者记录，Flink 无法维护这些信息。





### 注意

，当启动两个作业用同一个 topic 和 group id 消费 kafka， 如果两个作业会分别以同一个 group id commit offset 到kafka， 如果以 group offset 消费模式启动作业， 则会以最后一次 commit 的 offset 开始消费。



### 源码分析


先看下社区的分析：

http://apache-flink-user-mailing-list-archive.2336050.n4.nabble.com/Flink-kafka-group-question-td8185.html#none

Internally, the Flink Kafka connectors don’t use the consumer group management functionality because they are using lower-level APIs (SimpleConsumer in 0.8, and KafkaConsumer#assign(…) in 0.9) on each parallel instance for more control on individual partition consumption. So, essentially, the “group.id” setting in the Flink Kafka connector is only used for committing offsets back to ZK / Kafka brokers.

flink的版本没有用到group id这个属性。。

初步结论
https://issues.apache.org/jira/browse/FLINK-11325

connecter消费数据的时候，使用 ./bin/kafka-consumer-groups.sh就是无法获取 CONSUMER-ID HOST CLIENT-ID等值。因为Flink实现connecter的时候，就没有使用到kafka的这个feature。此时我们需要通过Flink的metric可以看到消费情况。

进一步看源码吧：

### KafkaConsumer实现



我们使用kafka-client的时候，一般使用KafkaConsumer构建我们的消费实例，使用poll来获取数据:

```
KafkaConsumer<Integer, String> consumer = new KafkaConsumer<>(props);
……
ConsumerRecords<Integer, String> records = consumer.poll();
```


我们看下org.apache.kafka.clients.consumer.KafkaConsumer核心部分


```java
 private KafkaConsumer(ConsumerConfig config, Deserializer<K> keyDeserializer, Deserializer<V> valueDeserializer) {
        try {
            String clientId = config.getString(ConsumerConfig.CLIENT_ID_CONFIG);
            if (clientId.isEmpty()) // 
                clientId = "consumer-" + CONSUMER_CLIENT_ID_SEQUENCE.getAndIncrement();
            this.clientId = clientId;
            this.groupId = config.getString(ConsumerConfig.GROUP_ID_CONFIG);
            LogContext logContext = new LogContext("[Consumer clientId=" + clientId + ", groupId=" + groupId + "] ");
            this.log = logContext.logger(getClass());
            boolean enableAutoCommit = config.getBoolean(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG);
            if (groupId == null) { // 未指定groupId的情况下，不能设置”自动提交offset“
                if (!config.originals().containsKey(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG))
                    enableAutoCommit = false;
                else if (enableAutoCommit)
                    throw new InvalidConfigurationException(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG + " cannot be set to true when default group id (null) is used.");
            } else if (groupId.isEmpty())
                log.warn("Support for using the empty group id by consumers is deprecated and will be removed in the next major release.");

            log.debug("Initializing the Kafka consumer");
            this.requestTimeoutMs = config.getInt(ConsumerConfig.REQUEST_TIMEOUT_MS_CONFIG);
            this.defaultApiTimeoutMs = config.getInt(ConsumerConfig.DEFAULT_API_TIMEOUT_MS_CONFIG);
            this.time = Time.SYSTEM;

            Map<String, String> metricsTags = Collections.singletonMap("client-id", clientId);
            MetricConfig metricConfig = new MetricConfig().samples(config.getInt(ConsumerConfig.METRICS_NUM_SAMPLES_CONFIG))
                    .timeWindow(config.getLong(ConsumerConfig.METRICS_SAMPLE_WINDOW_MS_CONFIG), TimeUnit.MILLISECONDS)
                    .recordLevel(Sensor.RecordingLevel.forName(config.getString(ConsumerConfig.METRICS_RECORDING_LEVEL_CONFIG)))
                    .tags(metricsTags);
            List<MetricsReporter> reporters = config.getConfiguredInstances(ConsumerConfig.METRIC_REPORTER_CLASSES_CONFIG,
                    MetricsReporter.class, Collections.singletonMap(ConsumerConfig.CLIENT_ID_CONFIG, clientId));
            reporters.add(new JmxReporter(JMX_PREFIX));
            this.metrics = new Metrics(metricConfig, reporters, time);
            this.retryBackoffMs = config.getLong(ConsumerConfig.RETRY_BACKOFF_MS_CONFIG);

            // load interceptors and make sure they get clientId
            Map<String, Object> userProvidedConfigs = config.originals();
            userProvidedConfigs.put(ConsumerConfig.CLIENT_ID_CONFIG, clientId);
            List<ConsumerInterceptor<K, V>> interceptorList = (List) (new ConsumerConfig(userProvidedConfigs, false)).getConfiguredInstances(ConsumerConfig.INTERCEPTOR_CLASSES_CONFIG,
                    ConsumerInterceptor.class);
            this.interceptors = new ConsumerInterceptors<>(interceptorList);
            if (keyDeserializer == null) {
                this.keyDeserializer = config.getConfiguredInstance(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, Deserializer.class);
                this.keyDeserializer.configure(config.originals(), true);
            } else {
                config.ignore(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG);
                this.keyDeserializer = keyDeserializer;
            }
            if (valueDeserializer == null) {
                this.valueDeserializer = config.getConfiguredInstance(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, Deserializer.class);
                this.valueDeserializer.configure(config.originals(), false);
            } else {
                config.ignore(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG);
                this.valueDeserializer = valueDeserializer;
            }
            ClusterResourceListeners clusterResourceListeners = configureClusterResourceListeners(keyDeserializer, valueDeserializer, reporters, interceptorList);
            this.metadata = new Metadata(retryBackoffMs, config.getLong(ConsumerConfig.METADATA_MAX_AGE_CONFIG),
                    true, false, clusterResourceListeners); 
            List<InetSocketAddress> addresses = ClientUtils.parseAndValidateAddresses(
                    config.getList(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG), config.getString(ConsumerConfig.CLIENT_DNS_LOOKUP_CONFIG));
            this.metadata.bootstrap(addresses, time.milliseconds());
            String metricGrpPrefix = "consumer";
            ConsumerMetrics metricsRegistry = new ConsumerMetrics(metricsTags.keySet(), "consumer");
            ChannelBuilder channelBuilder = ClientUtils.createChannelBuilder(config, time);
            IsolationLevel isolationLevel = IsolationLevel.valueOf(
                    config.getString(ConsumerConfig.ISOLATION_LEVEL_CONFIG).toUpperCase(Locale.ROOT));
            Sensor throttleTimeSensor = Fetcher.throttleTimeSensor(metrics, metricsRegistry.fetcherMetrics);
            int heartbeatIntervalMs = config.getInt(ConsumerConfig.HEARTBEAT_INTERVAL_MS_CONFIG);

            NetworkClient netClient = new NetworkClient(
                    new Selector(config.getLong(ConsumerConfig.CONNECTIONS_MAX_IDLE_MS_CONFIG), metrics, time, metricGrpPrefix, channelBuilder, logContext),
                    this.metadata,
                    clientId,
                    100, // a fixed large enough value will suffice for max in-flight requests
                    config.getLong(ConsumerConfig.RECONNECT_BACKOFF_MS_CONFIG),
                    config.getLong(ConsumerConfig.RECONNECT_BACKOFF_MAX_MS_CONFIG),
                    config.getInt(ConsumerConfig.SEND_BUFFER_CONFIG),
                    config.getInt(ConsumerConfig.RECEIVE_BUFFER_CONFIG),
                    config.getInt(ConsumerConfig.REQUEST_TIMEOUT_MS_CONFIG),
                    ClientDnsLookup.forConfig(config.getString(ConsumerConfig.CLIENT_DNS_LOOKUP_CONFIG)),
                    time,
                    true,
                    new ApiVersions(),
                    throttleTimeSensor,
                    logContext);
            this.client = new ConsumerNetworkClient(
                    logContext,
                    netClient,
                    metadata,
                    time,
                    retryBackoffMs,
                    config.getInt(ConsumerConfig.REQUEST_TIMEOUT_MS_CONFIG),
                    heartbeatIntervalMs); //Will avoid blocking an extended period of time to prevent heartbeat thread starvation
            OffsetResetStrategy offsetResetStrategy = OffsetResetStrategy.valueOf(config.getString(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG).toUpperCase(Locale.ROOT));
            this.subscriptions = new SubscriptionState(offsetResetStrategy);
            this.assignors = config.getConfiguredInstances(
                    ConsumerConfig.PARTITION_ASSIGNMENT_STRATEGY_CONFIG,
                    PartitionAssignor.class);

            int maxPollIntervalMs = config.getInt(ConsumerConfig.MAX_POLL_INTERVAL_MS_CONFIG);
            int sessionTimeoutMs = config.getInt(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG);
            // 指定groupId才会构建coordinator。这个coordinator后面会协调消费情况
            this.coordinator = groupId == null ? null :
                new ConsumerCoordinator(logContext,
                        this.client,
                        groupId,
                        maxPollIntervalMs,
                        sessionTimeoutMs,
                        new Heartbeat(time, sessionTimeoutMs, heartbeatIntervalMs, maxPollIntervalMs, retryBackoffMs),
                        assignors,
                        this.metadata,
                        this.subscriptions,
                        metrics,
                        metricGrpPrefix,
                        this.time,
                        retryBackoffMs,
                        enableAutoCommit,
                        config.getInt(ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG),
                        this.interceptors,
                        config.getBoolean(ConsumerConfig.EXCLUDE_INTERNAL_TOPICS_CONFIG),
                        config.getBoolean(ConsumerConfig.LEAVE_GROUP_ON_CLOSE_CONFIG));
            // 构建获取数据的fetcher
            this.fetcher = new Fetcher<>(
                    logContext,
                    this.client,
                    config.getInt(ConsumerConfig.FETCH_MIN_BYTES_CONFIG),
                    config.getInt(ConsumerConfig.FETCH_MAX_BYTES_CONFIG),
                    config.getInt(ConsumerConfig.FETCH_MAX_WAIT_MS_CONFIG),
                    config.getInt(ConsumerConfig.MAX_PARTITION_FETCH_BYTES_CONFIG),
                    config.getInt(ConsumerConfig.MAX_POLL_RECORDS_CONFIG),
                    config.getBoolean(ConsumerConfig.CHECK_CRCS_CONFIG),
                    this.keyDeserializer,
                    this.valueDeserializer,
                    this.metadata,
                    this.subscriptions,
                    metrics,
                    metricsRegistry.fetcherMetrics,
                    this.time,
                    this.retryBackoffMs,
                    this.requestTimeoutMs,
                    isolationLevel);

            config.logUnused();
            AppInfoParser.registerAppInfo(JMX_PREFIX, clientId, metrics);
            log.debug("Kafka consumer initialized");
        } catch (Throwable t) {
            // call close methods if internal objects are already constructed; this is to prevent resource leak. see KAFKA-2121
            close(0, true);
            // now propagate the exception
            throw new KafkaException("Failed to construct kafka consumer", t);
        }
    }

```

从这里我们大致了解了KafkaConsumer的构建过程，知道了client-id是由当参数传递给KafkaConsumer，或者是有client按“consumer-序列号”规则生成。
而consumer-id是有服务端生成，其过程:

KafkaConsumer实例构建后，会向服务端发起JOIN_GROUP操作kafkaApis
```
ApiKeys.JOIN_GROUP;
```

handleJoinGroupRequest=> handleJoinGroup

```
case Some(group) =>
  group.inLock {
    if ((groupIsOverCapacity(group)
          && group.has(memberId) && !group.get(memberId).isAwaitingJoin) // oversized group, need to shed members that haven't joined yet
        || (isUnknownMember && group.size >= groupConfig.groupMaxSize)) {
      group.remove(memberId)
      responseCallback(joinError(JoinGroupRequest.UNKNOWN_MEMBER_ID, Errors.GROUP_MAX_SIZE_REACHED))
    } else if (isUnknownMember) {
      doUnknownJoinGroup(group, requireKnownMemberId, clientId, clientHost, rebalanceTimeoutMs, sessionTimeoutMs, protocolType, protocols, responseCallback)
    } else {
      doJoinGroup(group, memberId, clientId, clientHost, rebalanceTimeoutMs, sessionTimeoutMs, protocolType, protocols, responseCallback)
    }

    // attempt to complete JoinGroup
    if (group.is(PreparingRebalance)) {
      joinPurgatory.checkAndComplete(GroupKey(group.groupId))
    }
  }

```

第一次请求时服务端没有分配memberId(即consumerId),按isUnknownMember处理

```
// doUnknownJoinGroup
val newMemberId = clientId + "-" + group.generateMemberIdSuffix

```

```
def generateMemberIdSuffix = UUID.randomUUID().toString

```


### FlinkKafkaConsumer实现
Flink的通用kafka-connector部分源码:

FlinkKafkaConsumer

```java
private FlinkKafkaConsumer(
    List<String> topics,
    Pattern subscriptionPattern,
    KafkaDeserializationSchema<T> deserializer,
    Properties props) {

    super(
        topics,
        subscriptionPattern,
        deserializer,
        getLong(
            checkNotNull(props, "props"),
            KEY_PARTITION_DISCOVERY_INTERVAL_MILLIS, PARTITION_DISCOVERY_DISABLED),
        !getBoolean(props, KEY_DISABLE_METRICS, false));

    this.properties = props;
    setDeserializer(this.properties);

    // configure the polling timeout
    try {
        if (properties.containsKey(KEY_POLL_TIMEOUT)) {
            this.pollTimeout = Long.parseLong(properties.getProperty(KEY_POLL_TIMEOUT));
        } else {
            this.pollTimeout = DEFAULT_POLL_TIMEOUT;
        }
    }
    catch (Exception e) {
        throw new IllegalArgumentException("Cannot parse poll timeout for '" + KEY_POLL_TIMEOUT + '\'', e);
    }
}

```
connector拉取数据的逻辑见org.apache.flink.streaming.connectors.kafka.internal.KafkaFetcher


```java
@Override
public void runFetchLoop() throws Exception {
    try {
        final Handover handover = this.handover;

        // kick off the actual Kafka consumer
        consumerThread.start();

        while (running) {
            // this blocks until we get the next records
            // it automatically re-throws exceptions encountered in the consumer thread
            final ConsumerRecords<byte[], byte[]> records = handover.pollNext();

            // get the records for each topic partition
            for (KafkaTopicPartitionState<TopicPartition> partition : subscribedPartitionStates()) {
                                // 这里拉数据
                List<ConsumerRecord<byte[], byte[]>> partitionRecords =
                    records.records(partition.getKafkaPartitionHandle());

                for (ConsumerRecord<byte[], byte[]> record : partitionRecords) {
                    final T value = deserializer.deserialize(record);

                    if (deserializer.isEndOfStream(value)) {
                        // end of stream signaled
                        running = false;
                        break;
                    }

                    // emit the actual record. this also updates offset state atomically
                    // and deals with timestamps and watermark generation
                    emitRecord(value, partition, record.offset(), record);
                }
            }
        }
    }
    finally {
        // this signals the consumer thread that no more work is to be done
        consumerThread.shutdown();
    }

    // on a clean exit, wait for the runner thread
    try {
        consumerThread.join();
    }
    catch (InterruptedException e) {
        // may be the result of a wake-up interruption after an exception.
        // we ignore this here and only restore the interruption state
        Thread.currentThread().interrupt();
    }
}

```

org.apache.kafka.clients.consumer.ConsumerRecords从指定partition获取数据

```java
/**
 * Get just the records for the given partition
 * 从指定partition获取数据
 * @param partition The partition to get records for
 */
public List<ConsumerRecord<K, V>> records(TopicPartition partition) {
    List<ConsumerRecord<K, V>> recs = this.records.get(partition);
    if (recs == null)
        return Collections.emptyList();
    else
        return Collections.unmodifiableList(recs);
}
    

```


在flink中，会通过
```java
    final ConsumerRecords<byte[], byte[]> records = handover.pollNext();
```


```
partitionRecords =
                    records.records(partition.getKafkaPartitionHandle());

```
去拉去对应partition的数据。

那么consumer的partition如何重新分配的呢
在KafkaComsumerThreader. run的时候会分配

```java
if (newPartitions != null) {
    reassignPartitions(newPartitions);
}
```

```java
/**
     * Reestablishes the assigned partitions for the consumer. The reassigned partitions consists of
     * the provided new partitions and whatever partitions was already previously assigned to the
     * consumer.
     *
     * <p>The reassignment process is protected against wakeup calls, so that after this method
     * returns, the consumer is either untouched or completely reassigned with the correct offset
     * positions.
     *
     * <p>If the consumer was already woken-up prior to a reassignment resulting in an interruption
     * any time during the reassignment, the consumer is guaranteed to roll back as if it was
     * untouched. On the other hand, if there was an attempt to wakeup the consumer during the
     * reassignment, the wakeup call is "buffered" until the reassignment completes.
     *
     * <p>This method is exposed for testing purposes.
     */
    @VisibleForTesting
    void reassignPartitions(List<KafkaTopicPartitionState<T, TopicPartition>> newPartitions)
            throws Exception {
        if (newPartitions.size() == 0) {
            return;
        }
        hasAssignedPartitions = true;
        boolean reassignmentStarted = false;

        // since the reassignment may introduce several Kafka blocking calls that cannot be
        // interrupted,
        // the consumer needs to be isolated from external wakeup calls in setOffsetsToCommit() and
        // shutdown()
        // until the reassignment is complete.
        final KafkaConsumer<byte[], byte[]> consumerTmp;
        synchronized (consumerReassignmentLock) {
            consumerTmp = this.consumer;
            this.consumer = null;
        }

        final Map<TopicPartition, Long> oldPartitionAssignmentsToPosition = new HashMap<>();
        try {
            for (TopicPartition oldPartition : consumerTmp.assignment()) {
                oldPartitionAssignmentsToPosition.put(
                        oldPartition, consumerTmp.position(oldPartition));
            }

            final List<TopicPartition> newPartitionAssignments =
                    new ArrayList<>(
                            newPartitions.size() + oldPartitionAssignmentsToPosition.size());
            newPartitionAssignments.addAll(oldPartitionAssignmentsToPosition.keySet());
            newPartitionAssignments.addAll(convertKafkaPartitions(newPartitions));

            // reassign with the new partitions
            consumerTmp.assign(newPartitionAssignments);
            reassignmentStarted = true;

            // old partitions should be seeked to their previous position
            for (Map.Entry<TopicPartition, Long> oldPartitionToPosition :
                    oldPartitionAssignmentsToPosition.entrySet()) {
                consumerTmp.seek(
                        oldPartitionToPosition.getKey(), oldPartitionToPosition.getValue());
            }

            // offsets in the state of new partitions may still be placeholder sentinel values if we
            // are:
            //   (1) starting fresh,
            //   (2) checkpoint / savepoint state we were restored with had not completely
            //       been replaced with actual offset values yet, or
            //   (3) the partition was newly discovered after startup;
            // replace those with actual offsets, according to what the sentinel value represent.
            for (KafkaTopicPartitionState<T, TopicPartition> newPartitionState : newPartitions) {
                if (newPartitionState.getOffset()
                        == KafkaTopicPartitionStateSentinel.EARLIEST_OFFSET) {
                    consumerTmp.seekToBeginning(
                            Collections.singletonList(newPartitionState.getKafkaPartitionHandle()));
                    newPartitionState.setOffset(
                            consumerTmp.position(newPartitionState.getKafkaPartitionHandle()) - 1);
                } else if (newPartitionState.getOffset()
                        == KafkaTopicPartitionStateSentinel.LATEST_OFFSET) {
                    consumerTmp.seekToEnd(
                            Collections.singletonList(newPartitionState.getKafkaPartitionHandle()));
                    newPartitionState.setOffset(
                            consumerTmp.position(newPartitionState.getKafkaPartitionHandle()) - 1);
                } else if (newPartitionState.getOffset()
                        == KafkaTopicPartitionStateSentinel.GROUP_OFFSET) {
                    // the KafkaConsumer by default will automatically seek the consumer position
                    // to the committed group offset, so we do not need to do it.

                    newPartitionState.setOffset(
                            consumerTmp.position(newPartitionState.getKafkaPartitionHandle()) - 1);
                } else {
                    consumerTmp.seek(
                            newPartitionState.getKafkaPartitionHandle(),
                            newPartitionState.getOffset() + 1);
                }
            }
        } catch (WakeupException e) {
            // a WakeupException may be thrown if the consumer was invoked wakeup()
            // before it was isolated for the reassignment. In this case, we abort the
            // reassignment and just re-expose the original consumer.

            synchronized (consumerReassignmentLock) {
                this.consumer = consumerTmp;

                // if reassignment had already started and affected the consumer,
                // we do a full roll back so that it is as if it was left untouched
                if (reassignmentStarted) {
                    this.consumer.assign(
                            new ArrayList<>(oldPartitionAssignmentsToPosition.keySet()));

                    for (Map.Entry<TopicPartition, Long> oldPartitionToPosition :
                            oldPartitionAssignmentsToPosition.entrySet()) {
                        this.consumer.seek(
                                oldPartitionToPosition.getKey(), oldPartitionToPosition.getValue());
                    }
                }

                // no need to restore the wakeup state in this case,
                // since only the last wakeup call is effective anyways
                hasBufferedWakeup = false;

                // re-add all new partitions back to the unassigned partitions queue to be picked up
                // again
                for (KafkaTopicPartitionState<T, TopicPartition> newPartition : newPartitions) {
                    unassignedPartitionsQueue.add(newPartition);
                }

                // this signals the main fetch loop to continue through the loop
                throw new AbortedReassignmentException();
            }
        }

        // reassignment complete; expose the reassigned consumer
        synchronized (consumerReassignmentLock) {
            this.consumer = consumerTmp;

            // restore wakeup state for the consumer if necessary
            if (hasBufferedWakeup) {
                this.consumer.wakeup();
                hasBufferedWakeup = false;
            }
        }
    }
```

一句话总结
connector自己实现了FlinkKafkaConsumer,且没有按照kafka的feature实现coordinator以及JOIN_GROUOP的逻辑。消费数据，是通过将partition重新分配给consumer，直接poll，不是走coordinator逻辑。

### 总结

配置相同的group.id消费相同的topic


不管有没有开启checkPoint


两个程序相互隔离，同一条数据，两个程序都可以消费到。

差别在于消费的位置

如果配置startFromLast，都会从最新的 数据开始消费 

如果采用默认配置，第一次消费的时候从上面kafka上的offset开始消费，后面就开始各管各的。


### Reference

https://ci.apache.org/projects/flink/flink-docs-release-1.11/zh/dev/connectors/kafka.html

http://apache-flink.147419.n8.nabble.com/FlinkKafkaConsumer-td6818.html


https://blog.csdn.net/u013128262/article/details/105442182


http://apache-flink-user-mailing-list-archive.2336050.n4.nabble.com/Flink-kafka-group-question-td8185.html#none

