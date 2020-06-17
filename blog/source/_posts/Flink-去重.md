---
title: Flink 去重
date: 2020-06-15 20:55:07
tags:
categories:
	- Flink
---
## 背景



数据去重（data deduplication）是我们大数据攻城狮司空见惯的问题了。除了统计UV等传统用法之外，去重的意义更在于消除不可靠数据源产生的脏数据——即重复上报数据或重复投递数据的影响，使流式计算产生的结果更加准确。



以一个实际场景为例：计算每个广告每小时的点击用户数，广告点击日志包含：广告位ID、用户设备ID、点击时间。



### 实现步骤分析：

1. 为了当天的数据可重现，这里选择事件时间也就是广告点击时间作为每小时的窗口期划分
2. 数据分组使用广告位ID+点击事件所属的小时
3. 选择processFunction来实现，一个状态用来保存数据、另外一个状态用来保存对应的数据量
4. 计算完成之后的数据清理，按照时间进度注册定时器清理



### 数据准备



```java
public class AdData {

	private int id;
	private String devId;
	private long time;
}

```



主流程 



```java
StreamExecutionEnvironment env = StreamContextEnvironment.getExecutionEnvironment();


DataStreamSource<AdData> dataStreamSource = env.fromCollection(null);

DataStream<AdData> dedupStream = dataStreamSource.map(item -> item).assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor<AdData>(Time.minutes(1)) {

   @Override
   public long extractTimestamp(AdData element) {
      return element.getTime();
   }
})	}).keyBy(item -> Tuple2.of(TimeWindow.getWindowStartWithOffset(item.getTime(), 0,
			Time.hours(1).toMilliseconds()) + Time.hours(1).toMilliseconds(), item.getId()))
			.process(new RocksDbDeduplicateProcessFunc());


env.execute();
```

### 朴素方法论



HashSet去重



### 状态后台去重



```java
public class RocksDbDeduplicateProcessFunc extends KeyedProcessFunction<Tuple2<Long, Integer>, AdData, AdData> {

   private MapState<String, Integer> devIdState;
   private MapStateDescriptor<String, Integer> devIdStateDesc;
   private ValueState<Long> countState;
   private ValueStateDescriptor<Long> countStateDesc;

   @Override
   public void open(Configuration parameters) throws Exception {
      super.open(parameters);
      devIdStateDesc = new MapStateDescriptor("devIdState", TypeInformation.of(String.class), TypeInformation.of(Integer.class));
      devIdState = getRuntimeContext().getMapState(devIdStateDesc);
      countStateDesc = new ValueStateDescriptor("countState", TypeInformation.of(Long.class));
      countState = getRuntimeContext().getState(countStateDesc);
   }

   @Override
   public void processElement(AdData value, Context ctx, Collector<AdData> out) throws Exception {
      long currW = ctx.timerService().currentWatermark();
      if (ctx.getCurrentKey().f0 + 1 <= currW) {
         System.out.println("late data:" + value);
         return;
      }
      String devId = value.getDevId();
      Integer match = devIdState.get(devId);
      if (!devIdState.contains(devId)) {
         //表示不存在
         devIdState.put(devId, 1);
         countState.update(countState.value() + 1);
         //还需要注册一个定时器
         ctx.timerService().registerEventTimeTimer(ctx.getCurrentKey().f0 + 1);
      }
   }

   // 数据清理通过注册定时器方式ctx.timerService().registerEventTimeTimer(ctx.getCurrentKey.time + 1)表示当watermark大于该小时结束时间+1就会执行清理动作，调用onTimer方法。
   @Override
   public void onTimer(long timestamp, OnTimerContext ctx, Collector<AdData> out) throws Exception {
      super.onTimer(timestamp, ctx, out);
      devIdState.clear();
      countState.clear();
   }
}
```



主要是通过状态后台去重，tuple2.f0对应的是规整后的小时时间，当 watermark > tuple2.fo的时候，为下一个小时的数据到来，清理状态数据。如果担心混了，可以id+time作为mapState的key.





### SQL方式



