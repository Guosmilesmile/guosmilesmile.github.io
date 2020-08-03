---
title: Calcite RBO规则
date: 2020-08-03 16:40:05
tags:
categories: Calcite
---



规则来源于CoreRules.java



#### AGGREGATE_PROJECT_MERGE

Rule that recognizes an {@link Aggregate} on top of a {@link Project} and if possible aggregates through the Project or removes the Project. 

通过识别聚合操作对应的投影(project),对可能的投影进行聚合或者删除。

```
/**
 * Planner rule that recognizes a {@link org.apache.calcite.rel.core.Aggregate}
 * on top of a {@link org.apache.calcite.rel.core.Project} and if possible
 * aggregate through the project or removes the project.
 *
 * <p>This is only possible when the grouping expressions and arguments to
 * the aggregate functions are field references (i.e. not expressions).
 *
 * <p>In some cases, this rule has the effect of trimming: the aggregate will
 * use fewer columns than the project did.
 */
```

* 删除投影
```

SELECT `DATE_CD`, SUM(`IB0002001_CN000`)
FROM (SELECT SUM(`test`), `CUBE2L_IB00040010_CN000`.`DATE_CD`, SUM(`CUBE2L_IB00040010_CN000`.`IDX_VAL`) AS `IB0002001_CN000`
FROM `CUBE2L_IB00040010_CN000`
WHERE `CUBE2L_IB00040010_CN000`.`IDX_ID` IN ('IB0002001_CN000') AND `CUBE2L_IB00040010_CN000`.`DATE_CD` = '2020-05-31'
GROUP BY `CUBE2L_IB00040010_CN000`.`DATE_CD`) AS `IB0002001_CN000`
GROUP BY `DATE_CD`

LogicalAggregate(group=[{0}], EXPR$1=[SUM($1)])
  LogicalProject(DATE_CD=[$0], IB0002001_CN000=[$2])
    LogicalAggregate(group=[{0}], EXPR$0=[SUM($1)], IB0002001_CN000=[SUM($2)])
      LogicalProject(DATE_CD=[$0], test=[$2], IDX_VAL=[$1])
        LogicalFilter(condition=[AND(=($3, 'IB0002001_CN000'), =($0, CAST('2020-05-31'):DATE NOT NULL))])
          LogicalTableScan(table=[[CUBE2L_IB00040010_CN000]])
After --------------------
LogicalProject(DATE_CD=[$0], IB0002001_CN000=[$2])
  LogicalAggregate(group=[{0}], EXPR$0=[SUM($1)], IB0002001_CN000=[SUM($2)])
    LogicalProject(DATE_CD=[$0], test=[$2], IDX_VAL=[$1])
      LogicalFilter(condition=[AND(=($3, 'IB0002001_CN000'), =($0, CAST('2020-05-31'):DATE NOT NULL))])
        LogicalTableScan(table=[[CUBE2L_IB00040010_CN000]])
        
SELECT `DATE_CD`, SUM(`IDX_VAL`) AS `IB0002001_CN000`
FROM `CUBE2L_IB00040010_CN000`
WHERE `IDX_ID` = 'IB0002001_CN000' AND `DATE_CD` = '2020-05-31'
GROUP BY `DATE_CD`


```




#### AGGREGATE_PROJECT_PULL_UP_CONSTANTS



Rule that removes constant keys from an {@link Aggregate}


Since the transformed relational expression has to match the original
relational expression, the constants are placed in a projection above the reduced aggregate. If those constants are not used, another rule will remove them from the project.

对常量进行处理，如果常量在子查询和主查询都存在，那么删除在子查询中的常量。如果常量只在子查询中存在，删除对应常量。


