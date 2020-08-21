---
title: Flink SQL join 维表
date: 2020-08-21 22:40:36
tags:
categories: Flink
---




### 语法


flink sql join 维表需要使用特定的语法

```
select a.id,b.name 
from ubt as a 
join info FOR SYSTEM_TIME AS OF a.proctime as b 
on a.id=b.id
```

FOR SYSTEM_TIME AS OF a.proctime

意味着每当处理左表的一条消息时，都会根据条件到维表数据库查询数据，ON语句块指定关联查询条件，而a.proctime是左边的处理时间字段，字段名可以随意指定，但必须是processing time，当前不支持event time，也就是说这种方法不支持根据数据流的事件时间去查维度表里的对应时刻的数据。


### 使用说明

* 仅支持Blink planner
* 仅支持SQL，目前不支持Table API
* 目前不支持基于事件时间(event time)的temporal table join
* 维表可能会不断变化，JOIN行为发生后，维表中的数据发生了变化（新增、更新或删除），则已关联的维表数据不会被同步变化
* 维表和维表不能进行JOIN
* 维表必须指定主键。维表JOIN时，ON的条件必须包含所有主键的等值条件

### 缓存机制


按照上面的处理方式：每处理一条流里的消息，都要到数据库里查询维表，而维表一般存在第三方数据库，这就导致每次都要有远程请求，特别是数据流大的情况下，频繁的维表查询，也会对外部数据库造成很大压力、降低整体吞吐，所以对维度表进行缓存不失为一个好的策略。但是用缓存也有个潜在的风险：缓存里的数据有可能不是最新的，这要在性能和正确性之间做权衡。

LRU缓存

维表数据量大的情况下，可以用LRU算法缓存部分数据。

```
connector.lookup.cache.max-rows:缓存最大的记录条数，默认-1

connector.lookup.cache.ttl:缓存失效时间，默认-1
```


如果以上两个参数任意一个设置成-1，那就会禁用查询缓存。查询缓存底层用google guava cache实现.

在做join过程中(底层是在eval函数中实现的)，一旦某个关键字join维度表，就会将关键字作为key、关联查询维度表后的值作为value，放入到缓存中，在缓存没有失效之前，如果后面再次以该关键字join，会先在缓存里查找是否有对应的值，有就直接返回，没有再去访问外部数据系统，并把结果进行缓存。流程如下：


### 实现原理

如果要实现维表join，那定义的TableSource必须实现LookupableTableSource接口，当我们执行jdbc类型的create table sql语句时，BLink查询优化器会自动根据sql语句创建JDBCTableSource对象，同时该对象又实现了LookupableTableSource接口，因此我们创建的mysql表才支持维度查询。

LookupableTableSource 源码如下：

```java
/**
 * A {@link TableSource} which supports for lookup accessing via key column(s).
 * For example, MySQL TableSource can implement this interface to support lookup accessing.
 * When temporal join this MySQL table, the runtime behavior could be in a lookup fashion.
 *
 * @param <T> type of the result
 */
@Experimental
public interface LookupableTableSource<T> extends TableSource<T> {

	/**
	 * Gets the {@link TableFunction} which supports lookup one key at a time.
	 * @param lookupKeys the chosen field names as lookup keys, it is in the defined order
	 */
	TableFunction<T> getLookupFunction(String[] lookupKeys);

	/**
	 * Gets the {@link AsyncTableFunction} which supports async lookup one key at a time.
	 * @param lookupKeys the chosen field names as lookup keys, it is in the defined order
	 */
	AsyncTableFunction<T> getAsyncLookupFunction(String[] lookupKeys);

	/**
	 * Returns true if async lookup is enabled.
	 *
	 * <p>The lookup function returned by {@link #getAsyncLookupFunction(String[])} will be
	 * used if returns true. Otherwise, the lookup function returned by
	 * {@link #getLookupFunction(String[])} will be used.
	 */
	boolean isAsyncEnabled();
}
```

isAsyncEnabled:是否异步访问，如果异步访问使用异步方法，否则使用同步方法，当前不支持异步访问。JDBCTableSource只支持同步操作，


getAsyncLookupFunction:返回异步TableFunction函数，并发地处理多个请求和回复，从而连续的请求之间不需要阻塞等待。

getLookupFunction:返回同步TableFunction函数，同步访问外部数据库，即来一条数据，通过Key去查询外部数据库，等到返回数据输出关联结果，继续处理数据，这会对系统的吞吐率有影响


