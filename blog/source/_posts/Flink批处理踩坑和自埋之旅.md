---
title: Flink批处理踩坑和自埋之旅
date: 2019-09-24 22:21:08
tags:
categories:
	- Flink
---


### 一、close导致数据变多

```java
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
		List<String> list = Lists.newArrayList("11");
		env.setParallelism(1);
		DataSet<String> dataSet = env.fromCollection(list);
		dataSet.mapPartition(new MapPartitionFunction<String, String>() {
			@Override
			public void mapPartition(Iterable<String> values, Collector<String> out) throws Exception {
				int count = 0;
				for (String value : values) {
					out.collect(value);
					count++;
				}
				out.close();
			}
		}).groupBy(new KeySelector<String, String>() {
			@Override
			public String getKey(String value) throws Exception {
				return value;
			}
		}).reduce(new ReduceFunction<String>() {
			@Override
			public String reduce(String v1, String value2) throws Exception {
				return v1 + value2;
			}
		}).print();
```

### 现象
如上代码会出现数据被重复发送的情况，如果上游发送的内容是11，得到的答案居然是1111。 


### 分析

出现数据重复，一开始怀疑的是数据被重复发送了，涉及到数据聚合,因此将下游的reduce替换成reduceGroup，发现拿到是数据是正常的。

在替换回reduce又异常。

发现在mapPartition中调用了close()函数，一般close函数只需要复写，在operation的生命周期中，会自动调用
```
open 
userFunction.do
close
```
如果调用close，会讲数据flush向下游。导致框架在调用close的时候又发了一次。

那为什么在reduceGroup不会呢。 因为reduceGroup是获取全量的数据作为List才操作用户函数，reduce是两两合并。在信息传递的时候，如果是最后一条会加上一条endofbuffer的事件。所以reduceGroup会在合适的时候加上这句话然后flush。reduce的机制不是

### 处理

去掉close后正常。




### 二、reduce导致yarn容器挂掉

#### 现象

1. 服务器load值500+，无法界定是因为load值高导致容器被yarn认为异常剔除还是其他原因。
2. cpu普通很低
3. iotop查看到磁盘写入很高直接将load跑高，abrt-hook-ccpp 有多个进程 
```
abrt-hook-ccpp 是linux的程序，在进程崩溃的时候会将内存快照等信息dump到磁盘
```
#### 分析
1. 首先排除因为计算导致的cpu异常
2. 内存可能是一个导致爆炸的原因     
通过修改读取数据的大小，将数据压到200M，依旧出现这种情况，当时提供的服务器是5台，每台98G。开始出现灵异事件。
3. 将reduce内的操作全部剔除直接返回，程序正常运行。
4. 重点分析reduce内的操作.


```java

.reduce(new ReduceFunction<DuplicateEntity>() {
    @Override
    public DuplicateEntity reduce(DuplicateEntity value1, DuplicateEntity value2) throws Exception {
        RangeLestEntity rangeEntities = value1.getRangeEntities();
        RangeLestEntity rangeEntities2 = value2.getRangeEntities();
        for (RangeEntity rangeEntity : rangeEntities2.getTreeSet()) {
            rangeEntities.addRange(rangeEntity);
        }
        value1.setRangeEntities(rangeEntities);
        return value1;
    }
})
```

只很对RangeLestEntity这个实体进行操作。


```java
//异常类
public class RangeLestEntity extends TreeSet<RangeEntity> {


}
```
将该类修改为如下操作，运行正常。

```java

//正常运行类
public class RangeLestEntity  {

    private TreeSet<RangeEntity> treeSet = new TreeSet<>();

}

```

#### 猜测

flink的内存管理是自行管理的，内存管理的基础是序列化也要自行实现，针对一个实体，需要判断是否是subClass，如果是subClass就直接处理，如果是父类，要进行进一步处理，在处理的时候会出现问题。估计是一个BUG。


#### 建议

自己构建的类，进行不要继承太过负责的集合。