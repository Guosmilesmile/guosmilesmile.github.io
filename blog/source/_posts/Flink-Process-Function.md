---
title: Flink Process Function
date: 2019-05-01 12:12:29
tags:
---

ProcessFunction是一个低层次的流处理操作，可以认为是能够访问到keyed state和timers的FlatMapFunction，输入流中接收到的每个事件都会调用它来处理。    
对于容错性状态，ProcessFunction可以通过RuntimeContext来访问Flink的keyed state，方法与其他状态性函数访问keyed state一样。
```java
stream.keyBy(...).process(new MyProcessFunction())
```

本案例中我们将利用 timer 来判断何时 收齐 了某个 window 下所有商品的点击量数据。

```java
/** 求某个窗口中前 N 名的热门点击商品，key 为窗口时间戳，输出为 TopN 的结果字符串 */
public static class TopNHotItems extends KeyedProcessFunction<Tuple, ItemViewCount, String> {

  private final int topSize;

  public TopNHotItems(int topSize) {
    this.topSize = topSize;
  }

  // 用于存储商品与点击数的状态，待收齐同一个窗口的数据后，再触发 TopN 计算
  private ListState<ItemViewCount> itemState;

  @Override
  public void open(Configuration parameters) throws Exception {
    super.open(parameters);
    // 状态的注册
    ListStateDescriptor<ItemViewCount> itemsStateDesc = new ListStateDescriptor<>(
        "itemState-state",
        ItemViewCount.class);
    itemState = getRuntimeContext().getListState(itemsStateDesc);
  }

  @Override
  public void processElement(
      ItemViewCount input,
      Context context,
      Collector<String> collector) throws Exception {

    // 每条数据都保存到状态中
    itemState.add(input);
    // 注册 windowEnd+1 的 EventTime Timer, 当触发时，说明收齐了属于windowEnd窗口的所有商品数据
    context.timerService().registerEventTimeTimer(input.windowEnd + 1);
  }

  @Override
  public void onTimer(
      long timestamp, OnTimerContext ctx, Collector<String> out) throws Exception {
    // 获取收到的所有商品点击量
    List<ItemViewCount> allItems = new ArrayList<>();
    for (ItemViewCount item : itemState.get()) {
      allItems.add(item);
    }
    // 提前清除状态中的数据，释放空间
    itemState.clear();
    // 按照点击量从大到小排序
    allItems.sort(new Comparator<ItemViewCount>() {
      @Override
      public int compare(ItemViewCount o1, ItemViewCount o2) {
        return (int) (o2.viewCount - o1.viewCount);
      }
    });
    // 将排名信息格式化成 String, 便于打印
    StringBuilder result = new StringBuilder();
    result.append("====================================\n");
    result.append("时间: ").append(new Timestamp(timestamp-1)).append("\n");
    for (int i=0;i<topSize;i++) {
      ItemViewCount currentItem = allItems.get(i);
      // No1:  商品ID=12224  浏览量=2413
      result.append("No").append(i).append(":")
            .append("  商品ID=").append(currentItem.itemId)
            .append("  浏览量=").append(currentItem.viewCount)
            .append("\n");
    }
    result.append("====================================\n\n");

    out.collect(result.toString());
  }
}
```
* 为什么要注册windowEnd+1定时器？  

1. 由于 Watermark 的进度是全局的，在 processElement 方法中，每当收到一条数据（ ItemViewCount ），我们就注册一个 windowEnd+1 的定时器（Flink 框架会自动忽略同一时间的重复注册，因此可以重复注册）
2. windowEnd+1 的定时器被触发时，应该是到下一个窗口了，即收齐了该 windowEnd 下的所有商品窗口统计值

* ListState是做什么用的
使用了 ListState<ItemViewCount> 来存储收到的每条 ItemViewCount 消息，保证在发生故障时，状态数据的不丢失和一致性。 ListState 是 Flink 提供的类似 Java List 接口的 State API，它集成了框架的 checkpoint 机制，自动做到了 exactly-once 的语义保证。