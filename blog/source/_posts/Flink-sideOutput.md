---
title: Flink sideOutput 侧输出
date: 2019-03-17 13:17:42
tags:
categories:
    - 流式计算 
    - Flink
---

除了从DataStream操作的结果中获取主数据流之外，你还可以产生任意数量额外的侧输出结果流。侧输出结果流的数据类型不需要与主数据流的类型一致，不同侧输出流的类型也可以不同。当您想要拆分数据流时(通常必须复制流),然后从每个流过滤出您不想拥有的数据，此操作将非常有用。
当使用侧输出流时，你首先得定义一个OutputTag，这个OutputTag将用来标识一个侧输出流:

```
// this needs to be an anonymous inner class, so that we can analyze the type
OutputTag<String> outputTag = new OutputTag<String>("side-output") {};
```

可以通过以下函数将数据发送到旁路输出：

* ProcessFunction
* CoProcessFunction
* ProcessWindowFunction
* ProcessAllWindowFunction

注意，OutputTag是根据侧输出流所包含的元素的类型来输入的。
数据发送到侧输出流只能从一个ProcessFunction中发出，你可以使用Context参数来发送数据到一个通过OutputTag标记的侧输出流中:
```
DataStream<Integer> input = ...;

final OutputTag<String> outputTag = new OutputTag<String>("side-output"){};

SingleOutputStreamOperator<Integer> mainDataStream = input
  .process(new ProcessFunction<Integer, Integer>() {

      @Override
      public void processElement(
          Integer value,
          Context ctx,
          Collector<Integer> out) throws Exception {
        // 将数据发送到常规输出中
        out.collect(value);

        // 将数据发送到侧输出中
        ctx.output(outputTag, "sideout-" + String.valueOf(value));
      }
    });
```

你可以在DataStream操作的结果中使用getSideOutput(OutputTag)来获取侧输出，这里为您提供一个DataStream类型，用于输出端输出流的结果：
```
final OutputTag<String> outputTag = new OutputTag<String>("side-output"){};

SingleOutputStreamOperator<Integer> mainDataStream = ...;

DataStream<String> sideOutputStream = mainDataStream.getSideOutput(outputTag);
```

```
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        InputStream in = FlinkSideOutPut.class.getResourceAsStream("/flink_content.properties");
        final ParameterTool parameterTool = ParameterTool.fromPropertiesFile(in);
        int sourceParam = parameterTool.getInt("source.param");
        int lineParam = parameterTool.getInt("line.param");
        FlinkKafkaConsumer08 flinkKafkaConsumer08 = new PlayerCountSource().init();
        DataStream<String> input = env.addSource(flinkKafkaConsumer08).setParallelism(sourceParam);
        DataStream<PlayerCountEvent> playerCountEventDataStream = input.flatMap(new String2PlayerCountTransformation()).setParallelism(lineParam);
        final OutputTag<PlayerCountEvent> outputTag = new OutputTag<PlayerCountEvent>("side-output") {
        };
        SingleOutputStreamOperator<PlayerCountEvent> mainDataStream = playerCountEventDataStream
                .process(new ProcessFunction<PlayerCountEvent, PlayerCountEvent>() {

                    //可以在此进行分流，次数把在线人数大于和小于10分成两条流
                    @Override
                    public void processElement(PlayerCountEvent playerCountEvent, Context context, Collector<PlayerCountEvent> collector) throws Exception {
                        if (playerCountEvent.getCount() > 10) {
                            collector.collect(playerCountEvent);
                        } else {
                            // emit data to side output
                            context.output(outputTag, playerCountEvent);
                        }


                    }
                });

        DataStream<PlayerCountEvent> sideOutputStream = mainDataStream.getSideOutput(outputTag);
        sideOutputStream.print();
        env.execute();
```
### 作用
~~目前觉得使用旁路输出可以将一条流分为多条流，并且是在一次算子中执行，而不是通过两次fliter，使用两次fliter会降低效率并且过滤多次原始数据。~~

#### 目前来看，将一条流分成多条流，可以使用旁路输出，但是直接使用Split和select一起使用，旁路输出的使用是用来获取event time的时候，获取延迟的数据，而不是直接丢弃。


demo：
```java
final OutputTag<T> lateOutputTag = new OutputTag<T>("late-data"){};

DataStream<T> input = ...;

SingleOutputStreamOperator<T> result = input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .sideOutputLateData(lateOutputTag)
    .<windowed transformation>(<window function>);

DataStream<T> lateStream = result.getSideOutput(lateOutputTag);

```

#### Reference
https://ci.apache.org/projects/flink/flink-docs-release-1.7/dev/stream/operators/windows.html#getting-late-data-as-a-side-output

https://blog.csdn.net/rlnLo2pNEfx9c/article/details/86285634