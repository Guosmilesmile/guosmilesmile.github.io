---
title: flink sql count踩坑
date: 2021-08-02 22:41:27
tags:
categories:
	- Flink
---




## Reference
转自https://mp.weixin.qq.com/s/5XDkmuEIfHB_WsMHPeinkw

## 1.序篇

通过本文你可了解到

* 踩坑场景篇-这个坑是啥样的
* 问题排查篇-坑的排查过程
* 问题原理解析篇-导致问题的机制是什么
* 避坑篇-如何避免这种问题
* 展望篇-有什么机制可以根本避免这种情况


#### 先说下结论
在非窗口类 flink sql 任务中，会存在 retract 机制，即上游会向下游发送「撤回消息（做减法）」，**最新的结果消息（做加法）**两条消息来计算结果，保证结果正确性。

而如果我们在上下游中间使用了映射类 udf 改变了**撤回消息（做减法）「的一些字段值时，就可能会导致」撤回消息（做减法）**不能被正常处理，最终导致结果的错误。

## 2.踩坑场景篇-这个坑是啥样的


在介绍坑之前我们先介绍下我们的需求、实现方案的背景。

### 2.1.背景

在各类游戏中都会有一种场景，一个用户可以从 A 等级升级到 B 等级，用户可以不断的升级，但是一个用户同一时刻只会在同一个等级。需求指标就是当前分钟各个等级的用户数。

### 2.2.预期效果

