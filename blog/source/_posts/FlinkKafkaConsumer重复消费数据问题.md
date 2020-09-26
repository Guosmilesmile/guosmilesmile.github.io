---
title: FlinkKafkaConsumer重复消费数据问题
date: 2020-09-07 15:36:35
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