---
title: Flink1.11升级填坑
date: 2020-07-23 19:55:50
tags:
categories:
	- Flink
---

## 背景

现有集群版本是Flink 1.10.1，想要升级到社区最新的版本Flink 1.11.1.





## 踩坑过程



### No hostname could be resolved for ip address



详细的社区邮件讨论过程如下：

http://apache-flink.147419.n8.nabble.com/Flink-1-11-submit-job-timed-out-td4982.html



在提交作业的时候，JM会疯狂刷出大量的日志No hostname could be resolved for ip address xxxx。该xxxx ip是kubernetes分配给flink TM的内网ip，JM由于这个报错，直接time out。

```SHELL
kubectl run -i -t busybox --image=busybox --restart=Never
```



进入到pod中反向解析flink TM的ip失败。



```SHELL
/ # nslookup 10.47.96.2
Server:		10.96.0.10
Address:	10.96.0.10:53

** server can't find 2.96.47.10.in-addr.arpa: NXDOMAIN

```





而解析JM居然可以成功



```shell
/ # nslookup 10.34.128.8
Server:		10.96.0.10
Address:	10.96.0.10:53

8.128.34.10.in-addr.arpa	name = 10-34-128-8.flink-jobmanager.flink-test.svc.cluster.local

```



唯一的差别就是JM是有service。



通过添加社区提供的可选配置解决问题taskmanager-query-state-service.yaml。



https://ci.apache.org/projects/flink/flink-docs-release-1.11/ops/deployment/kubernetes.html



不过目前跟社区的沟通中，社区是没有遇到这个问题的，该问题还在进一步讨论中。





### 新版本waterMark改动



新版的waterMark的生成改为



```java
@Public
public interface WatermarkGenerator<T> {

	/**
	 * Called for every event, allows the watermark generator to examine and remember the
	 * event timestamps, or to emit a watermark based on the event itself.
	 */
	void onEvent(T event, long eventTimestamp, WatermarkOutput output);

	/**
	 * Called periodically, and might emit a new watermark, or not.
	 *
	 * <p>The interval in which this method is called and Watermarks are generated
	 * depends on {@link ExecutionConfig#getAutoWatermarkInterval()}.
	 */
	void onPeriodicEmit(WatermarkOutput output);
}

```

使用方式改为：



```java
dataStream.assignTimestampsAndWatermarks(WatermarkStrategy.forBoundedOutOfOrderness(Duration.ofSeconds(3)));
```



跟旧版本的相比extractTimestamp提取时间戳的操作不见了。



```java
public class BoundedOutOfOrdernessGenerator implements AssignerWithPeriodicWatermarks<MyEvent> {

    private final long maxOutOfOrderness = 3500; // 3.5 seconds

    private long currentMaxTimestamp;

    @Override
    public long extractTimestamp(MyEvent element, long previousElementTimestamp) {
        long timestamp = element.getCreationTime();
        currentMaxTimestamp = Math.max(timestamp, currentMaxTimestamp);
        return timestamp;
    }

    @Override
    public Watermark getCurrentWatermark() {
        // return the watermark as current highest timestamp minus the out-of-orderness bound
        return new Watermark(currentMaxTimestamp - maxOutOfOrderness);
    }
}
```





如果按照新版的升级，那么数据的timeStamp会变成Long.Min。正确的使用方式是



```java
dataStream.assignTimestampsAndWatermarks(
				WatermarkStrategy
						.<Tuple2<String,Long>>forBoundedOutOfOrderness(Duration.ofSeconds(5))
						.withTimestampAssigner((event, timestamp)->event.f1));

```



```java
.assignTimestampsAndWatermarks(WatermarkStrategy.<StationLog>forBoundedOutOfOrderness(Duration.ofSeconds(3))
				.withTimestampAssigner(new SerializableTimestampAssigner<StationLog>() {
					@Override
					public long extractTimestamp(StationLog element, long recordTimestamp) {
						return element.getCallTime(); //指定EventTime对应的字段
					}
				})
```



如果有自定义，使用方式如下



```java
.assignTimestampsAndWatermarks(((WatermarkStrategy)(ctx)->new BoundOutOrdernessStrategy(60,60)
				.withTimestampAssigner(new SerializableTimestampAssigner<StationLog>() {
					@Override
					public long extractTimestamp(StationLog element, long recordTimestamp) {
						return element.getCallTime(); //指定EventTime对应的字段
					}
				})
```







### flink1.11，idea运行失败



社区讨论见



http://apache-flink.147419.n8.nabble.com/flink1-11-idea-td4576.html



作业的依赖从1.10.1升级到1.11.0，在idea运行的时候报错



```java
Exception in thread "main" java.lang.IllegalStateException: No ExecutorFactory found to execute the application.
   at org.apache.flink.core.execution.DefaultExecutorServiceLoader.getExecutorFactory(DefaultExecutorServiceLoader.java:84)
   at org.apache.flink.streaming.api.environment.StreamExecutionEnvironment.executeAsync(StreamExecutionEnvironment.java:1803)
   at org.apache.flink.streaming.api.environment.StreamExecutionEnvironment.execute(StreamExecutionEnvironment.java:1713)
   at org.apache.flink.streaming.api.environment.LocalStreamEnvironment.execute(LocalStreamEnvironment（）
```





解决方法：



尝试加一下这个依赖
groupId: org.apache.flink
artifactId: flink-clients_${scala.binary.version}



导致原因



https://ci.apache.org/projects/flink/flink-docs-master/release-notes/flink-1.11.html#reversed-dependency-from-flink-streaming-java-to-flink-client-flink-15090





