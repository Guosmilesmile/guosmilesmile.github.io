---
title: Flink 批处理  map 与 mapPartition
date: 2019-03-21 22:49:52
tags:
categories: Flink
---

#### map

```
采用一个数据元并生成一个数据元。
data.map(new MapFunction<String, Integer>() {
  public Integer map(String value) { return Integer.parseInt(value); }
});

```

#### MapPartition

```
在单个函数调用中转换并行分区。该函数将分区作为Iterable流来获取，并且可以生成任意数量的结果值。每个分区中的数据元数量取决于并行度和先前的 算子操作。

data.mapPartition(new MapPartitionFunction<String, Long>() {
  public void mapPartition(Iterable<String> values, Collector<Long> out) {
    long c = 0;
    for (String s : values) {
      c++;
    }
    out.collect(c);
  }
});
```

### 区别
* map是一条一条数据进行处理，体现出来函数的入参是string，mapParition是将这条条数据变成一批处理，体现在入参是Iterable<String> values。
 

mapPartition：是一个分区一个分区拿出来的
好处就是以后我们操作完数据了需要存储到mysql中，这样做的好处就是几个分区拿几个连接，如果用map的话，就是多少条数据拿多少个mysql的连接