JDBCTableSource只支持同步look up模式，而TableFunction本质上是个UDTF(user-defined table function)：输入一条数据可能返回多条数据，也可能返回一条数据。eval则是TableFunction最重要的方法，当程序有一个输入元素时，就会调用eval一次，具体的业务逻辑实现就在这个方法里，最后将查询到的数据用collect()发送到下游。eval的核心代码：


源码见JDBCLookupFunction

缓存的创建
```java
@Override
	public void open(FunctionContext context) throws Exception {
		try {
			establishConnection();
			statement = dbConn.prepareStatement(query);
			this.cache = cacheMaxSize == -1 || cacheExpireMs == -1 ? null : CacheBuilder.newBuilder()
					.expireAfterWrite(cacheExpireMs, TimeUnit.MILLISECONDS)
					.maximumSize(cacheMaxSize)
					.build();
		} catch (SQLException sqe) {
			throw new IllegalArgumentException("open() failed.", sqe);
		} catch (ClassNotFoundException cnfe) {
			throw new IllegalArgumentException("JDBC driver class not found.", cnfe);
		}
	}
```

open 方法在进行初始化算子实例的进行调用，transient关键字主要的作用是告诉JVM，这个字段不需要序列化。

之所以建议很多能够在open函数里面初始化的变量用transient，是因为这些变量本身不太需要参与序列化，
比如一些cache之类的；或者有些变量也做不到序列化，比如一些连接相关的对象。




缓存的使用
```java
                statement.clearParameters();
				for (int i = 0; i < keys.length; i++) {
					JDBCUtils.setField(statement, keySqlTypes[i], keys[i], i);
				}
				try (ResultSet resultSet = statement.executeQuery()) {
					if (cache == null) {
						while (resultSet.next()) {
							collect(convertToRowFromResultSet(resultSet));
						}
					} else {
						ArrayList<Row> rows = new ArrayList<>();
						while (resultSet.next()) {
							Row row = convertToRowFromResultSet(resultSet);
							rows.add(row);
							collect(row);
						}
						rows.trimToSize();
						cache.put(keyRow, rows);
					}
				}
				break;
				
```


所以对于维度查询的实现逻辑也是在eval函数里，具体来说在JDBCLookupFunction类里，该函数实现了根据关键字段查找外部数据库以及数据缓存的功能，也就是说维度查询本质上是用UDTF实现的。

StreamExecLookupJoinRule规则解释器负责把表达式转换成物理执行计划，StreamExecLookupJoin把执行假话转换成Look up join。

Temporal Table.Join，是单流join，本质上是Lookup join：维表(被Lookup的表)作为被驱动表，左表每来一条消息，就用过滤关键词到数据库查询一次。temporal table join是轻量级的，不会保存状态数据。




### 如何自定义维表


main 函数
```java

public static void main(String[] args) throws Exception {
    StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
    EnvironmentSettings settings = EnvironmentSettings.newInstance().useBlinkPlanner().inStreamingMode().build();
    StreamTableEnvironment tableEnvironment = StreamTableEnvironment.create(env,settings);

    Properties properties = new Properties();
    properties.setProperty("bootstrap.servers", "localhost:9092");
    properties.setProperty("zookeeper.connect", "localhost:2181");
    properties.setProperty("group.id", "test");

    FlinkKafkaConsumer011<String> myConsumer = new FlinkKafkaConsumer011<String>("test", new SimpleStringSchema(),properties);
    DataStream<UserTest> source = env.addSource(myConsumer)
        .map(new MapFunction<String, UserTest>() {
            @Override
            public UserTest map(String s) throws Exception {
                UserTest userTest = new UserTest();
                userTest.setId(Integer.valueOf(s.split(",")[0]));
                userTest.setName(s.split(",")[1]);
                return userTest;
            }
        });

    tableEnvironment.registerDataStream("ubt",source,"id,name,proctime.proctime");

    MysqlAsyncLookupTableSource tableSource = MysqlAsyncLookupTableSource.Builder
        .newBuilder().withFieldNames(new String[]{"id","name"})
        .withFieldTypes(new TypeInformation[] {Types.INT,Types.STRING})
        .build();
    tableEnvironment.registerTableSource("info",tableSource);

    String sql = "select a.id,b.name from ubt as a join info FOR SYSTEM_TIME AS OF a.proctime as b on a.id=b.id";
    Table table = tableEnvironment.sqlQuery(sql);
    DataStream<Tuple2<Boolean, UserTest>> result = tableEnvironment.toRetractStream(table,UserTest.class);
    result.process(new ProcessFunction<Tuple2<Boolean,UserTest>, Object>() {
        @Override
        public void processElement(Tuple2<Boolean, UserTest> booleanUserTestTuple2, Context context, Collector<Object> collector) throws Exception {
            if (booleanUserTestTuple2.f0) {
                System.out.println(JSON.toJSONString(booleanUserTestTuple2.f1));
            }
        }
    });
    env.execute("");
}
```

