---
title: 解决Flink1.11不能指定SQL任务JobName问题
date: 2020-08-13 20:00:14
tags:
categories:
	- Flink
---

### Reference

[解决Flink1.11.0不能指定SQL任务JobName问题](https://www.jianshu.com/p/5981646cb1d4)

### 背景

Flink最近刚发布了1.11.0版本，由于加了很多新的功能，对sql的支持更加全面，我就迫不及待的在本地运行了个demo，但是运行的时候报错了：

```
Exception in thread "main" java.lang.IllegalStateException: No operators defined in streaming topology. Cannot execute.
```
虽然报错，但任务却是正常运行，不过任务却不能指定jobname了。

### 原因分析

先看下我的代码：

```
public static void main(String[] args) {
    treamExecutionEnvironment streamEnv = StreamExecutionEnvironment.getExecutionEnvironment();
    EnvironmentSettings settings = EnvironmentSettings.newInstance()
                .inStreamingMode()
                .useBlinkPlanner()
                .build();
    StreamTableEnvironment streamTableEnv = StreamTableEnvironment.create(streamEnv, settings);
    streamEnv.setParallelism(1);

    streamTableEnv.executeSql("CREATE TABLE source xxxx");
    streamTableEnv.executeSql("CREATE TABLE sink xxxx");
    streamTableEnv.executeSql("INSERT INTO sink xxxxx FROM source");
    streamEnv.execute("FlinkTest");
}
```


报错代码在 streamEnv.execute(), 程序找不到算子，所以报错？那问题出在哪？我们先回顾flink1.10.0的版本，看下之前是怎么执行的。之前的版本是通过 sqlUpdate() 方法执行sql的：


```java
public void sqlUpdate(String stmt) {
        List<Operation> operations = parser.parse(stmt);

        if (operations.size() != 1) {
            throw new TableException(UNSUPPORTED_QUERY_IN_SQL_UPDATE_MSG);
        }
        Operation operation = operations.get(0);
        if (operation instanceof ModifyOperation) {
            List<ModifyOperation> modifyOperations = Collections.singletonList((ModifyOperation) operation);
            // 一直是false
            if (isEagerOperationTranslation()) {
                translate(modifyOperations);
            } else {
            // 加到transfomation的list中
                buffer(modifyOperations);
            }
        } else if (operation instanceof CreateTableOperation) {
            ....
        }
}

/**
     * Defines the behavior of this {@link TableEnvironment}. If true the queries will
     * be translated immediately. If false the {@link ModifyOperation}s will be buffered
     * and translated only when {@link #execute(String)} is called.
     *
     * <p>If the {@link TableEnvironment} works in a lazy manner it is undefined what
     * configurations values will be used. It depends on the characteristic of the particular
     * parameter. Some might used values current to the time of query construction (e.g. the currentCatalog)
     * and some use values from the time when {@link #execute(String)} is called (e.g. timeZone).
     *
     * @return true if the queries should be translated immediately.
     */
    protected boolean isEagerOperationTranslation() {
        return false;
    }
```

从isEagerOperationTranslation 方法注释就很清楚的知道了，任务只有在 调用execute(String)方法的时候才会把算子遍历组装成task，这其实是1.11版本之前flink运行sql任务的逻辑。但是1.11版本后，我们不需要再显示指定 execute(String) 方法执行sql任务了(jar包任务不受影响)。下面我们来看1.11版本的 executeSql方法：


```java
@Override
    public TableResult executeSql(String statement) {
        List<Operation> operations = parser.parse(statement);

        if (operations.size() != 1) {
            throw new TableException(UNSUPPORTED_QUERY_IN_EXECUTE_SQL_MSG);
        }

        return executeOperation(operations.get(0));
    }
private TableResult executeOperation(Operation operation) {
        if (operation instanceof ModifyOperation) {
            //直接执行
            return executeInternal(Collections.singletonList((ModifyOperation) operation));
        } else //......
}
```

从1.11版本的代码可以看出，INSERT 语句直接执行，并没有把算子加到transformation的List中，所以当调用 execute(String) 方法时会报错，报错并不影响执行，但是却不能指定jobName了，很多时候jobName 能够反映出 job的业务和功能，不能指定jobname是很多场景所不能接受的。


### Flink 1.11 改动

1.11 对 StreamTableEnvironment.execute()和 StreamExecutionEnvironment.execute() 的执行方式有所调整

简单概述为：
1. StreamTableEnvironment.execute() 只能执行 sqlUpdate 和 insertInto 方法执行作业；
2. Table 转化为 DataStream 后只能通过 StreamExecutionEnvironment.execute() 来执行作业；
3. 新引入的 TableEnvironment.executeSql() 和 StatementSet.execute() 方法是直接执行sql作业
(异步提交作业)，不需要再调用 StreamTableEnvironment.execute()
或 StreamExecutionEnvironment.execute()

详细参考：

[1]
https://ci.apache.org/projects/flink/flink-docs-release-1.11/zh/dev/table/common.html#%E7%BF%BB%E8%AF%91%E4%B8%8E%E6%89%A7%E8%A1%8C%E6%9F%A5%E8%AF%A2

[2]
https://ci.apache.org/projects/flink/flink-docs-release-1.11/zh/dev/table/common.html#%E5%B0%86%E8%A1%A8%E8%BD%AC%E6%8D%A2%E6%88%90-datastream-%E6%88%96-dataset

如果需要批量执行多条sql，应该通过StatementSet 来执行。

```java

StatementSet stmtSet = tEnv.createStatementSet();

Table table1 = tEnv.from("MySource1").where($("word").like("F%"));
stmtSet.addInsert("MySink1", table1);

Table table2 = table1.unionAll(tEnv.from("MySource2"));
stmtSet.addInsert("MySink2", table2);
StatementSet.execute();

```


### 修改源码增加jobname

首先我们追踪代码到executeInternal，如下:

```java

@Override
    public TableResult executeInternal(List<ModifyOperation> operations) {
        List<Transformation<?>> transformations = translate(operations);
        List<String> sinkIdentifierNames = extractSinkIdentifierNames(operations);
        String jobName = "insert-into_" + String.join(",", sinkIdentifierNames);
        // 增加配置 job.name指定jobname
        String name = tableConfig.getConfiguration().getString("job.name", jobName);
        Pipeline pipeline = execEnv.createPipeline(transformations, tableConfig, name);
        try {
            JobClient jobClient = execEnv.executeAsync(pipeline);
            TableSchema.Builder builder = TableSchema.builder();
            Object[] affectedRowCounts = new Long[operations.size()];
            for (int i = 0; i < operations.size(); ++i) {
                // use sink identifier name as field name
                builder.field(sinkIdentifierNames.get(i), DataTypes.BIGINT());
                affectedRowCounts[i] = -1L;
            }

            return TableResultImpl.builder()
                    .jobClient(jobClient)
                    .resultKind(ResultKind.SUCCESS_WITH_CONTENT)
                    .tableSchema(builder.build())
                    .data(Collections.singletonList(Row.of(affectedRowCounts)))
                    .build();
        } catch (Exception e) {
            throw new TableException("Failed to execute sql", e);
        }
    }
```


从上面不难看出，默认jobname是 insert-into_ + sink的表名，正如代码所示，我已经把指定jobname的功能加上了，只需要增加一个job.name的TableConfig即可，然后重新编译flink代码: mvn clean install -DskipTests -Dfas, 线上环境替换掉 flink-table_2.11-1.11.0.jar jar包即可，如果是本地Idea运行，把flink编译好就可以了。

主程序修改如下:


```java
public static void main(String[] args) {
    treamExecutionEnvironment streamEnv = StreamExecutionEnvironment.getExecutionEnvironment();
    EnvironmentSettings settings = EnvironmentSettings.newInstance()
                .inStreamingMode()
                .useBlinkPlanner()
                .build();
    StreamTableEnvironment streamTableEnv = StreamTableEnvironment.create(streamEnv, settings);
    streamEnv.setParallelism(1);
    streamTableEnv.getConfig().getConfiguration().setString("job.name", "OdsCanalFcboxSendIngressStream");
    streamTableEnv.executeSql("CREATE TABLE source xxxx");
    streamTableEnv.executeSql("CREATE TABLE sink xxxx");
    streamTableEnv.executeSql("INSERT INTO sink xxxxx FROM source");
    // streamEnv.execute("FlinkTest");
}
```

