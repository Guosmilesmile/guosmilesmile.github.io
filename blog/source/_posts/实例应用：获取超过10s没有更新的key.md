---
title: 实例应用：获取超过10s没有更新的key
date: 2020-10-12 20:00:20
tags:
categories:
	- Flink
---



## 主要思路

1.ValueState内部包含了计数、key和最后修改时间      
2.对于每一个输入的记录,ProcessFunction都会增加计数,然后注册对应的过期检测timer       
3.在onTimer中进行检测和输出


## 上代码

模拟数据source如下：数据以Tuple3的形式， key，无用，时间戳的形式向下游发送，每隔5秒发送一条数据.

```java

import org.apache.flink.api.java.tuple.Tuple3;
import org.apache.flink.streaming.api.functions.source.RichParallelSourceFunction;

public class StreamDemoDataSource extends RichParallelSourceFunction<Tuple3<String, Long, Long>> {

    private volatile boolean running = true;

    @Override
    public void run(SourceContext<Tuple3<String, Long, Long>> sourceContext) throws Exception {

        Tuple3[] elements = new Tuple3[]{
                Tuple3.of("a", 1L, 1000000050000L),
                Tuple3.of("a", 1L, 1000000054000L),
                Tuple3.of("a", 1L, 1000000079900L),
                Tuple3.of("a", 1L, 1000000115000L),
                Tuple3.of("b", 1L, 1000000100000L),
                Tuple3.of("b", 1L, 1000000108000L)
        };

        int count = 0;
        while (running && count < elements.length) {
            sourceContext.collect(new Tuple3<>((String) elements[count].f0, (Long) elements[count].f1, (Long) elements[count].f2));
            count++;
            Thread.sleep(5000);
        }

    }

    @Override
    public void cancel() {
        running = false;
    }

}
```

KeyprocessFunction

```java

import org.apache.flink.api.common.state.ValueState;
import org.apache.flink.api.common.state.ValueStateDescriptor;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.configuration.Configuration;
import org.apache.flink.streaming.api.functions.KeyedProcessFunction;
import org.apache.flink.util.Collector;
import org.joda.time.DateTime;

public class CountWithTimeoutFunction extends KeyedProcessFunction<String, Tuple2<String, Long>, Tuple2<String, Long>> {


    private ValueState<CountWithTimestamp> state;

    //最先调用
    @Override
    public void open(Configuration parameters) throws Exception {
        //根据上下文获取状态
        state = getRuntimeContext().getState(new ValueStateDescriptor<>("myState", CountWithTimestamp.class));
    }

    @Override
    public void processElement(Tuple2<String, Long> input, Context ctx, Collector<Tuple2<String, Long>> out) throws Exception {
        CountWithTimestamp current = state.value();
        if (current == null) {
            current = new CountWithTimestamp();
            current.key = input.f0;
        }
        //更新ValueState
        current.count++;
        //这里面的context可以获取时间戳
        current.lastModified = ctx.timestamp();
        System.out.println("元素" + input.f0 + "进入事件时间为：" + new DateTime(current.lastModified).toString("yyyy-MM-dd HH:mm:ss"));
        state.update(current);

        //注册ProcessTimer,更新一次就会有一个ProcessTimer
        ctx.timerService().registerEventTimeTimer(current.lastModified + 9000);
        System.out.println("定时触发时间为：" + new DateTime(current.lastModified + 9000).toString("yyyy-MM-dd HH:mm:ss"));

    }

    @Override
    public void onTimer(long timestamp, OnTimerContext ctx, Collector<Tuple2<String, Long>> out) throws Exception {
        //获取上次时间,与参数中的timestamp相比,如果相差等于10s 就会输出
        CountWithTimestamp res = state.value();
        System.out.println("当前时间为：" + new DateTime(timestamp).toString("yyyy-MM-dd HH:mm:ss") + " 当前state :" + res);
        if (timestamp >= res.lastModified + 9000) {
            System.out.println("定时器被触发：" + "当前时间为" + new DateTime(timestamp).toString("yyyy-MM-dd HH:mm:ss") + " 最近修改时间为" + new DateTime(res.lastModified).toString("yyyy-MM-dd HH:mm:ss"));
            out.collect(new Tuple2<String, Long>(res.key, res.count));
        }

    }
}

```

```java

import org.joda.time.DateTime;

public class CountWithTimestamp {

    //单词
    public String key;
    //单词计数
    public long count;
    //最近更新时间
    public long lastModified;

    @Override
    public String toString() {
        return "CountWithTimestamp{" +
                "key='" + key + '\'' +
                ", count=" + count +
                ", lastModified=" + new DateTime(lastModified).toString("yyyy-MM-dd HH:mm:ss") +
                '}';
    }


}

```


主函数如下：

```java

import org.apache.flink.api.common.eventtime.SerializableTimestampAssigner;
import org.apache.flink.api.common.eventtime.WatermarkStrategy;
import org.apache.flink.api.common.functions.MapFunction;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.api.java.tuple.Tuple3;
import org.apache.flink.streaming.api.TimeCharacteristic;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

import java.time.Duration;

public class KeyNotifyFlink {

    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
      
        env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
        DataStream<Tuple2<String, Long>> data = env.addSource(new StreamDemoDataSource()).setParallelism(1)
                .assignTimestampsAndWatermarks(WatermarkStrategy.<Tuple3<String, Long, Long>>forBoundedOutOfOrderness(Duration.ofSeconds(0))
                        .withTimestampAssigner(new SerializableTimestampAssigner<Tuple3<String, Long, Long>>() {
                            @Override
                            public long extractTimestamp(Tuple3<String, Long, Long> element, long recordTimestamp) {
                                return element.f2; //指定EventTime对应的字段
                            }
                        }))
               .map(new MapFunction<Tuple3<String, Long, Long>, Tuple2<String, Long>>() {
                    @Override
                    public Tuple2<String, Long> map(Tuple3<String, Long, Long> input) throws Exception {
                        return Tuple2.of(input.f0, input.f1);
                    }
                });

        data.keyBy(item -> item.f0)
                .process(new CountWithTimeoutFunction()).print();


        env.execute();

    }
}
```