* 常量在子查询和主查询都存在
```
SELECT 5, `DATE_CD`, SUM(`IB0002001_CN000`)
FROM (SELECT 5, SUM(`test`), `CUBE2L_IB00040010_CN000`.`DATE_CD`, SUM(`CUBE2L_IB00040010_CN000`.`IDX_VAL`) AS `IB0002001_CN000`
FROM `CUBE2L_IB00040010_CN000`
WHERE `CUBE2L_IB00040010_CN000`.`IDX_ID` IN ('IB0002001_CN000') AND `CUBE2L_IB00040010_CN000`.`DATE_CD` = '2020-05-31'
GROUP BY `CUBE2L_IB00040010_CN000`.`DATE_CD`) AS `IB0002001_CN000`
GROUP BY `DATE_CD`

LogicalProject(EXPR$0=[5], DATE_CD=[$0], EXPR$2=[$1])
  LogicalAggregate(group=[{0}], EXPR$2=[SUM($1)])
    LogicalProject(DATE_CD=[$0], IB0002001_CN000=[$2])
      LogicalAggregate(group=[{0}], EXPR$1=[SUM($1)], IB0002001_CN000=[SUM($2)])
        LogicalProject(DATE_CD=[$0], test=[$2], IDX_VAL=[$1])
          LogicalFilter(condition=[AND(=($3, 'IB0002001_CN000'), =($0, CAST('2020-05-31'):DATE NOT NULL))])
            LogicalTableScan(table=[[CUBE2L_IB00040010_CN000]])
            
After --------------------

LogicalProject(EXPR$0=[5], DATE_CD=[$0], EXPR$2=[$1])
  LogicalAggregate(group=[{0}], EXPR$2=[SUM($1)])
    LogicalProject(DATE_CD=[$0], IB0002001_CN000=[$2])
      LogicalAggregate(group=[{0}], EXPR$1=[SUM($1)], IB0002001_CN000=[SUM($2)])
        LogicalProject(DATE_CD=[$0], test=[$2], IDX_VAL=[$1])
          LogicalFilter(condition=[AND(=($3, 'IB0002001_CN000'), =($0, CAST('2020-05-31'):DATE NOT NULL))])
            LogicalTableScan(table=[[CUBE2L_IB00040010_CN000]])
            
SELECT 5 AS `EXPR$0`, `DATE_CD`, SUM(`IB0002001_CN000`) AS `EXPR$2`
FROM (SELECT `DATE_CD`, SUM(`IDX_VAL`) AS `IB0002001_CN000`
FROM `CUBE2L_IB00040010_CN000`
WHERE `IDX_ID` = 'IB0002001_CN000' AND `DATE_CD` = '2020-05-31'
GROUP BY `DATE_CD`) AS `t2`
GROUP BY `DATE_CD`

```

* 子查询存在常量，主查询不存在

```
SELECT `DATE_CD`, SUM(`IB0002001_CN000`)
FROM (SELECT 5, SUM(`test`), `CUBE2L_IB00040010_CN000`.`DATE_CD`, SUM(`CUBE2L_IB00040010_CN000`.`IDX_VAL`) AS `IB0002001_CN000`
FROM `CUBE2L_IB00040010_CN000`
WHERE `CUBE2L_IB00040010_CN000`.`IDX_ID` IN ('IB0002001_CN000') AND `CUBE2L_IB00040010_CN000`.`DATE_CD` = '2020-05-31'
GROUP BY `CUBE2L_IB00040010_CN000`.`DATE_CD`) AS `IB0002001_CN000`
GROUP BY `DATE_CD`

LogicalAggregate(group=[{0}], EXPR$1=[SUM($1)])
  LogicalProject(DATE_CD=[$0], IB0002001_CN000=[$2])
    LogicalAggregate(group=[{0}], EXPR$1=[SUM($1)], IB0002001_CN000=[SUM($2)])
      LogicalProject(DATE_CD=[$0], test=[$2], IDX_VAL=[$1])
        LogicalFilter(condition=[AND(=($3, 'IB0002001_CN000'), =($0, CAST('2020-05-31'):DATE NOT NULL))])
          LogicalTableScan(table=[[CUBE2L_IB00040010_CN000]])

After --------------------

LogicalAggregate(group=[{0}], EXPR$1=[SUM($1)])
  LogicalProject(DATE_CD=[$0], IB0002001_CN000=[$2])
    LogicalAggregate(group=[{0}], EXPR$1=[SUM($1)], IB0002001_CN000=[SUM($2)])
      LogicalProject(DATE_CD=[$0], test=[$2], IDX_VAL=[$1])
        LogicalFilter(condition=[AND(=($3, 'IB0002001_CN000'), =($0, CAST('2020-05-31'):DATE NOT NULL))])
          LogicalTableScan(table=[[CUBE2L_IB00040010_CN000]])

SELECT `DATE_CD`, SUM(`IB0002001_CN000`) AS `EXPR$1`
FROM (SELECT `DATE_CD`, SUM(`IDX_VAL`) AS `IB0002001_CN000`
FROM `CUBE2L_IB00040010_CN000`
WHERE `IDX_ID` = 'IB0002001_CN000' AND `DATE_CD` = '2020-05-31'
GROUP BY `DATE_CD`) AS `t2`
GROUP BY `DATE_CD`



```