select  HOUR(TUMBLE_START(ts, INTERVAL '1' HOUR)),count(distinct devId)  from pv group by id TUMBLE(time, INTERVAL '1' HOUR)

uv 的统计我们通过内置的 `COUNT(DISTINCT user_id)`来完成，Flink SQL 内部对 COUNT DISTINCT 做了非常多的优化，因此可以放心使用。

这里我们使用 `HOUR` 内置函数，从一个 TIMESTAMP 列中提取出一天中第几个小时的值。

https://ci.apache.org/projects/flink/flink-docs-release-1.10/dev/table/tuning/streaming_aggregation_optimization.html

```java
table.optimizer.distinct-agg.split.enabled=true
```



如果需求是统计每日网站的uv值：



1. SELECT datatime,count(DISTINCT devId) FROM pv group by datatime
2. 
   select count(*),datatime from(select distinct devId,datatime from pv ) a group by datatime



两种方式对比:

这两种方式最终都能得到相同的结果，但是经过分析其在内部实现上差异还是比较大，第一种在分组上选择datatime ，内部使用的累加器DistinctAccumulator 每一个datatime都会与之对应一个对象，在该维度上所有的设备id, 都会存储在该累加器对象的map中，而第二种选择首先细化分组，使用datatime+devId分开存储，然后外部使用时间维度进行计数，简单归纳就是：
第一种: datatime->Value{devI1,devId2..}
第二种: datatime+devId->row(0)
聚合函数中accumulator 是存储在ValueState中的，第二种方式的key会比第一种方式数量上多很多，但是其ValueState占用空间却小很多，而在实际中我们通常会选择Rocksdb方式作为状态后端，rocksdb中value大小是有上限的，第一种方式很容易到达上限，那么使用第二种方式会更加合适；(2*32上限)



### 布隆过滤器去重



```java
public class BloomFilterDeduplicateProcessFunc extends KeyedProcessFunction<Tuple2<Long, Integer>, AdData, AdData> {

   private static final int BF_CARDINAL_THRESHOLD = 1000000;
   private static final double BF_FALSE_POSITIVE_RATE = 0.01;

   private volatile BloomFilter<String> subOrderFilter;

   private ValueState<Long> countState;
   private ValueStateDescriptor<Long> countStateDesc;


   @Override
   public void open(Configuration parameters) throws Exception {
      super.open(parameters);
      subOrderFilter = BloomFilter.create(Funnels.stringFunnel(Charset.defaultCharset()), BF_CARDINAL_THRESHOLD, BF_FALSE_POSITIVE_RATE);
      countStateDesc = new ValueStateDescriptor("countState", TypeInformation.of(Long.class));
      countState = getRuntimeContext().getState(countStateDesc);

   }

   @Override
   public void processElement(AdData value, Context ctx, Collector<AdData> out) throws Exception {
      long currW = ctx.timerService().currentWatermark();
      if (ctx.getCurrentKey().f0 + 1 <= currW) {
         System.out.println("late data:" + value);
         return;
      }
      String devId = value.getDevId();
      if (!subOrderFilter.mightContain(devId)) {
         //表示不存在
         subOrderFilter.put(devId);
         countState.update(countState.value() + 1);
         //还需要注册一个定时器
         ctx.timerService().registerEventTimeTimer(ctx.getCurrentKey().f0 + 1);
      }
   }

   /**
    * 数据清理通过注册定时器方式ctx.timerService().registerEventTimeTimer(ctx.getCurrentKey.time + 1)表示当watermark大于该小时结束时间+1就会执行清理动作，调用onTimer方法。
    */
   @Override
   public void onTimer(long timestamp, OnTimerContext ctx, Collector<AdData> out) throws Exception {
      super.onTimer(timestamp, ctx, out);
      subOrderFilter = BloomFilter.create(Funnels.stringFunnel(Charset.defaultCharset()), BF_CARDINAL_THRESHOLD, BF_FALSE_POSITIVE_RATE);
      countState.clear();
   }
}
```



