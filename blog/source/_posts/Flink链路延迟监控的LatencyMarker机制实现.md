---
title: Flink链路延迟监控的LatencyMarker机制实现
date: 2020-06-10 22:09:43
tags:
categories:
	- Flink
---


## 背景

对于实时的流式处理系统来说，我们需要关注数据输入、计算和输出的及时性，所以处理延迟是一个比较重要的监控指标，特别是在数据量大或者软硬件条件不佳的环境下。Flink早在FLINK-3660（https://issues.apache.org/jira/browse/FLINK-3660 ） 就为用户提供了开箱即用的链路延迟监控功能，只需要配置好metrics.latency.interval参数，再观察TaskManagerJobMetricGroup/operator_id/operator_subtask_index/latency这个metric即可。本文简单walk一下源码，看看它是如何实现的，并且简要说明注意事项。


### LatencyMarker的产生
与通过水印来标记事件时间的推进进度相似，Flink也用一种特殊的流元素（StreamElement）作为延迟的标记，称为LatencyMarker。


![image](https://note.youdao.com/yws/api/personal/file/E50EBE26D6E74AB99F7AB9F5B3A396C4?method=download&shareKey=50155054038914b6361e579b997a6b9e)

LatencyMarker的数据结构甚简单，只有3个field，即它被创建时携带的时间戳、算子ID和算子并发实例（sub-task）的ID。

```java
public final class LatencyMarker extends StreamElement {


	/** The time the latency mark is denoting. */
	private final long markedTime;

	private final OperatorID operatorId;

	private final int subtaskIndex;
}

```

LatencyMarker和水印不同，不需要通过用户抽取产生，而是在Source端自动按照metrics.latency.interval参数指定的周期生成。StreamSource专门实现了一个内部类LatencyMarksEmitter用来发射LatencyMarker，而它又借用了负责协调处理时间的服务ProcessingTimeService，如下代码所示。

```java

	LatencyMarksEmitter<OUT> latencyEmitter = null;
		if (latencyTrackingInterval > 0) {
			latencyEmitter = new LatencyMarksEmitter<>(
				getProcessingTimeService(),
				collector,
				latencyTrackingInterval,
				this.getOperatorID(),
				getRuntimeContext().getIndexOfThisSubtask());
		}
		
		
private static class LatencyMarksEmitter<OUT> {
		private final ScheduledFuture<?> latencyMarkTimer;

		public LatencyMarksEmitter(
				final ProcessingTimeService processingTimeService,
				final Output<StreamRecord<OUT>> output,
				long latencyTrackingInterval,
				final OperatorID operatorId,
				final int subtaskIndex) {

			latencyMarkTimer = processingTimeService.scheduleAtFixedRate(
				new ProcessingTimeCallback() {
					@Override
					public void onProcessingTime(long timestamp) throws Exception {
						try {
							// ProcessingTimeService callbacks are executed under the checkpointing lock
							output.emitLatencyMarker(new LatencyMarker(processingTimeService.getCurrentProcessingTime(), operatorId, subtaskIndex));
						} catch (Throwable t) {
							// we catch the Throwables here so that we don't trigger the processing
							// timer services async exception handler
							LOG.warn("Error while emitting latency marker.", t);
						}
					}
				},
				0L,
				latencyTrackingInterval);
		}

```

AbstractStreamOperator是所有Flink Streaming算子的基类，在它的初始化方法setup()中，会先创建用于延迟统计的LatencyStats实例。


```java
final String configuredGranularity = taskManagerConfig.getString(MetricOptions.LATENCY_SOURCE_GRANULARITY);
LatencyStats.Granularity granularity;
try {
    granularity = LatencyStats.Granularity.valueOf(configuredGranularity.toUpperCase(Locale.ROOT));
} catch (IllegalArgumentException iae) {
    granularity = LatencyStats.Granularity.OPERATOR;
    LOG.warn(
        "Configured value {} option for {} is invalid. Defaulting to {}.",
        configuredGranularity,
        MetricOptions.LATENCY_SOURCE_GRANULARITY.key(),
        granularity);
}
TaskManagerJobMetricGroup jobMetricGroup = this.metrics.parent().parent();
this.latencyStats = new LatencyStats(jobMetricGroup.addGroup("latency"),
    historySize,
    container.getIndexInSubtaskGroup(),
    getOperatorID(),
    granularity);
```

LatencyStats中的延迟最终会转化为直方图表示，通过直方图就可以统计出延时的最大值、最小值、均值、分位值（quantile）等指标。以下是reportLatency()方法的源码。

```java
public void reportLatency(LatencyMarker marker) {
		final String uniqueName = granularity.createUniqueHistogramName(marker, operatorId, subtaskIndex);

		DescriptiveStatisticsHistogram latencyHistogram = this.latencyStats.get(uniqueName);
		if (latencyHistogram == null) {
			latencyHistogram = new DescriptiveStatisticsHistogram(this.historySize);
			this.latencyStats.put(uniqueName, latencyHistogram);
			granularity.createSourceMetricGroups(metricGroup, marker, operatorId, subtaskIndex)
				.addGroup("operator_id", String.valueOf(operatorId))
				.addGroup("operator_subtask_index", String.valueOf(subtaskIndex))
				.histogram("latency", latencyHistogram);
		}

		long now = System.currentTimeMillis();
		latencyHistogram.update(now - marker.getMarkedTime());
	}
```

延迟是由当前时间戳减去LatencyMarker携带的时间戳得到的，所以在Sink端统计到的就是全链路延迟了。


整体流程如下

![image](https://note.youdao.com/yws/api/personal/file/671527AB61A24CDF8087512BA3910FDD?method=download&shareKey=d457dbf531b118a26cb431139199ddb0)



### 注意事项

1. LatencyMarker不参与window、MiniBatch的缓存计时，直接被中间Operator下发
2. Metric路径:TaskManagerJobMetricGroup/operator_id/operator_subtask_index/latency
3. 每个中间Operator、以及Sink都会统计自己与Source节点的链路延迟，我们在监控页面，一般展示Source至Sink链路延迟
4. 延迟粒度细分到Task，可以用来排查哪台机器的Task时延偏高，进行对比和运维排查
5. 从实现原理来看，发送时延标记间隔配置大一些（例如20秒一次），一般不会影响系统处理业务数据的性能

### Reference

https://blog.csdn.net/nazeniwaresakini/article/details/106615777


https://cloud.tencent.com/developer/article/1549048