#### AGGREGATE_ANY_PULL_UP_CONSTANTS

效果同上，上面的方法针对的是project，这个方法针对所有node。

差别：上面针对LogicalProject，该方法针对RelNode


#### AGGREGATE_STAR_TABLE


This pattern indicates that an aggregate table may exist. The rule asks
the star table for an aggregate table at the required level of aggregation.

暂时没看懂是做什么用的。


#### AGGREGATE_PROJECT_STAR_TABLE

同上

#### AGGREGATE_PROJECT_STAR_TABLE

同上

#### AGGREGATE_REDUCE_FUNCTIONS

将聚合函数进行拆分，例如avg拆成sum/count

```java
/**
 * <p>Rewrites:
 * <ul>
 *
 * <li>AVG(x) &rarr; SUM(x) / COUNT(x)
 *
 * <li>STDDEV_POP(x) &rarr; SQRT(
 *     (SUM(x * x) - SUM(x) * SUM(x) / COUNT(x))
 *    / COUNT(x))
 *
 * <li>STDDEV_SAMP(x) &rarr; SQRT(
 *     (SUM(x * x) - SUM(x) * SUM(x) / COUNT(x))
 *     / CASE COUNT(x) WHEN 1 THEN NULL ELSE COUNT(x) - 1 END)
 *
 * <li>VAR_POP(x) &rarr; (SUM(x * x) - SUM(x) * SUM(x) / COUNT(x))
 *     / COUNT(x)
 *
 * <li>VAR_SAMP(x) &rarr; (SUM(x * x) - SUM(x) * SUM(x) / COUNT(x))
 *        / CASE COUNT(x) WHEN 1 THEN NULL ELSE COUNT(x) - 1 END
 *
 * <li>COVAR_POP(x, y) &rarr; (SUM(x * y) - SUM(x, y) * SUM(y, x)
 *     / REGR_COUNT(x, y)) / REGR_COUNT(x, y)
 *
 * <li>COVAR_SAMP(x, y) &rarr; (SUM(x * y) - SUM(x, y) * SUM(y, x) / REGR_COUNT(x, y))
 *     / CASE REGR_COUNT(x, y) WHEN 1 THEN NULL ELSE REGR_COUNT(x, y) - 1 END
 *
 * <li>REGR_SXX(x, y) &rarr; REGR_COUNT(x, y) * VAR_POP(y)
 *
 * <li>REGR_SYY(x, y) &rarr; REGR_COUNT(x, y) * VAR_POP(x)
 *
 * </ul>
 *
 * <p>Since many of these rewrites introduce multiple occurrences of simpler
 * forms like {@code COUNT(x)}, the rule gathers common sub-expressions as it
 * goes.
 */

```


#### AGGREGATE_MERGE

如果顶部的聚合key是子查询的聚合key的子集，那么会合并成一个group by 语句，并且合并聚合函数

For example, SUM of SUM becomes SUM; SUM of COUNT becomes COUNT;
MAX of MAX becomes MAX; MIN of MIN becomes MIN. AVG of AVG would not
match, nor would COUNT of COUNT.

```
SELECT `DATE_CD`, SUM(`IB0002001_CN000`)
FROM (SELECT `CUBE2L_IB00040010_CN000`.`DATE_CD`, COUNT(`CUBE2L_IB00040010_CN000`.`IDX_VAL`) AS `IB0002001_CN000`
FROM `CUBE2L_IB00040010_CN000`
WHERE `CUBE2L_IB00040010_CN000`.`IDX_ID` IN ('IB0002001_CN000') AND `CUBE2L_IB00040010_CN000`.`DATE_CD` = '2020-05-31'
GROUP BY `CUBE2L_IB00040010_CN000`.`DATE_CD`) AS `IB0002001_CN000`
GROUP BY `DATE_CD`

LogicalAggregate(group=[{0}], EXPR$1=[SUM($1)])
  LogicalAggregate(group=[{0}], IB0002001_CN000=[COUNT($1)])
    LogicalProject(DATE_CD=[$0], IDX_VAL=[$1])
      LogicalFilter(condition=[AND(=($3, 'IB0002001_CN000'), =($0, CAST('2020-05-31'):DATE NOT NULL))])
        LogicalTableScan(table=[[CUBE2L_IB00040010_CN000]])

After --------------------

LogicalAggregate(group=[{0}], EXPR$1=[COUNT($1)])
  LogicalProject(DATE_CD=[$0], IDX_VAL=[$1])
    LogicalFilter(condition=[AND(=($3, 'IB0002001_CN000'), =($0, CAST('2020-05-31'):DATE NOT NULL))])
      LogicalTableScan(table=[[CUBE2L_IB00040010_CN000]])

SELECT `DATE_CD`, COUNT(`IDX_VAL`) AS `EXPR$1`
FROM `CUBE2L_IB00040010_CN000`
WHERE `IDX_ID` = 'IB0002001_CN000' AND `DATE_CD` = '2020-05-31'
GROUP BY `DATE_CD`


```

