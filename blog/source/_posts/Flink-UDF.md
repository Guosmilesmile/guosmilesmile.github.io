---
title: Flink-UDF
date: 2019-03-17 21:24:32
tags:
categories:
  - 流式计算 
  - Flink
---


### Scalar Functions 标量函数

标量函数，是指返回一个值的函数。标量函数是实现将0，1，或者多个标量值转化为一个新值。    
实现一个标量函数需要继承ScalarFunction，并且实现一个或者多个evaluation方法。标量函数的行为就是通过evaluation方法来实现的。evaluation方法必须定义为public，命名为eval。evaluation方法的输入参数类型和返回值类型决定着标量函数的输入参数类型和返回值类型。evaluation方法也可以被重载实现多个eval。同时evaluation方法支持变参数，例如：eval(String... strs)。

切割一条流，切割出域名
```java
public static class SplitScalar extends ScalarFunction {
        private String separator = " ";

        @Override
        public void open(FunctionContext context) throws Exception {
            super.open(context);
        }

        public SplitScalar(String separator) {
            this.separator = separator;
        }

        public String eval(String str) {
            for (String s : str.split(separator)) {
                return s;
            }
            return "";
        }
    }
```

```java
调用方法
  String fields = "channel,count,statisticTime,proctime.proctime";
  String fieldsTypes = "string,long,long";
  String[] strings = StringUtils.splitByWholeSeparator(fieldsTypes, ",");
  TypeInformation[] types = new TypeInformation[]{Types.STRING, Types.STRING, Types.LONG, Types.LONG};
  
  tableEnv.registerFunction("SplitScalar", new SplitScalar("/"));

  tableEnv.registerDataStream("playerTable", playerCountEventDataStream, fields);

  Table table = tableEnv.sqlQuery("select channel,SplitScalar(channel), sum(`count`) as `count`, statisticTime from playerTable group by channel,statisticTime,TUMBLE(proctime, INTERVAL '5' SECOND)");
       
```


## Table Functions 表函数

与标量函数相似之处是输入可以0，1，或者多个参数，但是不同之处可以输出任意数目的行数。返回的行也可以包含一个或者多个列。    

为了自定义表函数，需要继承TableFunction，实现一个或者多个evaluation方法。表函数的行为定义在这些evaluation方法内部，函数名为eval并且必须是public。TableFunction可以重载多个eval方法。Evaluation方法的输入参数类型，决定着表函数的输入类型。Evaluation方法也支持变参，例如：eval(String... strs)。返回表的类型取决于TableFunction的基本类型。Evaluation方法使用collect(T)发射输出rows。      

在Table API中，表函数在scala语言中使用方法如下：.join(Expression) 或者 .leftOuterJoin(Expression)，在java语言中使用方法如下：.join(String) 或者.leftOuterJoin(String)。

Join操作算子会使用表函数(操作算子右边的表)产生的所有行进行(cross) join 外部表(操作算子左边的表)的每一行。


leftOuterJoin操作算子会使用表函数(操作算子右边的表)产生的所有行进行(cross) join 外部表(操作算子左边的表)的每一行，并且在表函数返回一个空表的情况下会保留所有的outer rows。

```java
 public static class Split extends TableFunction<Tuple2<String, Integer>> {
        private String separator = " ";

        public Split(String separator) {
            this.separator = separator;
        }

        public void eval(String str) {
            for (String s : str.split(separator)) {
                // use collect(...) to emit a row
                collect(new Tuple2<String, Integer>(s, s.length()));
            }
        }
    }

```


```java
调用方法
  String fields = "channel,count,statisticTime,proctime.proctime";
  String fieldsTypes = "string,long,long";
  String[] strings = StringUtils.splitByWholeSeparator(fieldsTypes, ",");
  TypeInformation[] types = new TypeInformation[]{Types.STRING, Types.STRING, Types.LONG, Types.LONG};
  
  tableEnv.registerFunction("SplitScalar", new SplitScalar("/"));

  tableEnv.registerDataStream("playerTable", playerCountEventDataStream, fields);

  Table table = tableEnv.sqlQuery("select channel, ones, twos from playerTable LEFT JOIN LATERAL TABLE(split(channel)) as T(ones, twos) ON TRUE");
       
```


### 差异
标量函数和表函数的差异在于sql的写法上