采用的是Flink 1.11 的新的水印生成策略，可以参考
https://guosmilesmile.github.io/2020/07/23/Flink1-11%E5%8D%87%E7%BA%A7%E5%A1%AB%E5%9D%91/
### 结果


```

元素a进入事件时间为：2001-09-09 09:47:30
定时触发时间为：2001-09-09 09:47:39
元素a进入事件时间为：2001-09-09 09:47:34
定时触发时间为：2001-09-09 09:47:43
元素a进入事件时间为：2001-09-09 09:47:59
定时触发时间为：2001-09-09 09:48:08
进入定时器，当前时间为：2001-09-09 09:47:39 当前state :CountWithTimestamp{key='a', count=3, lastModified=2001-09-09 09:47:59}
进入定时器，当前时间为：2001-09-09 09:47:43 当前state :CountWithTimestamp{key='a', count=3, lastModified=2001-09-09 09:47:59}
元素a进入事件时间为：2001-09-09 09:48:35
定时触发时间为：2001-09-09 09:48:44
进入定时器，当前时间为：2001-09-09 09:48:08 当前state :CountWithTimestamp{key='a', count=4, lastModified=2001-09-09 09:48:35}
元素b进入事件时间为：2001-09-09 09:48:20
定时触发时间为：2001-09-09 09:48:29
元素b进入事件时间为：2001-09-09 09:48:28
定时触发时间为：2001-09-09 09:48:37
进入定时器，当前时间为：2001-09-09 09:48:29 当前state :CountWithTimestamp{key='b', count=2, lastModified=2001-09-09 09:48:28}
进入定时器，当前时间为：2001-09-09 09:48:37 当前state :CountWithTimestamp{key='b', count=2, lastModified=2001-09-09 09:48:28}
触发定时器：当前时间为2001-09-09 09:48:37 最近修改时间为2001-09-09 09:48:28
(b,2)
进入定时器，当前时间为：2001-09-09 09:48:44 当前state :CountWithTimestamp{key='a', count=4, lastModified=2001-09-09 09:48:35}
触发定时器：当前时间为2001-09-09 09:48:44 最近修改时间为2001-09-09 09:48:35
(a,4)

Process finished with exit code 0


```

## 注意

###  setStreamTimeCharacteristic


env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
这行必须加，不然会以processTime进行处理，一直得不到结果，因为注册的是EventTime（registerEventTimeTimer）


结果如下

```
元素a进入事件时间为：2001-09-09 09:47:30
定时触发时间为：2001-09-09 09:47:39
元素a进入事件时间为：2001-09-09 09:47:34
定时触发时间为：2001-09-09 09:47:43
元素a进入事件时间为：2001-09-09 09:47:59
定时触发时间为：2001-09-09 09:48:08
元素a进入事件时间为：2001-09-09 09:48:35
定时触发时间为：2001-09-09 09:48:44
元素b进入事件时间为：2001-09-09 09:48:20
定时触发时间为：2001-09-09 09:48:29
元素b进入事件时间为：2001-09-09 09:48:28
定时触发时间为：2001-09-09 09:48:37
当前时间为：2001-09-09 09:47:39 当前state :CountWithTimestamp{key='a', count=4, lastModified=2001-09-09 09:48:35}
当前时间为：2001-09-09 09:47:43 当前state :CountWithTimestamp{key='a', count=4, lastModified=2001-09-09 09:48:35}
当前时间为：2001-09-09 09:48:08 当前state :CountWithTimestamp{key='a', count=4, lastModified=2001-09-09 09:48:35}
当前时间为：2001-09-09 09:48:29 当前state :CountWithTimestamp{key='b', count=2, lastModified=2001-09-09 09:48:28}

```



### ctx.timestamp() 为null

该现象出现于没有加assignTimestampsAndWatermarks 


withTimestampAssigner中的extractTimestamp方法是在TimestampsAndWatermarksOperator.processElement中被调用


TimestampsAndWatermarksOperator.processElement

```java
@Override
	public void processElement(final StreamRecord<T> element) throws Exception {
		final T event = element.getValue();
		final long previousTimestamp = element.hasTimestamp() ? element.getTimestamp() : Long.MIN_VALUE;
		final long newTimestamp = timestampAssigner.extractTimestamp(event, previousTimestamp);

		element.setTimestamp(newTimestamp);
		output.collect(element);
		watermarkGenerator.onEvent(event, newTimestamp, wmOutput);
	}

```


ctx.timestamp()源码见KeyedProcessOperator#ContextImpl.timestamp()

```java
	@Override
		public Long timestamp() {
			checkState(element != null);

			if (element.hasTimestamp()) {
				return element.getTimestamp();
			} else {
				return null;
			}
		}
```

如果没有assignTimestampsAndWatermarks就会以nullPoint的异常出现


### Reference 


https://mp.weixin.qq.com/s?__biz=MzU3MzgwNTU2Mg==&mid=2247487531&idx=1&sn=183d114f36a697eb7df595fa24fec31c&chksm=fd3d56beca4adfa830cdbd99cf0adde30c7622c3541252cdf172ca0b66e60ae475e7796fdc96&mpshare=1&scene=24&srcid=&sharer_sharetime=1588985687033&sharer_shareid=797dbcdd3a4e624875c639b16a4ef5d9#rd