#### AggregateRemoveRule


```
/**
 * Planner rule that removes
 * a {@link org.apache.calcite.rel.core.Aggregate}
 * if it computes no aggregate functions
 * (that is, it is implementing {@code SELECT DISTINCT}),
 * or all the aggregate functions are splittable,
 * and the underlying relational expression is already distinct.
 */
 ```
 
 如果没有使用聚合函数，或者所有聚合函数都是可拆分的，并且基础关系表达式已经不同,则删除聚合函数
 
 
 
 ```
 SELECT `DATE_CD`, SUM(`IB0002001_CN000`)
FROM (SELECT `CUBE2L_IB00040010_CN000`.`DATE_CD`, SUM(`CUBE2L_IB00040010_CN000`.`IDX_VAL`) AS `IB0002001_CN000`
FROM `CUBE2L_IB00040010_CN000`
WHERE `CUBE2L_IB00040010_CN000`.`IDX_ID` IN ('IB0002001_CN000') AND `CUBE2L_IB00040010_CN000`.`DATE_CD` = '2020-05-31'
GROUP BY `CUBE2L_IB00040010_CN000`.`DATE_CD`) AS `IB0002001_CN000`
GROUP BY `DATE_CD`

LogicalAggregate(group=[{0}], EXPR$1=[SUM($1)])
  LogicalAggregate(group=[{0}], IB0002001_CN000=[SUM($1)])
    LogicalProject(DATE_CD=[$0], IDX_VAL=[$1])
      LogicalFilter(condition=[AND(=($3, 'IB0002001_CN000'), =($0, CAST('2020-05-31'):DATE NOT NULL))])
        LogicalTableScan(table=[[CUBE2L_IB00040010_CN000]])

After --------------------

LogicalAggregate(group=[{0}], IB0002001_CN000=[SUM($1)])
  LogicalProject(DATE_CD=[$0], IDX_VAL=[$1])
    LogicalFilter(condition=[AND(=($3, 'IB0002001_CN000'), =($0, CAST('2020-05-31'):DATE NOT NULL))])
      LogicalTableScan(table=[[CUBE2L_IB00040010_CN000]])

SELECT `DATE_CD`, SUM(`IDX_VAL`) AS `IB0002001_CN000`
FROM `CUBE2L_IB00040010_CN000`
WHERE `IDX_ID` = 'IB0002001_CN000' AND `DATE_CD` = '2020-05-31'
GROUP BY `DATE_CD`


```

#### AGGREGATE_EXPAND_DISTINCT_AGGREGATES

```java
/**
 * Planner rule that expands distinct aggregates
 * (such as {@code COUNT(DISTINCT x)}) from a
 * {@link org.apache.calcite.rel.core.Aggregate}.
 *
 * <p>How this is done depends upon the arguments to the function. If all
 * functions have the same argument
 * (e.g. {@code COUNT(DISTINCT x), SUM(DISTINCT x)} both have the argument
 * {@code x}) then one extra {@link org.apache.calcite.rel.core.Aggregate} is
 * sufficient.
 *
 * <p>If there are multiple arguments
 * (e.g. {@code COUNT(DISTINCT x), COUNT(DISTINCT y)})
 * the rule creates separate {@code Aggregate}s and combines using a
 * {@link org.apache.calcite.rel.core.Join}.
 */
 ```
 
对distinct函数进行展开。例如将 COUNT(DISTINCT x)函数展开为两层sql，先group by x，再通过一次select count（x）得出结果

