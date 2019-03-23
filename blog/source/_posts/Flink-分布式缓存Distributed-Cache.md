---
title: Flink 分布式缓存Distributed Cache
date: 2019-03-21 22:48:41
tags:
categories: Flink
---


### 不管是流式还是批处理都可以使用
Flink提供了一个分布式缓存，类似于Apache Hadoop，可以在本地访问用户函数的并行实例。此函数可用于共享包含静态外部数据的文件，如字典或机器学习的回归模型。

缓存的工作原理如下。程序在其作为缓存文件的特定名称下注册本地或远程文件系统（如HDFS或S3）的文件或目录ExecutionEnvironment。执行程序时，Flink会自动将文件或目录复制到所有工作程序的本地文件系统。用户函数可以查找指定名称下的文件或目录，并从worker的本地文件系统访问它。


```java
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

// register a file from HDFS
env.registerCachedFile("hdfs:///path/to/your/file", "hdfsFile")

// register a local executable file (script, executable, ...)如果是可执行文件或者脚本，就多一个参数
env.registerCachedFile("file:///path/to/exec/file", "localExecFile", true)
.env.registerCachedFile("D://file.txt", "localExecFile")

// define your program and execute
...
DataSet<String> input = ...
DataSet<Integer> result = input.map(new MyMapper());
...
env.execute();
```



访问用户函数中的缓存文件或目录（此处为a MapFunction）。该函数必须扩展RichFunction类，因为它需要访问RuntimeContext。

```java
// extend a RichFunction to have access to the RuntimeContext
public final class MyMapper extends RichMapFunction<String, Integer> {

    @Override
    public void open(Configuration config) {

      // access cached file via RuntimeContext and DistributedCache
      File myFile = getRuntimeContext().getDistributedCache().getFile("hdfsFile");
      // read the file (or navigate the directory)
      ...
    }

    @Override
    public Integer map(String value) throws Exception {
      // use content of cached file
      ...
    }
}
```

