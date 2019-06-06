---
title: Flink allowedLateness
date: 2019-03-24 13:34:18
tags: 
categories:
	- 流式计算 
	- Flink
---


### out-of-order element与late element
* 通过watermark机制来处理out-of-order的问题，属于第一层防护，属于全局性的防护，通常说的乱序问题的解决办法，就是指这类；
* 通过窗口上的allowedLateness机制来处理out-of-order的问题，属于第二层防护，属于特定window operator的防护，late element的问题就是指这类。

### allowedLateness

当watermark通过end-of-window之后，再有之前的数据到达时，这些数据会被删除。为了避免有些迟到的数据被删除，因此产生了allowedLateness的概念。
```java
DataStream<T> input = ...;

input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .allowedLateness(<time>)
    .<windowed transformation>(<window function>);
```
**注意**：对于trigger是默认的EventTimeTrigger的情况下，allowedLateness会再次触发窗口的计算，而之前触发的数据，会buffer起来，直到watermark超过end-of-window + allowedLateness（）的时间，窗口的数据及元数据信息才会被删除。再次计算就是DataFlow模型中的Accumulating的情况。

### 例子
在EventTime的情况下，       
 
1. 一条记录的事件时间来控制此条记录属于哪一个窗口，Watermarks来控制这个窗口什么时候激活。
   
2. 假如一个窗口时间为00:00:00～00:00:05，Watermarks为5秒，那么当flink收到事件事件为00:00:10秒的数据时，即Watermarks到达00:00:05，激活这个窗口。

3. 假如设置allowedLateness为60秒，那么窗口的状态会一直保持到事件时间为00:01:05的数据到达，或者如果最后一条数据早于00:01:05秒，则等到最后一条数据到达后再等待此数据于00:01:05的差值时间。    
 
4. 那么在窗口被销毁前，可以通过一些方式再次激活。注意，allowedLateness只能控制窗口销毁行为，并不能控制窗口再次激活的行为，这是独立的两部分行为。
 
5. 官方文档推荐的方式为Getting late data as a side output，可以单独获得再次被激活的窗口流    
https://ci.apache.org/projects/flink/flink-docs-release-1.7/dev/stream/operators/windows.html#getting-late-data-as-a-side-output

### summary
* Flink中处理乱序依赖watermark+window+trigger，属于全局性的处理；
* 同时，对于window而言，还提供了allowedLateness方法，使得更大限度的允许乱序，属于局部性的处理；
* 其中，allowedLateness只针对Event Time有效；
* allowedLateness可用于TumblingEventTimeWindow、SlidingEventTimeWindow以及EventTimeSessionWindows，要注意这可能使得窗口再次被触发，相当于对前一次窗口的窗口的修正（累加计算或者累加撤回计算）；
* 要注意再次触发窗口时，UDF中的状态值的处理，要考虑state在计算时的去重问题。(重要)
* 最后要注意的问题，就是sink的问题，由于同一个key的同一个window可能被sink多次，因此sink的数据库要能够接收此类数据。
