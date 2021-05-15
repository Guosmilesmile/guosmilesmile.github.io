---
title: 关于Flink消费kafka开启checkpoint却没有配置重启策略的思考
date: 2021-05-15 15:41:29
tags:
categories: Flink
---



### 背景

最近交接了一个flink项目，这个项目有点神奇，我也想了好一会才明白。
这个flink程序，部署在yarn上。有如下几个配置


1. 开启了checkpoint
2. restart-strategy：none
3. setCommitOffsetsOnCheckpoints(true)
4. 停止脚本是一个pyhton脚本，在master调用kill -15 



### problem

1. 没有配置重启策略，如何利用到了checkpoint，重启程序理论是会有数据断层或者突刺的
2. kill居然可以把整个作业kill了，还把集群也关了
3. 没有配置重启策略，那么作业挂了怎么办


### 解答

#### 问题1
开启了checkpoint，并且设置了exactly-once，如果setCommitOffsetsOnCheckpoints设置为false。那么offset不会提交到kafka的，如果程序重启就会根据配置的Start Position Configuration开始消费

可以参考：
https://ci.apache.org/projects/flink/flink-docs-release-1.13/docs/connectors/datastream/kafka/

```
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

FlinkKafkaConsumer<String> myConsumer = new FlinkKafkaConsumer<>(...);
myConsumer.setStartFromEarliest();     // start from the earliest record possible
myConsumer.setStartFromLatest();       // start from the latest record
myConsumer.setStartFromTimestamp(...); // start from specified epoch timestamp (milliseconds)
myConsumer.setStartFromGroupOffsets(); // the default behaviour

DataStream<String> stream = env.addSource(myConsumer);
```

那这么说来，应该有突刺才对，但是如果配置了setCommitOffsetsOnCheckpoints(true)，就会在checkpoint结束后提交offset。如果作业挂了重启还是可以从checkpoint处开始消费的，就不会出现断层或者突刺


#### 问题2


为什么直接kill这个进程就可以kill整个作业呢？首先，没有配置重启参数，可是发现哪怕注释掉restart-strategy：none也可以kill。。

发现原来启动参数启动了 flink run -sea。。。

用的不是-d。 -sae

我们来看看-sae的解释

```
-sae,–shutdownOnAttachedExit : 如果是前台的方式提交，当客户端中断，集群执行的job任务也会shutdown。
```

其实脚本kill的是这个client进程。（说明作业在run，client保持这长连接。。。)


#### 问题3


cron中配置了监控这个client进行是否存在的任务，如果挂了就拉起



### 结论

看到这几个操作，有点野，真的是各种操作来实现flink自带的功能，曲线救国。

作业恢复直接配置重启策略即可，提交通过-d即可。作业的ha和失败，应该通过flink自带的功能来实现，而不是通过脚本来额外增加负担。



