---
title: Calcite RBO 出现 VALUES导致sql变型
date: 2020-11-21 16:22:21
tags:
categories: Calcite
---


## 背景


在使用calcite的时候，会遇到莫名奇妙的时候，sql在经过RBO优化后整个sql变形了，出现了如下内容

```sql
FROM (VALUES  (NULL, NULL, NULL, NULL, NULL, NULL)) AS `t` ( `DATE_CD`, `IDX_VAL`, `IDX_VALUE`, `test`, `IDX_ID`)
```


## 原因

明确出现该场景的主要原因是sql出现了false的过滤条件，经过优化的时候calcite将该场景转为VALUES


## 案发现场

### sql

```sql

select DATE_CD from tempTable where IDX_ID ='10' and IDX_ID ='20' 

```

```sql
select DATE_CD from tempTable where 1=2
```

## 场景一

在RBO中使用CoreRules.FILTER_REDUCE_EXPRESSIONS


直接看源码

```java
 @Override public void onMatch(RelOptRuleCall call) {
      final Filter filter = call.rel(0);
      final List<RexNode> expList =
          Lists.newArrayList(filter.getCondition());
      RexNode newConditionExp;
      boolean reduced;
      final RelMetadataQuery mq = call.getMetadataQuery();
      final RelOptPredicateList predicates =
          mq.getPulledUpPredicates(filter.getInput());
      if (reduceExpressions(filter, expList, predicates, true,
          matchNullability)) {
        assert expList.size() == 1;
        newConditionExp = expList.get(0);
        reduced = true;
      } else {
        // No reduction, but let's still test the original
        // predicate to see if it was already a constant,
        // in which case we don't need any runtime decision
        // about filtering.
        newConditionExp = filter.getCondition();
        reduced = false;
      }

      // Even if no reduction, let's still test the original
      // predicate to see if it was already a constant,
      // in which case we don't need any runtime decision
      // about filtering.
      if (newConditionExp.isAlwaysTrue()) {
        call.transformTo(
            filter.getInput());
      } else if (newConditionExp instanceof RexLiteral
          || RexUtil.isNullLiteral(newConditionExp, true)) {
        call.transformTo(createEmptyRelOrEquivalent(call, filter));
      } else if (reduced) {
        call.transformTo(call.builder()
            .push(filter.getInput())
            .filter(newConditionExp).build());
      } else {
        if (newConditionExp instanceof RexCall) {
          boolean reverse = newConditionExp.getKind() == SqlKind.NOT;
          if (reverse) {
            newConditionExp = ((RexCall) newConditionExp).getOperands().get(0);
          }
          reduceNotNullableFilter(call, filter, newConditionExp, reverse);
        }
        return;
      }

      // New plan is absolutely better than old plan.
      call.getPlanner().prune(filter);
    }
```


分析下具体的内容

这段代码会将获取到的所有condition经过下面的方法进行reduce，赋值给newConditionExp

```
reduceExpressions(filter, expList, predicates, true,
          matchNullability)
```


如果newConditionExp一直是true，就会直接prune这个filter，将input暴露出去
```java
 call.transformTo(
            filter.getInput());
```
如果有reduce就会将新的filter替换旧的filter

```java
  call.transformTo(call.builder()
            .push(filter.getInput())
            .filter(newConditionExp).build());
```

如果reduce后是只剩下一个数值或者null或者false，就会调用
```java
 call.transformTo(createEmptyRelOrEquivalent(call, filter));
```
createEmptyRelOrEquivalent方法的内容很简单,就是构建上面的values

```java
/**
     * For static schema systems, a filter that is always false or null can be
     * replaced by a values operator that produces no rows, as the schema
     * information can just be taken from the input Rel. In dynamic schema
     * environments, the filter might have an unknown input type, in these cases
     * they must define a system specific alternative to a Values operator, such
     * as inserting a limit 0 instead of a filter on top of the original input.
     *
     * <p>The default implementation of this method is to call
     * {@link RelBuilder#empty}, which for the static schema will be optimized
     * to an empty
     * {@link org.apache.calcite.rel.core.Values}.
     *
     * @param input rel to replace, assumes caller has already determined
     *              equivalence to Values operation for 0 records or a
     *              false filter.
     * @return equivalent but less expensive replacement rel
     */
    protected RelNode createEmptyRelOrEquivalent(RelOptRuleCall call, Filter input) {
      return call.builder().push(input).empty().build();
    }
```


### 解决方法

重写一个rule，将出现false就调用createEmptyRelOrEquivalent的地方去掉