每个分组内创建布隆过滤器。布隆过滤器的期望最大数据量应该按每天产生子订单最多的那个站点来设置，这里设为100万，并且可容忍的误判率为1%。根据上面科普文中的讲解，单个布隆过滤器需要8个哈希函数，其位图占用内存约114MB，压力不大。



每当一条数据进入时，调用BloomFilter.mightContain()方法判断对应的子订单ID是否已出现过。当没出现过时，调用put()方法将其插入BloomFilter，并交给ValueState。



### HyperLogLog去重





去重方法与上面的类似，将BloomFilter替换成对应的其他算法即可。





或者通过自定义aggFunction来实现。


```java
public class HLLDistinctFunction extends AggregateFunction<Long,HyperLogLog> {

    @Override public HyperLogLog createAccumulator() {
        return new HyperLogLog(0.001);
    }
     
    public void accumulate(HyperLogLog hll,String id){
      hll.offer(id);
    }
     
    @Override public Long getValue(HyperLogLog accumulator) {
        return accumulator.cardinality();
    }
}

```





使用aggFunction和KeyedProcessFunction的差别在于aggFunction默认会使用内部的状态后台，每次来一条数据就会跟rocksdb交互，如果是keyedProcessFunction，取决于用户代码，processElement的写法，如果采用第一种方法状态后台的方式，那跟aggFcuntion没什么差别。



看下aggFunction的源码



WindowedStream.java



```java
@PublicEvolving
public <ACC, V, R> SingleOutputStreamOperator<R> aggregate(
      AggregateFunction<T, ACC, V> aggregateFunction,
      WindowFunction<V, R, K, W> windowFunction,
      TypeInformation<ACC> accumulatorType,
      TypeInformation<R> resultType) {


......

if (evictor != null) {
....
} else {
   AggregatingStateDescriptor<T, ACC, V> stateDesc = new AggregatingStateDescriptor<>("window-contents",
         aggregateFunction, accumulatorType.createSerializer(getExecutionEnvironment().getConfig()));

   operator = new WindowOperator<>(windowAssigner,
         windowAssigner.getWindowSerializer(getExecutionEnvironment().getConfig()),
         keySel,
         input.getKeyType().createSerializer(getExecutionEnvironment().getConfig()),
         stateDesc,
         new InternalSingleValueWindowFunction<>(windowFunction),
         trigger,
         allowedLateness,
         lateDataOutputTag);
}

return input.transform(opName, resultType, operator);


}
```



主要是在这边使用了AggregatingStateDescriptor，如果是process使用的是ListStateDescriptor。

看下AggregatingStateDescriptor的构造函数，是会把用户定义的聚合函数作为入参。

在WindowOperator中存在一个状态



```java
windowState = (InternalAppendingState<K, W, IN, ACC, ACC>) getOrCreateKeyedState(windowSerializer, windowStateDescriptor);
```



这个状态会根据是状态后端的选择而变更。但是内部主体还是AggregatingStateDescriptor。

每当一个数据到达，会调用到



```java
windowState.add(element.getValue());


@Override
public ACC apply(ACC accumulator, IN value) {
   if (accumulator == null) {
      accumulator = aggFunction.createAccumulator();
   }
   return aggFunction.add(value, accumulator);
}
```



这也是为什么选用aggFunction而不是ProcessWindowFunction的原因，processWindowFunction是一个list，aggFunction是来一条数据聚合一条数据。



窗口的使用小技巧可以参考



[https://guosmilesmile.github.io/2020/01/01/Flink-Window%E7%9A%845%E4%B8%AA%E4%BD%BF%E7%94%A8%E5%B0%8F%E6%8A%80%E5%B7%A7/](https://guosmilesmile.github.io/2020/01/01/Flink-Window的5个使用小技巧/)







### Reference

https://www.jianshu.com/p/f6042288a6e3



http://wuchong.me/blog/2020/02/25/demo-building-real-time-application-with-flink-sql/



https://blog.csdn.net/u013516966/article/details/103659306



https://blog.csdn.net/u013516966/article/details/103724895#comments

