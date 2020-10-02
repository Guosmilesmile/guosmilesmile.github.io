---
title: Calcite in 过长报错
date: 2020-08-21 22:38:20
tags:
categories: Calcite
---



这次遇到一种神奇的现象

```
SELECT `DATE_CD`, SUM(`IB0002001_CN000`)
FROM (SELECT `IDX_ID`, `CUBE2L_IB00040010_CN000`.`DATE_CD`, SUM(`CUBE2L_IB00040010_CN000`.`IDX_VAL`) AS `IB0002001_CN000`
FROM `CUBE2L_IB00040010_CN000`
GROUP BY `CUBE2L_IB00040010_CN000`.`DATE_CD`, `IDX_ID`) AS `IB0002001_CN000`
WHERE `IDX_ID` IN ('A', 'B', 'C', 'D', 'E', 'F', 'G', 'H', 'I', 'J', 'K', 'L', 'M', 'N', 'O', 'P', 'Q', 'R', 'S', 'T') AND `DATE_CD` = '2020-05-31'
GROUP BY `DATE_CD`
```

采用规则CoreRules.AGGREGATE_REMOVE去除所以的外层select。会出现报错。。

```
 java.lang.AssertionError: Need to implement org.apache.calcite.plan.volcano.RelSubset
 ```
 
 看下转出来的规则树，出现了神奇的HepRelVertex
 
 ```
 LogicalAggregate(group=[{0}], EXPR$1=[SUM($1)])
  LogicalProject(DATE_CD=[$1], IB0002001_CN000=[$2])
    LogicalFilter(condition=[AND(true, =($1, CAST('2020-05-31'):DATE NOT NULL))])
      LogicalJoin(condition=[=($0, $3)], joinType=[inner])
        LogicalProject(IDX_ID=[$1], DATE_CD=[$0], IB0002001_CN000=[$2])
          LogicalAggregate(group=[{0, 1}], IB0002001_CN000=[SUM($2)])
            LogicalProject(DATE_CD=[$0], IDX_ID=[$3], IDX_VAL=[$1])
              LogicalTableScan(table=[[CUBE2L_IB00040010_CN000]])
        HepRelVertex(subset=[rel#41:RelSubset#0.NONE.[]])
```

如果不采取规则，得出的规则树是

```
LogicalAggregate(group=[{0}], EXPR$1=[SUM($1)])
  LogicalProject(DATE_CD=[$1], IB0002001_CN000=[$2])
    LogicalFilter(condition=[AND(true, =($1, CAST('2020-05-31'):DATE NOT NULL))])
      LogicalJoin(condition=[=($0, $3)], joinType=[inner])
        LogicalProject(IDX_ID=[$1], DATE_CD=[$0], IB0002001_CN000=[$2])
          LogicalAggregate(group=[{0, 1}], IB0002001_CN000=[SUM($2)])
            LogicalProject(DATE_CD=[$0], IDX_ID=[$3], IDX_VAL=[$1])
              LogicalTableScan(table=[[CUBE2L_IB00040010_CN000]])
        LogicalAggregate(group=[{0}])
          LogicalValues(tuples=[[{ 'A' }, { 'B' }, { 'C' }, { 'D' }, { 'E' }, { 'F' }, { 'G' }, { 'H' }, { 'I' }, { 'J' }, { 'K' }, { 'L' }, { 'M' }, { 'N' }, { 'O' }, { 'P' }, { 'Q' }, { 'R' }, { 'S' }, { 'T' }]])

```

可以看出过滤条件居然变成了inner join的子查询了。


而CoreRules.AGGREGATE_REMOVE规则会把LogicalAggregate/LogicalValues这种树状结构错误优化得到HepRelVertex，就会出现错误。


如果修复这个问题呢，有两个做法


#### 方法一，修改规则，不适配LogicalValues即可

复制CoreRules.AGGREGATE_REMOVE规则源码，修改matchs方法

```java

@Override
public boolean matches(RelOptRuleCall call){
    Aggregate aggregate = call.rel(0);
    RelNode input = aggregate.getInput();
    if(((HepRelVertex) input).getCurrentRel() instanceof logicalValues){
        return false;
    }
    return super.matches(call);
}

```



#### 方法二，过滤条件不转为子查询

通过测试，条件超过19个就会出现子查询，20是一个限制。

过滤源码找出所有跟20有关的数字，发现有这么一项配置

```
public class SqlToelConverter{
    
    public static final int DEFAULT_IN_SUB_QUERY_THRESHOLD =20;
    
}
```

一直找下去就可以发现在构建planner的时候有可以配置的地方。

```java
 SqlToRelConverter.ConfigBuilder configBuilder = SqlToRelConverter.configBuilder();
        SqlToRelConverter.Config config = configBuilder.withInSubQueryThreshold(Integer.MAX_VALUE).build()
        
 FrameworkConfig frameworkConfig = Frameworks.newConfigBuilder()
  .sqlToRelConverterConfig(config)
  .build();
        
```

完整代码
```java
 SqlParser.ConfigBuilder parserConfig = SqlParser.configBuilder();
        SqlParser.Config build = parserConfig.setCaseSensitive(false).setLex(Lex.MYSQL).build();
        SqlToRelConverter.ConfigBuilder configBuilder = SqlToRelConverter.configBuilder();
        SqlToRelConverter.Config config = configBuilder.withInSubQueryThreshold(Integer.MAX_VALUE).build();

        FrameworkConfig frameworkConfig = Frameworks.newConfigBuilder()
                .parserConfig(build)
                .sqlToRelConverterConfig(config)
                .defaultSchema(rootSchema)
                .build();
```


### 如果不想让子查询转为join


一般情况下，如果出现如下sql
```sql

DATE_CD in ( SELECT DATE_CD FROM DIM_DATE WHERE DATE_CD = '2020-05-31' ) 
 ```
 
 
 calcite 会自动将语句转为join语句，可以通过下面的开关关闭该功能。
 
```

 SqlToRelConverter.Config config = configBuilder.withExpand(false)
                .build();


```


