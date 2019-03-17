---
title: Flink 的时间类型
date: 2019-03-17 13:04:56
tags:
categories: Flink
---

### Event Time / Processing Time / Ingestion Time

<!--more-->
* Processing time(处理时间): 机器执行相应操作时的系统时间。     
当流程序在处理时间上运行时，所有基于时间的操作(比如时间窗口)都将使用运行各自操作的机器的系统时钟。每小时处理时间窗口将包含在系统时钟指示整个小时之间到达某个特定操作的所有记录。举例：如果一个应用在9.15分开始执行，第一个小时处理时间窗口将包含在上午9:15到10:00之间处理的事件，第一个小时处理时间窗口将包含在上午9:15到10:00之间处理的事件。     
处理时间提供最好的处理性能和最低的延迟，但是在分布式环境和异步环境下这个时间会不准。

* Event time（事件时间）：事件时间是指每个事件在其生产设备上发生的时间（源头标定的时间）。这个时间一般在记录进入Flink前就嵌入记录中，这个时间可以从每条记录中提取出来。事件时间程序必须指定如何生成事件时间的水位线，这个水位线是指示事件何时进行的信号。  
不管事件什么时候到达，或者它们的顺序，事件时间处理将产生完全一致和确定性的结果。但是，除非已知事件是按顺序到达的(通过时间戳)，否则事件时间处理将在等待无序事件导致一些延迟。由于只能等待有限的时间，这就对确定性事件时间应用程序的限制。   
假设所有数据都已到达，事件时间操作将按照预期执行，即使在处理无序或延迟事件或重新处理历史数据时也会产生正确且一致的结果。例如，每小时事件时间窗口将包含所有包含属于该小时的事件时间戳的记录，而不管它们到达的顺序如何，也不管它们在什么时候被处理

Ingestion time（导入时间）:摄入时间是导入Flink的时间，在Source获取每一条记录的时候，source的当前时间为这条记录的导入时间。  
导入时间概念上介于事件时间和处理时间之间。          
![image](https://ci.apache.org/projects/flink/flink-docs-release-1.6/fig/times_clocks.svg)

#### Setting a Time Characteristic（设置时间特性）
Flink DataStream程序的第一部分通常设置基本时间特征。该设置定义了数据流源的行为(例如，它们是否会分配时间戳)，以及像KeyedStream.timeWindow(time .seconds(30))这样的窗口操作应该使用什么时间概念。下面的例子定义了一小时的时间窗口。
```java
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

env.setStreamTimeCharacteristic(TimeCharacteristic.ProcessingTime);

// alternatively:
// env.setStreamTimeCharacteristic(TimeCharacteristic.IngestionTime);
// env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);

DataStream<MyEvent> stream = env.addSource(new FlinkKafkaConsumer09<MyEvent>(topic, schema, props));

stream
    .keyBy( (event) -> event.getUser() )
    .timeWindow(Time.hours(1))
    .reduce( (a, b) -> a.add(b) )
    .addSink(...);
```
**注意，为了使用事件时间，程序需要使用直接标记事件时间和提交水位线的source，或者程序必须在源文件之后注入Timestamp Assigner & Watermark Generator。这些函数描述了如何访问事件时间戳，以及事件流显示的异常程度有多严重**

### Event Time and Watermarks
支持事件时间的流处理器需要一种方法来度量事件时间的进度。例如，当事件时间超过一小时后，构建每小时窗口的窗口操作符需要得到通知，以便操作符可以在程序中关闭窗口。

Flink中度量事件时间进度的机制是水位线。A Watermark(t) 表明事件时间已经到达流中的时间t。下图显示了具有(逻辑)时间戳和水位线的事件流。在本例中，事件是按顺序排列的(相对于它们的时间戳)，这意味着水位线只是流中的周期性标记。     
![image](https://ci.apache.org/projects/flink/flink-docs-release-1.6/fig/stream_watermark_in_order.svg)

水位线对于无序的流是至关重要的，如下图所示，事件不是按照它们的时间戳来排序的。通常，水位线是一种声明，在流中的那个点之前，所有事件直到某个时间戳都应该到达。一旦水位线到达操作符，操作符可以将其内部事件时钟提前到水位线的值。（就是说，如果水位高告诉我们到了11，说明11之前的数据都已经到了，可以开始11前的计算，如果有比11小的数据到达，会跳过这个计算。）  
![image](https://ci.apache.org/projects/flink/flink-docs-release-1.6/fig/stream_watermark_out_of_order.svg)

#### Watermarks in Parallel Streams（并行流中的水位线）
水位线是在源函数处或直接在源函数之后生成的。源函数的每个并行子任务通常独立地生成其水位线。   
当水位线通过流媒体程序时，它们在到达的操作符处提前了事件时间。当操作符提前其事件时间时，它将为其后续操作符在下游生成一个新的水印。

![image](https://ci.apache.org/projects/flink/flink-docs-release-1.6/fig/parallel_streams_watermarks.svg)

## Generating Timestamps / Watermarks

首先想要设定时间事件，必须设定事件时间
```java
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
```

#### Assigning Timestamps（指定时间戳）

时间戳的指定和水位线的创建是一起的，可以通过两种方法赋予时间戳并且创建水位线
* 在数据源头指定
* 创建一个时间戳分配函数和水位线生成函数

#### Source Functions with Timestamps and Watermarks（源函数中的时间戳和水位线）

```java
@Override
public void run(SourceContext<MyType> ctx) throws Exception {
	while (/* condition */) {
		MyType next = getNext();
		ctx.collectWithTimestamp(next, next.getEventTimestamp());

		if (next.hasWatermarkTime()) {
			ctx.emitWatermark(new Watermark(next.getWatermarkTime()));
		}
	}
}
```


### Timestamp Assigners / Watermark Generators(时间戳指派/水位线生成)
时间戳指派函数会获取一条流，生成一条新流带有时间戳和水位线，如果原始流已经拥有时间戳或者水位线，那么时间戳指派函数会覆盖他们。

不管在任何场景，时间戳指派必须在第一个操作前指定，