```
SELECT `DATE_CD`, SUM(DISTINCT `IB0002001_CN000`)
FROM (SELECT `CUBE2L_IB00040010_CN000`.`DATE_CD`, SUM(`CUBE2L_IB00040010_CN000`.`IDX_VAL`) AS `IB0002001_CN000`
FROM `CUBE2L_IB00040010_CN000`
WHERE `CUBE2L_IB00040010_CN000`.`IDX_ID` IN ('IB0002001_CN000') AND `CUBE2L_IB00040010_CN000`.`DATE_CD` = '2020-05-31'
GROUP BY `CUBE2L_IB00040010_CN000`.`DATE_CD`) AS `IB0002001_CN000`
GROUP BY `DATE_CD`

LogicalAggregate(group=[{0}], EXPR$1=[SUM(DISTINCT $1)])
  LogicalAggregate(group=[{0}], IB0002001_CN000=[SUM($1)])
    LogicalProject(DATE_CD=[$0], IDX_VAL=[$1])
      LogicalFilter(condition=[AND(=($3, 'IB0002001_CN000'), =($0, CAST('2020-05-31'):DATE NOT NULL))])
        LogicalTableScan(table=[[CUBE2L_IB00040010_CN000]])

After --------------------

LogicalAggregate(group=[{0}], EXPR$1=[SUM($1)])
  LogicalAggregate(group=[{0, 1}])
    LogicalAggregate(group=[{0}], IB0002001_CN000=[SUM($1)])
      LogicalProject(DATE_CD=[$0], IDX_VAL=[$1])
        LogicalFilter(condition=[AND(=($3, 'IB0002001_CN000'), =($0, CAST('2020-05-31'):DATE NOT NULL))])
          LogicalTableScan(table=[[CUBE2L_IB00040010_CN000]])

SELECT `DATE_CD`, SUM(`IB0002001_CN000`) AS `EXPR$1`
FROM (SELECT `DATE_CD`, `IB0002001_CN000`
FROM (SELECT `DATE_CD`, SUM(`IDX_VAL`) AS `IB0002001_CN000`
FROM `CUBE2L_IB00040010_CN000`
WHERE `IDX_ID` = 'IB0002001_CN000' AND `DATE_CD` = '2020-05-31'
GROUP BY `DATE_CD`) AS `t1`
GROUP BY `DATE_CD`, `IB0002001_CN000`) AS `t2`
GROUP BY `DATE_CD`

Process finished with exit code 0

```

#### AGGREGATE_EXPAND_DISTINCT_AGGREGATES_TO_JOIN

同上，使用join而不是agg来完成distinct的展开。


#### AGGREGATE_FILTER_TRANSPOSE

```java
/** Rule that matches an {@link Aggregate}
   * on a {@link Join} and removes the left input
   * of the join provided that the left input is also a left join if
   * possible. */
```
匹配过滤器上的Aggregate并进行转置的规则，将聚合推到过滤器下方。
```java
* <p>This rule does not directly improve performance. The aggregate will
 * have to process more rows, to produce aggregated rows that will be thrown
 * away. The rule might be beneficial if the predicate is very expensive to
 * evaluate. The main use of the rule is to match a query that has a filter
 * under an aggregate to an existing aggregate table.

```
从说明上看，将agg提前与filter调用，这样会是的agg处理更多的行,这种规则适用于谓词过滤代价特别大的场景。


目前没有试出该如何使用。


#### AGGREGATE_JOIN_JOIN_REMOVE

多个join的时候，删除不用的join

```

SELECT DISTINCT s.product_id, pc.product_id
FROM sales s
	LEFT JOIN product p ON s.product_id = p.product_id
	LEFT JOIN product_class pc ON s.product_id = pc.product_id
	
become 

SELECT DISTINCT s.product_id, pc.product_id
FROM sales s
	LEFT JOIN product_class pc ON s.product_id = pc.product_id
```

#### AGGREGATE_JOIN_REMOVE

如果该联接是左联接或右联接，并且不计算任何聚集函数或所有聚集调用具有不同的联接，则删除该联接


```
select distinct s.product_id from
sales as s
left join product as p
on s.product_id = p.product_id


select distinct s.product_id from sales as s
 ```
 
#### AGGREGATE_JOIN_TRANSPOSE

Rule that pushes an {@link Aggregate} past a {@link Join}.



#### AGGREGATE_JOIN_TRANSPOSE_EXTENDED