MysqlAsyncLookupTableSource implements LookupableTableSource

```java
public static class MysqlAsyncLookupTableSource implements LookupableTableSource<UserTest> {

    private final String[] fieldNames;
    private final TypeInformation[] fieldTypes;

    public MysqlAsyncLookupTableSource(String[] fieldNames, TypeInformation[] fieldTypes) {
        this.fieldNames = fieldNames;
        this.fieldTypes = fieldTypes;
    }

    @Override
    public TableFunction<UserTest> getLookupFunction(String[] strings) {
        return null;
    }

    @Override
    public AsyncTableFunction<UserTest> getAsyncLookupFunction(String[] strings) {
        return MysqlAsyncLookupFunction.Builder.getBuilder()
            .withFieldNames(fieldNames)
            .withFieldTypes(fieldTypes)
            .build();
    }

    @Override
    public boolean isAsyncEnabled() {
        return true;
    }

    @Override
    public TableSchema getTableSchema() {
        return TableSchema.builder()
            .fields(fieldNames, TypeConversions.fromLegacyInfoToDataType(fieldTypes))
            .build();
    }

    public static final class Builder {
        private String[] fieldNames;
        private TypeInformation[] fieldTypes;

        private Builder() {
        }

        public static Builder newBuilder() {
            return new Builder();
        }

        public Builder withFieldNames(String[] fieldNames) {
            this.fieldNames = fieldNames;
            return this;
        }

        public Builder withFieldTypes(TypeInformation[] fieldTypes) {
            this.fieldTypes = fieldTypes;
            return this;
        }

        public MysqlAsyncLookupTableSource build() {
            return new MysqlAsyncLookupTableSource(fieldNames, fieldTypes);
        }
    }
}
```

MysqlAsyncLookupFunction extends AsyncTableFunction

```java

public static class MysqlAsyncLookupFunction extends AsyncTableFunction<UserTest> {
    private final String[] fieldNames;
    private final TypeInformation[] fieldTypes;

    public MysqlAsyncLookupFunction(String[] fieldNames, TypeInformation[] fieldTypes) {
        this.fieldNames = fieldNames;
        this.fieldTypes = fieldTypes;
    }

    // 每一条流数据都会调用此方法进行join
    public void eval(CompletableFuture<Collection<UserTest>> resultFuture, Object... keys) {
        // 进行维表查询
    }


    @Override
    public void open(FunctionContext context) {
        // 建立连接
    }

    @Override
    public void close(){}

    public static final class Builder {
        private String[] fieldNames;
        private TypeInformation[] fieldTypes;

        private Builder() {
        }

        public static Builder getBuilder() {
            return new Builder();
        }

        public Builder withFieldNames(String[] fieldNames) {
            this.fieldNames = fieldNames;
            return this;
        }

        public Builder withFieldTypes(TypeInformation[] fieldTypes) {
            this.fieldTypes = fieldTypes;
            return this;
        }

        public MysqlAsyncLookupFunction build() {
            return new MysqlAsyncLookupFunction(fieldNames, fieldTypes);
        }
    }
}
```

使用
```
select a.id,b.name 
from ubt as a 
join info FOR SYSTEM_TIME AS OF a.proctime as b 
on a.id=b.id
```

### Reference

http://apache-flink.147419.n8.nabble.com/flink-1-10-sql-td2486.html

https://liurio.github.io/2020/03/28/Flink%E6%B5%81%E4%B8%8E%E7%BB%B4%E8%A1%A8%E7%9A%84%E5%85%B3%E8%81%94/

[Flink CookBook-Table&Sql |维表Join原理解析](https://www.jianshu.com/p/189945244f79) 

https://juejin.im/post/6861595860782972941


https://ci.apache.org/projects/flink/flink-docs-release-1.11/zh/dev/table/sql/queries.html

https://ci.apache.org/projects/flink/flink-docs-release-1.11/zh/dev/table/streaming/temporal_tables.html


[Flink DataStream 关联维表实战](http://www.whitewood.me/2020/01/16/Flink-DataStream-%E5%85%B3%E8%81%94%E7%BB%B4%E8%A1%A8%E5%AE%9E%E6%88%98/)


[flink open 时候 transient使用问问题](http://apache-flink.147419.n8.nabble.com/flink-open-transient-td4186.html#a4187)