---
title: Flink 流处理简单引导
date: 2019-03-17 12:52:10
tags:
---



### 示例程序

<!--more-->
```java
import org.apache.flink.api.common.functions.FlatMapFunction;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.windowing.time.Time;
import org.apache.flink.util.Collector;

public class WindowWordCount {

    public static void main(String[] args) throws Exception {

        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        DataStream<Tuple2<String, Integer>> dataStream = env
                .socketTextStream("localhost", 9999)
                .flatMap(new Splitter())
                .keyBy(0)
                .timeWindow(Time.seconds(5))
                .sum(1);

        dataStream.print();

        env.execute("Window WordCount");
    }

    public static class Splitter implements FlatMapFunction<String, Tuple2<String, Integer>> {
        @Override
        public void flatMap(String sentence, Collector<Tuple2<String, Integer>> out) throws Exception {
            for (String word: sentence.split(" ")) {
                out.collect(new Tuple2<String, Integer>(word, 1));
            }
        }
    }

}
```

### DataSource

从StreamExecutionEnvironment可以获取的预定的source：   
基于文件的(File-based)：
* readTextFile(path)
* readFile(fileInputFormat, path)
* readFile(fileInputFormat, path, watchType, interval, pathFilter, typeInfo) 
基于Socket
* socketTextStream

基于集合
* Collection-based 
* fromCollection(Iterator, Class)
* fromElements(T ...)
* fromParallelCollection(SplittableIterator, Class) 并行的从迭代器中获取数据。
* generateSequence(from, to) 从form到to并行的生成一系列间隔一定的数

自定义
* addSource 添加一个新的数据源函数。例如：addSource(new FlinkKafkaConsumer08<>(...)).


### Data Sink

* writeAsText()
* writeAsCsv(...)
* print() / printToErr()
* writeUsingOutputFormat()
* writeToSocket
* addSink
* writeUsingOutputFormat() 

以上write*()方法都是内部debug使用，都不在flink的checkpointing的实践中。

### Controlling Latency（控制延迟）
元素不会一条条在网络上传输，而是会缓冲一部分。缓冲的大小可以在Flink 的配置文件中调整。虽然这种方法很好地优化了吞吐量，但是当传入的流不够快时，它会导致延迟问题。为了控制吞吐和牙齿，可以通过env.setBufferTimeout(timeoutMillis)设置一个最大的等待时间。超过这个时间，缓冲会马上下发而不是等到缓冲队列满。默认的时间是100ms。

```java
LocalStreamEnvironment env = StreamExecutionEnvironment.createLocalEnvironment();
env.setBufferTimeout(timeoutMillis);

env.generateSequence(1,10).map(new MyMapper()).setBufferTimeout(timeoutMillis);
```

如果是为-1，那么缓存没有满，数据是不会下发的。如果想要缩小延迟，可以将这个时间设置为一个接近0（5或者10ms）。应该避免缓冲区超时为0，因为这会导致严重的性能下降。

### 调试

##### Local Execution Environment
创建本地环境，打上断点

```java
final StreamExecutionEnvironment env = StreamExecutionEnvironment.createLocalEnvironment();

DataStream<String> lines = env.addSource(/* some source */);
// build your program

env.execute();
```

#### Collection Data Sources
可以使用java的集合数据充当数据源方便调试
```java
final StreamExecutionEnvironment env = StreamExecutionEnvironment.createLocalEnvironment();

// Create a DataStream from a list of elements
DataStream<Integer> myInts = env.fromElements(1, 2, 3, 4, 5);

// Create a DataStream from any Java collection
List<Tuple2<String, Integer>> data = ...
DataStream<Tuple2<String, Integer>> myTuples = env.fromCollection(data);

// Create a DataStream from an Iterator
Iterator<Long> longIt = ...
DataStream<Long> myLongs = env.fromCollection(longIt, Long.class);
```

注意：目前，集合数据源要求数据类型和迭代器实现可序列化。此外，收集数据源不能并行执行

#### Iterator Data Sink

```java
import org.apache.flink.streaming.experimental.DataStreamUtils

DataStream<Tuple2<String, Integer>> myResult = ...
Iterator<Tuple2<String, Integer>> myOutput = DataStreamUtils.collect(myResult)
```