![image](https://note.youdao.com/yws/api/personal/file/167756F31262445DAFCAC5E409554D2E?method=download&shareKey=80b6b06f06cf5c5e1aa4be1aa4be845d)

### 2.3.解决思路


1. 获取到当前所有用户的最新等级
2. 一个用户同一时刻只会在一个等级，所以对每一个等级的用户做 count 操作

### 2.4.解决方案


1. 获取到当前所有用户的最新等级：flink sql row_number() 就可以实现，按照数据的 rowtime 进行逆序排序就可以获取到用户当前最新的等级
2. 对每一个等级的用户做 count 操作：对 row_number() 的后的明细结果进行 count 操作

具体实现 sql 如下，非常简单：
```sql
WITH detail_tmp AS (
  SELECT
    等级,
    id,
    `timestamp`
  FROM
    (
      SELECT
        等级,
        id,
        `timestamp`,
        -- row_number 获取最新状态
        row_number() over(
          PARTITION by id
          ORDER BY
            `timestamp` DESC
        ) AS rn
      FROM
        source_db.source_table
    )
  WHERE
    rn = 1
)
SELECT
  DIM.中文等级 as 等级,
  sum(part_uv) as uv
FROM
  (
    SELECT
      等级,
      count(id) as part_uv
    FROM
      detail_tmp
    GROUP BY
      等级,
      mod(id, 1024)
  )
-- 上游数据的等级名称是数字，需求方要求给转换成中文，所以这里加了一个 udf 映射
LEFT JOIN LATERAL TABLE(等级中文映射_UDF(等级)) AS DIM(中文等级) ON TRUE
GROUP BY
  DIM.中文等级
```

### 2.4.2.参数配置

使用 minibatch 参数方式控制数据输出频率。

```
table.exec.mini-batch.enabled : true
-- 设定 60s 的触发间隔
table.exec.mini-batch.allow-latency : 60s
table.exec.mini-batch.size : 10000000000
```

任务 plan。

![image](https://note.youdao.com/yws/api/personal/file/E3D26ED964D24FD7AA57BE7733D43EFB?method=download&shareKey=a65a7497eeae70d00132efcfae1be131)

### 2.5.问题场景

这段 SQL 跑了 n 年都没有问题，但是有一天运营在配置【等级中文映射_UDF】时，不小心将一个等级的中文名给映射错了，虽然马上恢复了，但是当天的实时数据和离线数据对比后却发现，实时产出的数值比离线大很多！！！而之前都是保持一致的。

## 3.问题排查篇-坑的排查过程

首先我们想一下，这个指标是算 uv 的，运营将等级中文名配置错了，也应该是把原有等级的最终结果算少啊，怎么会算多呢？？？

然后我们将场景复现了下，来看看代码：

任务代码，大家可以直接 copy 到本地运行：


```java
public class Test {

    public static void main(String[] args) throws Exception {

        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        env.setParallelism(1);

        EnvironmentSettings settings = EnvironmentSettings
                .newInstance()
                .useBlinkPlanner()
                .inStreamingMode().build();

        StreamTableEnvironment tEnv = StreamTableEnvironment.create(env, settings);

        // 模拟输入
        DataStream<Tuple3<String, Long, Long>> tuple3DataStream =
                env.fromCollection(Arrays.asList(
                        Tuple3.of("2", 1L, 1627218000000L),
                        Tuple3.of("2", 101L, 1627218000000L + 6000L),
                        Tuple3.of("2", 201L, 1627218000000L + 7000L),
                        Tuple3.of("2", 301L, 1627218000000L + 7000L)));
        // 分桶取模 udf
        tEnv.registerFunction("mod", new Mod_UDF());

        // 中文映射 udf
        tEnv.registerFunction("status_mapper", new StatusMapper_UDF());

        tEnv.createTemporaryView("source_db.source_table", tuple3DataStream,
                "status, id, timestamp");

        String sql = "WITH detail_tmp AS (\n"
                + "  SELECT\n"
                + "    status,\n"
                + "    id,\n"
                + "    `timestamp`\n"
                + "  FROM\n"
                + "    (\n"
                + "      SELECT\n"
                + "        status,\n"
                + "        id,\n"
                + "        `timestamp`,\n"
                + "        row_number() over(\n"
                + "          PARTITION by id\n"
                + "          ORDER BY\n"
                + "            `timestamp` DESC\n"
                + "        ) AS rn\n"
                + "      FROM source_db.source_table"
                + "    )\n"
                + "  WHERE\n"
                + "    rn = 1\n"
                + ")\n"
                + "SELECT\n"
                + "  DIM.status_new as status,\n"
                + "  sum(part_uv) as uv\n"
                + "FROM\n"
                + "  (\n"
                + "    SELECT\n"
                + "      status,\n"
                + "      count(distinct id) as part_uv\n"
                + "    FROM\n"
                + "      detail_tmp\n"
                + "    GROUP BY\n"
                + "      status,\n"
                + "      mod(id, 100)\n"
                + "  )\n"
                + "LEFT JOIN LATERAL TABLE(status_mapper(status)) AS DIM(status_new) ON TRUE\n"
                + "GROUP BY\n"
                + "  DIM.status_new";

        Table result = tEnv.sqlQuery(sql);

        tEnv.toRetractStream(result, Row.class).print();

        env.execute();
    }

}
```
 UDF 代码：
 
 
```java
public class StatusMapper_UDF extends TableFunction<String> {

    public void eval(String status) {
        if (status.equals("1")) {
            collector.collect("等级1");
        } else if (status.equals("2")) {
            collector.collect("等级2");
        } else if (status.equals("3")) {
            collector.collect("等级3");
        }
    }

}
```
在正确情况（模拟 UDF 没有任何变动的情况下）的输出结果：
```
(true,等级2,1)
(false,等级2,1)
(true,等级2,2)
(false,等级2,2)
(true,等级2,3)
(false,等级2,3)
(true,等级2,4)
```

最终等级2 的 uv 数为 4，结果复合预期✅。

模拟下用户修改了 udf 配置之后，UDF 代码如下：


```java
public class StatusMapper_UDF extends TableFunction<String> {

    private int i = 0;

    public void eval(String status) {

        if (i == 5) {
            collect("等级4");
        } else {
            if ("1".equals(status)) {
                collector.collect("等级1");
            } else if ("2".equals(status)) {
                collector.collect("等级2");
            } else if ("3".equals(status)) {
                collector.collect("等级3");
            }
        }
        i++;
    }

}
```


得到的结果如下：


```
(true,等级2,1)
(false,等级2,1)
(true,等级2,2)
(false,等级2,2)
(true,等级2,3)
(false,等级2,3)
(true,等级2,7)
```

最终等级2 的 uv 数为 7，很明显这是错误结果❌。

因此可以确定是由于这个 UDF 的处理逻辑变换而导致的结果出现错误。

下文就让我们来分析下其中缘由。


## 问题原理解析篇-导致问题的机制是什么

我们首先来分析下上述 SQL，可以发现整个 flink sql 任务是使用了 unbounded + minibatch 实现的，在 minibatch 触发条件触发时，上游算子会将之前的结果撤回，然后将最新的结果发出。

这个任务的 execution plan 如图所示。

![image](https://note.youdao.com/yws/api/personal/file/6544D0B99CCA47BABC6FBACDFB2EC35F?method=download&shareKey=8214f4abdef9a009d28907fcef291997)


可以从算子图中的一些计算逻辑可以看到，整个任务都是基于 retract 机制运行（count_retract、sum_retract 等）。

而涉及到 udf 的核心逻辑是在 Operator(ID = 7)，和 Operator(ID = 12) 之间。当 Operator(ID = 7) GroupAggregate 结果发生改变之后，会发一条「撤回消息（做减法）」，一条**最新的结果消息（做加法）**到 Operator(ID = 12) GroupAggregate。

![image](https://note.youdao.com/yws/api/personal/file/2D09CADD75814CE3BC90A77C8DAE50C0?method=download&shareKey=fcbe45a744070812d8e1a7ee3c401c7c)

>> Notes：简单解释下上面说的「撤回消息（做减法）」，「最新的结果消息（做加法）」。举个算 count 的例子：当整个任务的第一条数据来之后，之前没有数据，所以不用撤回，结果就是 0（没有数据） + 1（第一条数据） = 1（结果），当第二条结果来之后，就要将上次发的 1 消息（可以理解为是整个任务的一个中间结果）撤回，将最新的结果 2 发下去。那么计算方法就是 1（上次的结果） - 1（撤回） + 2（当前最新的结果消息）= 2（结果）。

通过算子图可以发现，【中文名称映射】UDF 是处于两个 GroupAggregate 之间的。也就是说 Operator(ID = 7) GroupAggregate 发出的「撤回消息（做减法）」，**最新的结果消息（做加法）「都会执行这个 UDF，那么就有可能」撤回消息（做减法）「中的某个作为下游 GroupAggregate 算子 key 的字段会被更改成其他值，那么这条消息就不会发到原来下游 GroupAggregate 算子的原始 key 中，那么原来的 key 的历史结果就撤回不了了。。。但是」最新的结果消息（做加法）**的字段没有被更改时，那么这个消息依然被发到了下游 GroupAggregate 算子，这就会导致没做减法，却做了加法，就会导致结果增加，如下图所示。

![image](https://note.youdao.com/yws/api/personal/file/B27B316927EE4E62923CBF212DCB2808?method=download&shareKey=29ca4f660a8c87f2b3c298c779c632f1)



![image](https://note.youdao.com/yws/api/personal/file/41E23E453B0F44529C25B30B31245333?method=download&shareKey=96c49e4f68769f87086dc0b56061556f)


从这个角度出发，我们来分析下上面的 case，从内层发给外层的消息一条一条来分析。

内层消息怎么来看呢？其实就是将上面的 SQL 中的 left join 删除，重新跑一遍就可以得到结果，结果如下：


```
(true,等级2,1)
(false,等级2,1)
(true,等级2,2)
(false,等级2,2)
(true,等级2,3)
(false,等级4,3)
(true,等级2,4)
```

来分析下内层消息发出之后对应到外层消息的操作：




内层|	外层
---|---
(true,等级2,1)|	(true,等级2,1)
(false,等级2,1)	|(false,等级2,1)
(true,等级2,2)|	(true,等级2,2)
(false,等级2,2)	|(false,等级2,2)
(true,等级2,3)|	(true,等级2,3)


前五条消息不会导致错误，不用详细说明。




内层|	外层
---|---
(true,等级2,1)|	(true,等级2,1)
(false,等级2,1)	|(false,等级2,1)
(true,等级2,2)|	(true,等级2,2)
(false,等级2,2)	|(false,等级2,2)
(true,等级2,3)|	(true,等级2,3)
(false,等级4,3)

第六条消息发出之后，经过 udf 的处理之后，中文名被映射成了【等级4】，而其再通过 hash partition 策略向下发送消息时，就不能将这条撤回消息发到原本 key 为【等级2】的算子中了，这条撤回消息也无法被处理了。



内层|	外层
---|---
(true,等级2,1)|	(true,等级2,1)
(false,等级2,1)	|(false,等级2,1)
(true,等级2,2)|	(true,等级2,2)
(false,等级2,2)	|(false,等级2,2)
(true,等级2,3)|	(true,等级2,3)
(false,等级4,3)|
(true,等级2,4)|	(false,等级2,3) (true,等级2,7)

第七条消息 (true,等级2,4) 发出后，外层 GroupAggregate 算子首先会将上次发出的记过撤回，即(false,等级2,3)，然后将(true,等级2,4)累加到当前的记过上，即 3（上次结果）+ 4（这次最新的结果）= 7（结果）。就导致了上述的错误结果。

定位到问题原因之后，我们来看看怎么避免上述错误。


## 6.避坑篇-如何避免这种问题


### 6.1.从源头避免

udf 这种映射维度的 udf 尽量在上线前就固定下来，避免后续更改造成的数据错误。

### 6.2.替换为 ScalarFunction 进行映射

```java
WITH detail_tmp AS (
  SELECT
    status,
    id,
    `timestamp`
  FROM
    (
      SELECT
        status,
        id,
        `timestamp`,
        row_number() over(
          PARTITION by id
          ORDER BY
            `timestamp` DESC
        ) AS rn
      FROM
        (
          SELECT
            status,
            id,
            `timestamp`
          FROM
            source_db.source_table
        ) t1
    ) t2
  WHERE
    rn = 1
)
SELECT
  -- 在此处进行中文名称映射
  等级中文映射_UDF(status) as status,
  sum(part_uv) as uv
FROM
  (
    SELECT
      status,
      count(distinct id) as part_uv
    FROM
      detail_tmp
    GROUP BY
      status,
      mod(id, 100)
  )
GROUP BY
  status
```

```java
public class StatusMapper_UDF extends ScalarFunction {

    private int i = 0;

    public String eval(String status) {

        if (i == 5) {
            i++;
            return "等级4";
        } else {
            i++;
            if ("1".equals(status)) {
                return "等级1";
            } else if ("2".equals(status)) {
                return "等级2";
            } else if ("3".equals(status)) {
                return "等级3";
            }
        }
        return "未知";
    }

}
```

还是刚刚的逻辑，刚刚的配方，我们先来看一下结果。

```
(true,等级2,1)
(false,等级2,1)
(true,等级2,2)
(false,等级2,2)
(true,等级2,3)
(false,等级4,3)
(true,等级2,4)

```

发现虽然依然会有 (false,等级4,3) 这样的错误撤回数据（这是 udf 决定的，没法避免），但是我们可以发现最终的结果是 (true,等级2,4)，结果依然是正确的。

再来分析下问什么这种方式可以解决，如图 plan。