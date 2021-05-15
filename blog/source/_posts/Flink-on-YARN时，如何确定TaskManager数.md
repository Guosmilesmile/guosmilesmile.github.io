---
title: Flink on YARN时，如何确定TaskManager数
date: 2021-05-15 15:02:27
tags:
categories: Flink
---




### Reference

https://www.jianshu.com/p/5b670d524fa5

### 答案

Job的最大并行度除以每个TaskManager分配的任务槽数。

### 问题

在[Flink 1.5 Release Notes](https://ci.apache.org/projects/flink/flink-docs-release-1.9/release-notes/flink-1.5.html)中，有这样一段话

```
Update Configuration for Reworked Job Deployment
Flink’s reworked cluster and job deployment component improves the integration with resource managers and enables dynamic resource allocation. One result of these changes is, that you no longer have to specify the number of containers when submitting applications to YARN and Mesos. Flink will automatically determine the number of containers from the parallelism of the application.

Although the deployment logic was completely reworked, we aimed to not unnecessarily change the previous behavior to enable a smooth transition. Nonetheless, there are a few options that you should update in your conf/flink-conf.yaml or know about.

The allocation of TaskManagers with multiple slots is not fully supported yet. Therefore, we recommend to configure TaskManagers with a single slot, i.e., set taskmanager.numberOfTaskSlots: 1
If you observed any problems with the new deployment mode, you can always switch back to the pre-1.5 behavior by configuring mode: legacy.
Please report any problems or possible improvements that you notice to the Flink community, either by posting to a mailing list or by opening a JIRA issue.

Note: We plan to remove the legacy mode in the next release.
```

这说明从1.5版本开始，Flink on YARN时的容器数量——亦即TaskManager数量——将由程序的并行度自动推算，也就是说flink run脚本的-yn/--yarncontainer参数不起作用了。那么自动推算的规则是什么呢？要弄清楚它，先来复习Flink的并行度（Parallelism）和任务槽（Task Slot）。


### 并行度（Parallelism）


与Spark类似地，一个Flink Job在生成执行计划时也划分成多个Task。Task可以是Source、Sink、算子或算子链（算子链有点意思，之后会另写文章详细说的）。Task可以由多线程并发执行，每个线程处理Task输入数据的一个子集。而并发的数量就称为Parallelism，即并行度。

Flink程序中设定并行度有4种级别，从低到高分别为：算子级别、执行环境（ExecutionEnvironment）级别、客户端（命令行）级别、配置文件（flink-conf.yaml）级别。实际执行时，优先级则是反过来的，算子级别最高。简单示例如下。


算子级别

```
dataStream.flatMap(new SomeFlatMapFunction()).setParallelism(4);
```

执行环境级别

```
streamExecutionEnvironment.setParallelism(4);
```

命令行级别


```
bin/flink -run --parallelism 4 example-0.1.jar

bin/flink -run -p 4 example-0.1.jar

```

flink-conf.yaml级别

```
parallelism.default: 4
```

### 任务槽（Task Slot）


Flink运行时由两个组件组成：JobManager与TaskManager，与Spark Standalone模式下的Master与Worker是同等概念。从官网抄来的图如下所示，很容易理解。

![image](https://note.youdao.com/yws/api/personal/file/12C3D15A4E7B424792C50837CD6EAF8E?method=download&shareKey=278932d58b2f39a6b0a74e01844186b6)

JobManager和TaskManager本质上都是JVM进程。为了提高Flink程序的运行效率和资源利用率，Flink在TaskManager中实现了任务槽（Task Slot）。任务槽是Flink计算资源的基本单位，每个任务槽可以在同一时间执行一个Task，而TaskManager可以拥有一个或者多个任务槽。

任务槽可以实现TaskManager中不同Task的资源隔离，不过是逻辑隔离，并且只隔离内存，亦即在调度层面认为每个任务槽“应该”得到taskmanager.heap.size的N分之一大小的内存。CPU资源不算在内。

TaskManager的任务槽个数在使用flink run脚本提交on YARN作业时用-ys/--yarnslots参数来指定，另外在flink-conf.yaml文件中也有默认值taskManager.numberOfTaskSlots。一般来讲，我们设定该参数时可以将它理解成一个TaskManager可以利用的CPU核心数，因此也要根据实际情况（集群的CPU资源和作业的计算量）来确定。


### 确定TaskManager数


以Flink自带示例中简化的WordCount程序为例

```
 // 执行环境并行度设为6
    env.setParallelism(6);
    // Source并行度为1
    DataStream<String> text = env
      .readTextFile(params.get("input"))
      .setParallelism(1);
    DataStream<Tuple2<String, Integer>> counts = text
      .flatMap(new Tokenizer())
      .keyBy(0)
      .sum(1);
    counts.print();
```

用--yarnslots 3参数来执行，即每个TaskManager分配3个任务槽。TaskManager、任务槽和任务的分布将如下图所示，方括号内的数字为并行线程的编号。