As {@link #AGGREGATE_JOIN_TRANSPOSE}, but extended to push down aggregate functions.


#### AGGREGATE_UNION_TRANSPOSE

Rule that pushes an {@link Aggregate} past a non-distinct {@link Union}


#### AGGREGATE_UNION_AGGREGATE

```java
/** Rule that matches an {@link Aggregate} whose input is a {@link Union}
   * one of whose inputs is an {@code Aggregate}.
   *
   * <p>Because it matches {@link RelNode} for each input of {@code Union}, it
   * will create O(N ^ 2) matches, which may cost too much during the popMatch
   * phase in VolcanoPlanner. If efficiency is a concern, we recommend that you
   * use {@link #AGGREGATE_UNION_AGGREGATE_FIRST}
   * and {@link #AGGREGATE_UNION_AGGREGATE_SECOND} instead. */
 ```
 

 
#### AGGREGATE_UNION_AGGREGATE_FIRST
 
 
#### AGGREGATE_UNION_AGGREGATE_SECOND
 
 
 
#### AggregateCaseToFilterRule

```
SELECT SUM(CASE WHEN gender = 'F' THEN salary END) FROM Emp

become 

SELECT SUM(salary) FILTER (WHERE gender = 'F')  FROM Emp


```
 
#### CALC_MERGE

合并LogicalCalc 

#### 忽略calc相关


#### FILTER_INTO_JOIN

Rule that tries to push filter expressions into a join condition and into the inputs of the join.

将join外层的filter下推到join左右两侧。谓词下推。

#### FILTER_MERGE

合并两个LogicalFilter


#### FILTER_CALC_MERGE

将Filter合并进LogicalCalc（LogicalCalc由Filter和Project组合成)

#### FILTER_TO_CALC

同PROJECT_TO_CALC



#### FILTER_PROJECT_TRANSPOSE

将filter下推到子查询中


```
SELECT `DATE_CD`, SUM(`IB0002001_CN000`)
FROM (SELECT `IDX_ID`, `CUBE2L_IB00040010_CN000`.`DATE_CD`, SUM(`CUBE2L_IB00040010_CN000`.`IDX_VAL`) AS `IB0002001_CN000`
FROM `CUBE2L_IB00040010_CN000`
GROUP BY `CUBE2L_IB00040010_CN000`.`DATE_CD`, `IDX_ID`) AS `IB0002001_CN000`
WHERE `IDX_ID` IN ('IB0002001_CN000') AND `DATE_CD` = '2020-05-31'
GROUP BY `DATE_CD`

LogicalAggregate(group=[{0}], EXPR$1=[SUM($1)])
  LogicalProject(DATE_CD=[$1], IB0002001_CN000=[$2])
    LogicalFilter(condition=[AND(=($0, 'IB0002001_CN000'), =($1, CAST('2020-05-31'):DATE NOT NULL))])
      LogicalProject(IDX_ID=[$1], DATE_CD=[$0], IB0002001_CN000=[$2])
        LogicalAggregate(group=[{0, 1}], IB0002001_CN000=[SUM($2)])
          LogicalProject(DATE_CD=[$0], IDX_ID=[$3], IDX_VAL=[$1])
            LogicalTableScan(table=[[CUBE2L_IB00040010_CN000]])

After --------------------

LogicalAggregate(group=[{0}], EXPR$1=[SUM($1)])
  LogicalProject(DATE_CD=[$1], IB0002001_CN000=[$2])
    LogicalProject(IDX_ID=[$1], DATE_CD=[$0], IB0002001_CN000=[$2])
      LogicalFilter(condition=[AND(=($1, 'IB0002001_CN000'), =($0, CAST('2020-05-31'):DATE NOT NULL))])
        LogicalAggregate(group=[{0, 1}], IB0002001_CN000=[SUM($2)])
          LogicalProject(DATE_CD=[$0], IDX_ID=[$3], IDX_VAL=[$1])
            LogicalTableScan(table=[[CUBE2L_IB00040010_CN000]])

SELECT `DATE_CD`, SUM(`IB0002001_CN000`) AS `EXPR$1`
FROM (SELECT `DATE_CD`, `IDX_ID`, SUM(`IDX_VAL`) AS `IB0002001_CN000`
FROM `CUBE2L_IB00040010_CN000`
GROUP BY `DATE_CD`, `IDX_ID`
HAVING `IDX_ID` = 'IB0002001_CN000' AND `DATE_CD` = '2020-05-31') AS `t1`
GROUP BY `DATE_CD`

```


