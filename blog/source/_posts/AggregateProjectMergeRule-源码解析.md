---
title: AggregateProjectMergeRule 源码解析
date: 2020-10-02 23:06:55
tags:
categories: Calcite
---

```java

public static RelNode apply(RelOptRuleCall call, Aggregate aggregate,
      Project project) {
    // Find all fields which we need to be straightforward field projections.
    final Set<Integer> interestingFields = RelOptUtil.getAllFields(aggregate);// 获取所有的用到的index

    // Build the map from old to new; abort if any entry is not a
    // straightforward field projection.
    final Map<Integer, Integer> map = new HashMap<>();
    for (int source : interestingFields) {
      final RexNode rex = project.getProjects().get(source); // 将在project中的index转为对应input的index，放入map中
      if (!(rex instanceof RexInputRef)) {
        return null;
      }
      map.put(source, ((RexInputRef) rex).getIndex());
    }

    final ImmutableBitSet newGroupSet = aggregate.getGroupSet().permute(map);// 将group中的index转为project的input的index
    ImmutableList<ImmutableBitSet> newGroupingSets = null;
    if (aggregate.getGroupType() != Group.SIMPLE) {
      newGroupingSets =
          ImmutableBitSet.ORDERING.immutableSortedCopy(
              ImmutableBitSet.permute(aggregate.getGroupSets(), map));
    }

    final ImmutableList.Builder<AggregateCall> aggCalls =
        ImmutableList.builder();
    final int sourceCount = aggregate.getInput().getRowType().getFieldCount();
    final int targetCount = project.getInput().getRowType().getFieldCount();
    final Mappings.TargetMapping targetMapping =   // 获取对应的mapping，将aggCall转为以input为数据源的aggCall
        Mappings.target(map, sourceCount, targetCount);
    for (AggregateCall aggregateCall : aggregate.getAggCallList()) {
      aggCalls.add(aggregateCall.transform(targetMapping));
    }
    // 将agg进行拷贝
    final Aggregate newAggregate =
        aggregate.copy(aggregate.getTraitSet(), project.getInput(),
            newGroupSet, newGroupingSets, aggCalls.build());

    // Add a project if the group set is not in the same order or
    // contains duplicates.
    final RelBuilder relBuilder = call.builder();
    relBuilder.push(newAggregate);
    final List<Integer> newKeys =
        Util.transform(aggregate.getGroupSet().asList(), map::get);
    if (!newKeys.equals(newGroupSet.asList())) {
      final List<Integer> posList = new ArrayList<>();
      for (int newKey : newKeys) {
        posList.add(newGroupSet.indexOf(newKey));
      }
      for (int i = newAggregate.getGroupCount();
           i < newAggregate.getRowType().getFieldCount(); i++) {
        posList.add(i);
      }
      relBuilder.project(relBuilder.fields(posList));
    }

    return relBuilder.build();
  }
  
  
 ```
 
### 可以借鉴的地方


#### 根据传递的map，改变排序。

```java

ImmutableBitSet newGroupSet = aggregate.getGroupSet().permute(map);
```

aggregate.getGroupSet() = {0}.
map = { 0->7,1->0}.
newGroupSet = {7}.


#### 获取agg所有用到的index

获取aggSet、aggCall所用到的index
```java

Set<Integer> interestingFields = RelOptUtil.getAllFields(aggregate)

```

####  获取project的input对应的index

```java
RexNode rex = project.getProjects().get(source)
```

加入project的是input的7，0两个元素。
agg的是project的1，2.
那么通过上面的代码可以得到用到的是1对应input的7，2对应input的0.

#### 改变aggCall数据源

```java
final Mappings.TargetMapping targetMapping =   // 获取对应的mapping，将aggCall转为以input为数据源的aggCall
        Mappings.target(map, sourceCount, targetCount);
    for (AggregateCall aggregateCall : aggregate.getAggCallList()) {
      aggCalls.add(aggregateCall.transform(targetMapping));
    }
```

old aggCall=COUNT(DISTINCT $1)
map={0->7,1->0}
通过转换，可以得到
aggCall=COUNT(DISTINCT $0)

