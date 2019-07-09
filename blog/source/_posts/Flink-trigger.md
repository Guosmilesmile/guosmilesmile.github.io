---
title: Flink trigger
date: 2019-03-24 13:31:37
tags:
categories:
    - Flink
---




## 1. 窗口触发器
触发器(Trigger)决定了窗口(请参阅窗口概述)博文)什么时候准备好被窗口函数处理。每个窗口分配器都带有一个默认的 Trigger。如果默认触发器不能满足你的要求，可以使用 trigger(...) 指定自定义的触发器。

触发器接口有五个方法来对不同的事件做出响应：
```java
public abstract TriggerResult onElement(T element, long timestamp, W window, TriggerContext ctx) throws Exception;

public abstract TriggerResult onProcessingTime(long time, W window, TriggerContext ctx) throws Exception;

public abstract TriggerResult onEventTime(long time, W window, TriggerContext ctx) throws Exception;

public void onMerge(W window, OnMergeContext ctx) throws Exception {
	throw new UnsupportedOperationException("This trigger does not support merging.");
}

public abstract void clear(W window, TriggerContext ctx) throws Exception;

```

* onElement() 方法，当每个元素被添加窗口时调用。
* onEventTime() 方法，当注册的事件时间计时器被触发时调用。
* onProcessingTime() 方法，当注册的处理时间计时器被触发时调用。
* onMerge() 方法，与状态触发器相关，并且在相应的窗口合并时合并两个触发器的状态。例如，使用会话窗口时。
* clear() 方法，在删除相应窗口时执行所需的任何操作。


以上方法有两件事要注意:

(1) 前三个函数决定了如何通过返回一个 TriggerResult 来对其调用事件采取什么操作。TriggerResult可以是以下之一：

* CONTINUE 什么都不做
* FIRE_AND_PURGE 触发计算，然后清除窗口中的元素
* FIRE 触发计算
* PURGE 清除窗口中的元素      

(2) 上面任何方法都可以用于注册处理时间计时器或事件时间计时器以供将来的操作使用。

### 1.1 触发与清除
一旦触发器确定窗口准备好可以处理数据，就将触发，即，它返回 FIRE 或 FIRE_AND_PURGE。这是窗口算子发出当前窗口结果的信号。给定一个带有 ProcessWindowFunction 的窗口，所有的元素都被传递给 ProcessWindowFunction (可能在将所有元素传递给 evictor 之后)。带有 ReduceFunction， AggregateFunction 或者 FoldFunction 的窗口只是简单地发出他们急切希望得到的聚合结果。

触发器触发时，可以是 FIRE 或 FIRE_AND_PURGE 。FIRE 保留窗口中的内容，FIRE_AND_PURGE 会删除窗口中的内容。默认情况下，内置的触发器只返回 FIRE，不会清除窗口状态。

备注
清除只是简单地删除窗口的内容，并保留窗口的元数据信息以及完整的触发状态。

### 1.2 窗口分配器的默认触发器

窗口分配器的默认触发器适用于许多情况。例如，所有的事件时间窗口分配器都有一个 EventTimeTrigger 作为默认触发器。一旦 watermark 到达窗口末尾，这个触发器就会被触发。

备注
全局窗口(GlobalWindow)的默认触发器是永不会被触发的 NeverTrigger。因此，在使用全局窗口时，必须自定义一个触发器。

通过使用 trigger() 方法指定触发器，将会覆盖窗口分配器的默认触发器。例如，如果你为 TumblingEventTimeWindows 指定 CountTrigger，那么不会再根据时间进度触发窗口，而只能通过计数。目前为止，如果你希望基于时间以及计数进行触发，则必须编写自己的自定义触发器。

### demo  

场景：每次上游下发一定个数的数据，才触发下游的计算
差异：与现有的CountTrigger不同在于onProcessingTime这个函数，出发计算并清空了窗口内的数据，如果用原生的CountTrigger，个数为500个触发计算，如果没有把之前的数据清空，那么会把之前的数据一并纳入计算，导致计算出来的在线人数是重复多次的。

```java
public class CountTimeTrigger<W extends Window> extends Trigger<Object, W> {
    private static final long serialVersionUID = 1L;

    private final long maxCount;

    private final ReducingStateDescriptor<Long> stateDesc = new ReducingStateDescriptor<>("count", new CountTimeTrigger.Sum(), LongSerializer.INSTANCE);

    private CountTimeTrigger(long maxCount) {
        this.maxCount = maxCount;
    }

    @Override
    public TriggerResult onElement(Object element, long timestamp, W window, TriggerContext ctx) throws Exception {
        ReducingState<Long> count = ctx.getPartitionedState(stateDesc);
        count.add(1L);
        if (count.get() >= maxCount) {
            count.clear();
            return TriggerResult.FIRE_AND_PURGE;
        }
        return TriggerResult.CONTINUE;
    }

    @Override
    public TriggerResult onEventTime(long time, W window, TriggerContext ctx) {
        ReducingState<Long> count = ctx.getPartitionedState(stateDesc);
        count.clear();
            return TriggerResult.FIRE_AND_PURGE;
    }

    @Override
    public TriggerResult onProcessingTime(long time, W window, TriggerContext ctx) throws Exception {
        ReducingState<Long> count = ctx.getPartitionedState(stateDesc);
        count.clear();
        return TriggerResult.FIRE_AND_PURGE;
    }

    @Override
    public void clear(W window, TriggerContext ctx) throws Exception {
        ctx.getPartitionedState(stateDesc).clear();
    }

    @Override
    public boolean canMerge() {
        return true;
    }

    @Override
    public void onMerge(W window, OnMergeContext ctx) throws Exception {
        ctx.mergePartitionedState(stateDesc);
    }

    @Override
    public String toString() {
        return "CountTrigger(" +  maxCount + ")";
    }

    /**
     * Creates a trigger that fires once the number of elements in a pane reaches the given count.
     *
     * @param maxCount The count of elements at which to fire.
     * @param <W> The type of {@link Window Windows} on which this trigger can operate.
     */
    public static <W extends Window> CountTimeTrigger<W> of(long maxCount) {
        return new CountTimeTrigger<>(maxCount);
    }

    private static class Sum implements ReduceFunction<Long> {
        private static final long serialVersionUID = 1L;

        @Override
        public Long reduce(Long value1, Long value2) throws Exception {
            return value1 + value2;
        }

    }
}
```