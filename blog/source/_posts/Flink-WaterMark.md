---
title: Flink WaterMark 水位线
date: 2019-03-17 12:58:30
tags:
categories: Flink
---

 1. 几个重要的概念简述：

* Window：Window是处理无界流的关键，Windows将流拆分为一个个有限大小的buckets，可以可以在每一个buckets中进行计算
* start_time,end_time：当Window时时间窗口的时候，每个window都会有一个开始时间和结束时间（前开后闭），这个时间是系统时间
* event-time: 事件发生时间，是事件发生所在设备的当地时间，比如一个点击事件的时间发生时间，是用户点击操作所在的手机或电脑的时间
* Watermarks：可以把他理解为一个水位线，这个Watermarks在不断的变化，一旦Watermarks大于了某个window的end_time，就会触发此window的计算，Watermarks就是用来触发window计算的。

<!--more-->
# 2. 如何使用Watermarks处理乱序的数据流

什么是乱序呢？可以理解为数据到达的顺序和他的event-time排序不一致。导致这的原因有很多，比如延迟，消息积压，重试等等

因为Watermarks是用来触发window窗口计算的，我们可以根据事件的event-time，计算出Watermarks，并且设置一些延迟，给迟到的数据一些机会。

假如我们设置10s的时间窗口（window），那么0~10s，10~20s都是一个窗口，以0~10s为例，0位start-time，10为end-time。假如有4个数据的event-time分别是8(A),12.5(B),9(C),13.5(D)，我们设置Watermarks为当前所有到达数据event-time的最大值减去延迟值3.5秒

```
当A到达的时候，Watermarks为max{8}-3.5=8-3.5 = 4.5 < 10,不会触发计算
当B到达的时候，Watermarks为max(12.8,5)-3.5=12.5-3.5 = 9 < 10,不会触发计算
当C到达的时候，Watermarks为max(12.5,8,9)-3.5=12.5-3.5 = 9 < 10,不会触发计算
当D到达的时候，Watermarks为max(13.5,12.5,8,9)-3.5=13.5-3.5 = 10 = 10,触发计算
触发计算的时候，会将AC（因为他们都小于10）都计算进去

通过上面这种方式，我们就将迟到的C计算进去了
```
这里的延迟3.5s是我们假设一个数据到达的时候，比他早3.5s的数据肯定也都到达了，这个是需要根据经验推算的，加入D到达以后有到达了一个E,event-time=6，但是由于0~10的时间窗口已经开始计算了，所以E就丢了。


#### 注意 使用水位线的时间，用的是毫秒时间戳，秒级时间戳会出问题。
```java
/**
*
* 该函数是以最大到达时间-允许延迟时间 作为水位线，例如     
* 现在是7:03分，3分钟作为允许延迟，时间窗口是1分钟，那么7.03分计算的是 
* 6.59-7.00分这个时间窗口的数据。6.59分之前的数据都会被丢弃。这种方式
* 的缺点是如果出现未来时间的数据，会导致很多数据被丢弃。
*
**/
public class BoundedOutOfOrdernessGenerator implements AssignerWithPeriodicWatermarks<MyEvent> {

    private final long maxOutOfOrderness = 3500; // 3.5 seconds

    private long currentMaxTimestamp;

    // 提取当前时间戳
    @Override
    public long extractTimestamp(MyEvent element, long previousElementTimestamp) {
        long timestamp = element.getCreationTime();
        currentMaxTimestamp = Math.max(timestamp, currentMaxTimestamp);
        return timestamp;
    }


    // 获取当前水位线
    @Override
    public Watermark getCurrentWatermark() {
        // return the watermark as current highest timestamp minus the out-of-orderness bound
        return new Watermark(currentMaxTimestamp - maxOutOfOrderness);
    }
}

```

```java
/**
* 以当前系统时间-允许延迟，作为水位线。 缺点是如果太多的未来数据，
* 就会分别单独占用一个时间窗口，很容易导致内存爆炸，建议使用rockdb 
* backend，不使用memory backend
**/
public class TimeLagWatermarkGenerator implements AssignerWithPeriodicWatermarks<MyEvent> {

	private final long maxTimeLag = 5000; // 5 seconds

	@Override
	public long extractTimestamp(MyEvent element, long previousElementTimestamp) {
		return element.getCreationTime();
	}

	@Override
	public Watermark getCurrentWatermark() {
		// return the watermark as current time minus the maximum time lag
		return new Watermark(System.currentTimeMillis() - maxTimeLag);
	}
}

```
