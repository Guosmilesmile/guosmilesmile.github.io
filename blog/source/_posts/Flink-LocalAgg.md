---
title: Flink LocalAgg
date: 2019-08-03 00:06:52
tags:
categories:
	- Flink
---


### 背景

参考了腾讯Oceanus和阿里Blink中minibatch的思路，针对热流进行预聚合的方式解决数据倾斜的问题。

## Local Keyed Streams

现实中，很多数据具有幂律分布(，幂律就是两个通俗的定律，一个是“长尾”理论，只有少数大的门户网站是很多人关注的，但是还有一个长长的尾巴，就是小网站，小公司。长尾理论就是对幂律通俗化的解释。另外一个通俗解释就是马太效应，穷者越穷富者越富)。在处理这类数据时，作业执行性能就会由于负载倾斜而急剧下降。

![image](https://note.youdao.com/yws/api/personal/file/BE6639701DD84910B0E7069F084387D8?method=download&shareKey=5d8081463a3605c4ac3391611a0162c3)

以WordCount程序作为示例。为了统计每个出现word的次数，我们需要将每个word送到对应的aggregator上进行统计。当有部分word出现的次数远远超过其他word时，那么将只有少数的几个aggregator在执行，而其他的aggregator将空闲。当我们增加更多的aggregator时，因为绝大部分word仍然只会被发送到少数那几个aggregator上，程序性能也不会得到任何提高。

### 解决思路

将数据在分发前提前聚合

![image](https://note.youdao.com/yws/api/personal/file/D3287FF635D24326BE7902971F354958?method=download&shareKey=6f56622fca40ccafe099b039abb0f76f)

#### 设计思路

1. 处理每一条记录的时候，先缓存
2. 到达触发条件仅需预聚合
3. 讲这批数据向下发放


#### 实现一个基于条数为触发条件的LocalAgg

* 首先需要有一个方法，方法主要是将新加的元素和存储在内存中相同key的元素仅需合并，做到处理一条数据合并一条数据，而不是一起合并。
* 达到阈值的时候，出发flush动作将数据下发。

```java
import org.apache.flink.api.common.functions.Function;
import org.apache.flink.util.Collector;

import javax.annotation.Nullable;
import java.util.Map;

/**
 * Basic interface for LocalAggFunction processing.
 *
 * @param <K>   The type of the key in the storage map
 * @param <V>   The type of the value in the storage map
 * @param <IN>  Type of the input elements.
 * @param <OUT> Type of the returned elements.
 */
public abstract class LocalAggFunction<K, V, IN, OUT> implements Function {

    private static final long serialVersionUID = -6672219582127325882L;

    /**
     * Adds the given input to the given value, returning the new  value.
     *
     * @param value the existing  value, maybe null
     * @param input the given input, not null
     */
    public abstract V addInput(@Nullable V value, IN input) throws Exception;

    /**
     * Called when a storage is finished. Transform a  to zero, one, or more output elements.
     */
    public abstract void finishBundle(Map<K, V> buffer, Collector<OUT> out) throws Exception;

    public void close() throws Exception {}
}


```

作为一个算子，因为是单流输入，需要继承AbstractStreamOperator和OneInputStreamOperator。

这个算子需要下面变量
1. keySelect获取key
2. Map存储数据
3. 计数
4. 用户函数


processElement处理元素的时候，先获取对应的key，然后内存中的对应key的元素，再交给用户函数去合并数据，并将返回的数据塞入内存中对应key的value。如果计数满足flush条件，调用用户函数的flinsh方法提交数据。




```java
import org.apache.flink.api.common.functions.util.FunctionUtils;
import org.apache.flink.api.java.functions.KeySelector;
import org.apache.flink.streaming.api.operators.AbstractStreamOperator;
import org.apache.flink.streaming.api.operators.OneInputStreamOperator;
import org.apache.flink.streaming.runtime.streamrecord.StreamRecord;
import org.apache.flink.util.Collector;

import java.util.HashMap;
import java.util.Map;

public class LocalAggOperation<IN, OUT, K, V> extends AbstractStreamOperator<OUT>
        implements OneInputStreamOperator<IN, OUT> {

    /**
     * KeySelector is used to extract key for bundle map.
     */
    private final KeySelector<IN, K> keySelector;

    private Map<K, V> storage = new HashMap<>();

    private long FlushCount;

    private long numOfElements = 0;

    /**
     * Output for stream records.
     */
    private transient Collector<OUT> collector;

    public LocalAggOperation(KeySelector<IN, K> keySelector, long flushCount, LocalAggFunction localAggFunction) {
        this.keySelector = keySelector;
        FlushCount = flushCount;
        this.function = localAggFunction;
    }

    protected K getKey(IN input) throws Exception {
        return this.keySelector.getKey(input);
    }

    private final LocalAggFunction<K, V, IN, OUT> function;


    @Override
    public void open() throws Exception {
        super.open();
        this.collector = new StreamRecordCollector<>(output);

    }

    @Override
    public void processElement(StreamRecord<IN> element) throws Exception {
        IN elementValue = element.getValue();

        K key = keySelector.getKey(elementValue);

        V value = storage.get(key);

        V newValue = function.addInput(value, elementValue);

        // update to map bundle
        storage.put(key, newValue);

        numOfElements++;

        if (numOfElements >= FlushCount) {
            function.finishBundle(storage, collector);
            storage.clear();
            numOfElements = 0;
        }


    }

    @Override
    public void close() throws Exception {
        try {
            function.finishBundle(storage, collector);
        } finally {
            Exception exception = null;

            try {
                super.close();
                if (function != null) {
                    FunctionUtils.closeFunction(function);//执行用户函数的close方法
                }
            } catch (InterruptedException interrupted) {
                exception = interrupted;

                Thread.currentThread().interrupt();
            } catch (Exception e) {
                exception = e;
            }

            if (exception != null) {
                LOG.warn("Errors occurred while closing the BundleOperator.", exception);
            }
        }
    }
}

```

StreamRecordCollector主要是实现StreamRecord的复用。
```java
public class StreamRecordCollector<T> implements Collector<T> {

    private final StreamRecord<T> element = new StreamRecord<>(null);

    private final Output<StreamRecord<T>> underlyingOutput;

    public StreamRecordCollector(Output<StreamRecord<T>> output) {
        this.underlyingOutput = output;
    }

    @Override
    public void collect(T record) {
        underlyingOutput.collect(element.replace(record));
    }

    @Override
    public void close() {
        underlyingOutput.close();
    }
}
```


构建流，其实有两种模式，一种是直接改在DataStream这个类中，一种是另外构建一个类。

```java

/**
 * creater of bundler operation
 */
public class LocalAggBuiler {

    public static <K, V, IN, OUT> SingleOutputStreamOperator<OUT> create(LocalAggFunction<K, V, IN, OUT> localAggFunction,
                                                                         long flushCount,
                                                                         KeySelector<IN, K> keySelector,
                                                                         DataStream dataStream) {

        TypeInformation<OUT> outType = TypeExtractor.getUnaryOperatorReturnType(
                (Function) localAggFunction,
                LocalAggFunction.class,
                2,
                3,
                new int[]{},
                dataStream.getType(),
                Utils.getCallLocationName(),
                true);

        return dataStream.transform(
                "localAgg function",
                outType,
                new LocalAggOperation(keySelector, flushCount, localAggFunction)
        );

    }

}

```

到这里就完成了开发。看下如何调用.构建一个TestLocalAggFunction类，实现其中的方法，如果有相同的key，字符串拼接，如果满足条件以key=value的形式下发。

```java
public class TestLocalAggFunction extends LocalAggFunction<String, String, Tuple2<String, String>, String> {

    private int finishCount = 0;
    private List<String> outputs = new ArrayList<>();

    @Override
    public String addInput(@Nullable String value, Tuple2<String, String> input) throws Exception {
        if (value == null) {
            return input.f1;
        } else {
            return value + "," + input.f1;
        }
    }

    @Override
    public void finishBundle(Map<String, String> buffer, Collector<String> out) throws Exception {
        finishCount++;
        outputs.clear();
        for (Map.Entry<String, String> entry : buffer.entrySet()) {
            out.collect(entry.getKey() + "=" + entry.getValue());
        }
    }

    int getFinishCount() {
        return finishCount;
    }

    List<String> getOutputs() {
        return outputs;
    }
}


public class LocalAggTest {

    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        List<Tuple2<String, String>> list = new ArrayList<>();
        list.add(Tuple2.of("gy", "s"));
        list.add(Tuple2.of("gy", "s"));
        list.add(Tuple2.of("gy1", "s"));
        list.add(Tuple2.of("gy1", "s"));
        list.add(Tuple2.of("gy1", "s"));
        DataStream<Tuple2<String, String>> dataStream = env.fromCollection(list);

        KeySelector<Tuple2<String, String>, String> keySelector =
                (KeySelector<Tuple2<String, String>, String>) value -> value.f0;


        DataStream<String> bundleStream = LocalAggBuiler.create(new TestLocalAggFunction(), 3, keySelector, dataStream);
        bundleStream.map(new MapFunction<String, String>() {
            @Override
            public String map(String value) throws Exception {
                return value;
            }
        }).print();
        env.execute();

    }


}

```

结果
```
gy=s,s
gy1=s
guoy1=s,s

```

### 优化

1. LocalAggOperation里关于count触发计算可以抽出一个抽象类叫做CountLocalTrigger，将触发相关的交给用户来实现。
2. 可以实现count和time一起避免数据没到一直不下发的情况。（不过会用到localAgg说明数据量不小了，应该不会出现这种，实现价值不高）

### Reference


https://mp.weixin.qq.com/s?__biz=MzI0NTIxNzE1Ng==&mid=2651217184&idx=1&sn=65da0d5eb9a8f6495c7e53d8fc1e11f9&chksm=f2a31fcbc5d496dd13a6bca6169398d733b62b3980833e170793bde621d3876ae37637853f6b&mpshare=1&scene=1&srcid=0802JsqNqF81zenG78BCNI3i&sharer_sharetime=1564757862875&sharer_shareid=797dbcdd3a4e624875c639b16a4ef5d9&key=6cd3c34421b09586e61385eb3b6f3b9cfdd05b739bfeed6fc1e4fa68ca2a5aac712193b4d850a44facb6eaf6565d321066f90eeda6e466cb5da193e9f0ab82c45e7b5ef23e75c66a7bf850734d6ca5cb&ascene=1&uin=MjU3NDYyMjA0Mw%3D%3D&devicetype=Windows+7&version=62060833&lang=zh_CN&pass_ticket=gV0s9pD7VKBTVznGaxHDu4CFeto3SssDnyhskiwLmml7tHl5ipJiMG%2BZktlqrVEi