```java
 @Override public void onMatch(RelOptRuleCall call) {
        final Filter filter = call.rel(0);
        final List<RexNode> expList =
                Lists.newArrayList(filter.getCondition());
        RexNode newConditionExp;
        boolean reduced;
        final RelMetadataQuery mq = call.getMetadataQuery();
        final RelOptPredicateList predicates =
                mq.getPulledUpPredicates(filter.getInput());
        if (reduceExpressions(filter, expList, predicates, true,
                matchNullability)) {
            assert expList.size() == 1;
            newConditionExp = expList.get(0);
            reduced = true;
        } else {
            // No reduction, but let's still test the original
            // predicate to see if it was already a constant,
            // in which case we don't need any runtime decision
            // about filtering.
            newConditionExp = filter.getCondition();
            reduced = false;
        }

        // Even if no reduction, let's still test the original
        // predicate to see if it was already a constant,
        // in which case we don't need any runtime decision
        // about filtering.
        if (newConditionExp.isAlwaysTrue()) {
            call.transformTo(
                    filter.getInput());
        } else if (reduced) {
            RelNode build = call.builder()
                    .push(filter.getInput())
                    .filter(newConditionExp).build();
            call.transformTo(build);
        } else {
            if (newConditionExp instanceof RexCall) {
                boolean reverse = newConditionExp.getKind() == SqlKind.NOT;
                if (reverse) {
                    newConditionExp = ((RexCall) newConditionExp).getOperands().get(0);
                }
                reduceNotNullableFilter(call, filter, newConditionExp, reverse);
            }
            return;
        }

        // New plan is absolutely better than old plan.
        call.getPlanner().prune(filter);
    }
```

返回的sql会变成

```sql
SELECT `DATE_CD`
FROM `tempTable`
WHERE FALSE
```

## 场景二

自定义Rule中使用RelBuilder.filter方法

有时候我们会自己使用RelBuilder来构建sql，进行RBO。一旦我们filter塞进去两个对立的条件，就会出现empty的调用。


RelBuilder源码如下

```java
 /**
     * Creates a {@link Filter} of a list of correlation variables
     * and a list of predicates.
     *
     * <p>The predicates are combined using AND,
     * and optimized in a similar way to the {@link #and} method.
     * If the result is TRUE no filter is created.
     */
    public RelBuilder filter(Iterable<CorrelationId> variablesSet,
                             Iterable<? extends RexNode> predicates) {
        final RexNode simplifiedPredicates =
                simplifier.simplifyFilterPredicates(predicates);

        if (simplifiedPredicates == null) {
            return empty();
        }
       
        if (!simplifiedPredicates.isAlwaysTrue()) {
            final Frame frame = stack.pop();
            final RelNode filter =
                    struct.filterFactory.createFilter(frame.rel,
                            simplifiedPredicates, ImmutableSet.copyOf(variablesSet));
            stack.push(new Frame(filter, frame.fields));
        }
        return this;
    }
```

进一步看下 simplifier.simplifyFilterPredicates.

Rimplifier
```java
 public RexNode simplifyFilterPredicates(Iterable<? extends RexNode> predicates) {
    final RexNode simplifiedAnds =
        withPredicateElimination(Bug.CALCITE_2401_FIXED)
            .simplifyUnknownAsFalse(
                RexUtil.composeConjunction(rexBuilder, predicates));
    if (simplifiedAnds.isAlwaysFalse()) {
      return null;
    }

    // Remove cast of BOOLEAN NOT NULL to BOOLEAN or vice versa. Filter accepts
    // nullable and not-nullable conditions, but a CAST might get in the way of
    // other rewrites.
    return removeNullabilityCast(simplifiedAnds);
  }
 ```

进行条件的合并是在这个方法

```java
 RexUtil.composeConjunction(rexBuilder, predicates)
```
从Rimplifier.simplifyand 可以看出，从composeConjunction得到false的结果会直接返回null，返回的null会触发RelBuilder调用empty.

### 解决方法


重写RelBuilder，直接调用RexUtil.composeConjunction(cluster.getRexBuilder(), predicates)即可。


```java
  /**
     * Creates a {@link Filter} of a list of correlation variables
     * and a list of predicates.
     *
     * <p>The predicates are combined using AND,
     * and optimized in a similar way to the {@link #and} method.
     * If the result is TRUE no filter is created.
     */
    public RelBuilder filter(Iterable<CorrelationId> variablesSet,
                             Iterable<? extends RexNode> predicates) {
        RexNode simplifiedPredicates =
                simplifier.simplifyFilterPredicates(predicates);

        if (simplifiedPredicates == null) {
             simplifiedPredicates = RexUtil.composeConjunction(cluster.getRexBuilder(), predicates);
            //return empty();
        }
      

        if (!simplifiedPredicates.isAlwaysTrue()) {
            final Frame frame = stack.pop();
            final RelNode filter =
                    struct.filterFactory.createFilter(frame.rel,
                            simplifiedPredicates, ImmutableSet.copyOf(variablesSet));
            stack.push(new Frame(filter, frame.fields));
        }
        return this;
    }
```