#### FILTER_AGGREGATE_TRANSPOSE

将Filter下推到agg下

以上面的规则为例，在加上FILTER_AGGREGATE_TRANSPOSE,可以将having的内容下推到where

```
builder.addRuleInstance(CoreRules.FILTER_PROJECT_TRANSPOSE);
builder.addRuleInstance(CoreRules.FILTER_AGGREGATE_TRANSPOSE);
```

```
SELECT `DATE_CD`, SUM(`IB0002001_CN000`)
FROM (SELECT `IDX_ID`, `CUBE2L_IB00040010_CN000`.`DATE_CD`, SUM(`CUBE2L_IB00040010_CN000`.`IDX_VAL`) AS `IB0002001_CN000`
FROM `CUBE2L_IB00040010_CN000`
GROUP BY `CUBE2L_IB00040010_CN000`.`DATE_CD`, `IDX_ID`) AS `IB0002001_CN000`
WHERE `IDX_ID` IN ('IB0002001_CN000') AND `DATE_CD` = '2020-05-31'
GROUP BY `DATE_CD`

LogicalAggregate(group=[{0}], EXPR$1=[SUM($1)])
  LogicalProject(DATE_CD=[$1], IB0002001_CN000=[$2])
    LogicalFilter(condition=[AND(=($0, 'IB0002001_CN000'), =($1, CAST('2020-05-31'):DATE NOT NULL))])
      LogicalProject(IDX_ID=[$1], DATE_CD=[$0], IB0002001_CN000=[$2])
        LogicalAggregate(group=[{0, 1}], IB0002001_CN000=[SUM($2)])
          LogicalProject(DATE_CD=[$0], IDX_ID=[$3], IDX_VAL=[$1])
            LogicalTableScan(table=[[CUBE2L_IB00040010_CN000]])

After --------------------

LogicalAggregate(group=[{0}], EXPR$1=[SUM($1)])
  LogicalProject(DATE_CD=[$1], IB0002001_CN000=[$2])
    LogicalProject(IDX_ID=[$1], DATE_CD=[$0], IB0002001_CN000=[$2])
      LogicalAggregate(group=[{0, 1}], IB0002001_CN000=[SUM($2)])
        LogicalFilter(condition=[AND(=($1, 'IB0002001_CN000'), =($0, 2020-05-31))])
          LogicalProject(DATE_CD=[$0], IDX_ID=[$3], IDX_VAL=[$1])
            LogicalTableScan(table=[[CUBE2L_IB00040010_CN000]])

SELECT `DATE_CD`, SUM(`IB0002001_CN000`) AS `EXPR$1`
FROM (SELECT `DATE_CD`, SUM(`IDX_VAL`) AS `IB0002001_CN000`
FROM (SELECT `DATE_CD`, `IDX_ID`, `IDX_VAL`
FROM `CUBE2L_IB00040010_CN000`) AS `t`
WHERE `IDX_ID` = 'IB0002001_CN000' AND `DATE_CD` = DATE '2020-05-31'
GROUP BY `DATE_CD`, `IDX_ID`) AS `t3`
GROUP BY `DATE_CD`

```

#### PROJECT_FILTER_TRANSPOSE

将project下推到filter之后

目前没有想到这种场景

#### PROJECT_JOIN_JOIN_REMOVE

```
SELECT s.product_id, pc.product_id
FROM sales s
	LEFT JOIN product p ON s.product_id = p.product_id
	LEFT JOIN product_class pc ON s.product_id = pc.product_id
 
 
 becomes

SELECT s.product_id, pc.product_id
FROM sales s
	LEFT JOIN product_class pc ON s.product_id = pc.product_id


 ```
 
### PROJECT_JOIN_REMOVE

```
SELECT s.product_id
FROM sales s
	LEFT JOIN product p ON s.product_id = p.product_id


become

select s.product_id from sales as s

```


#### PROJECT_JOIN_TRANSPOSE

将project下推到join

目前没有想到场景

#### PROJECT_MERGE

将project进行merge


#### JOIN_PROJECT_BOTH_TRANSPOSE


#### JOIN_PROJECT_LEFT_TRANSPOSE

#### JOIN_PROJECT_RIGHT_TRANSPOSE