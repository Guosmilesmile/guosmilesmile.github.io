---
title: Flink 1.10 内存模型和配置（翻译）
date: 2020-02-13 15:31:28
tags:
categories:
	- Flink
---



## 设置任务执行程序内存

https://ci.apache.org/projects/flink/flink-docs-release-1.10/ops/memory/mem_setup.html

Flink JVM进程的总进程内存包括由Flink应用程序消耗的内存(total Flink memory)和运行该进程的jvm消耗的内存。总Flink内存消耗包括JVM堆( JVM heap)、managed memory(由Flink管理)和堆外(或者 native)内存的使用。

![image](https://note.youdao.com/yws/api/personal/file/9473A52BC0664A23B95CADB8F933F782?method=download&shareKey=a15663241503e03f23b84dfce92709e4)

如果是运行在非本地，那么最简单的构建一个集群，在Flink中设置内存的最简单方法是配置以下两个选项之一：

```
Total Flink memory (taskmanager.memory.flink.size)
Total process memory (taskmanager.memory.process.size)
```
其余的内存组件将根据默认值或额外配置的选项自动调整.[该链接](https://ci.apache.org/projects/flink/flink-docs-release-1.10/ops/memory/mem_detail.html#detailed-memory-model)是关于其他内存组件的更多细节。

在standalone模式下，更适合的配置是Total Flink memory (taskmanager.memory.flink.size)，声明有多少内存分配给Flink本身。总Flink内存分为JVM堆、托管内存大小和直接内存。

如果在容器中部署，那么应该配置Total process memory (taskmanager.memory.process.size),它声明应该为Flink JVM进程分配多少内存，等价于整个容器的大小。

**在容器中，taskmanager.memory.process.size包含了taskmanager.memory.flink.size的内存，等于taskmanager.memory.flink.size+JVM Metaspace+ JVM Overhead**

另一种设置内存的方法是设置任务堆和托管内存(taskmanager.memory.task.heap)。大小和taskmanager.memory.managed.size)。[这里](https://ci.apache.org/projects/flink/flink-docs-release-1.10/ops/memory/mem_setup.html#configure-heap-and-managed-memory)更详细地描述了这种更细粒度的方法。


==note== 必须使用上面提到的三种方法之一来配置Flink的内存(本地执行除外)，否则Flink启动将失败，这意味着必须显式地配置下列没有默认值的选项子集之一：

```
taskmanager.memory.flink.size
taskmanager.memory.process.size
taskmanager.memory.task.heap.size and taskmanager.memory.managed.size
```

==note==  不推荐显示的同时配置 total process memory 和 total Flink memory，由于潜在的内存配置冲突，它可能导致部署失败。其他内存组件的额外配置也需要谨慎，因为它可能会产生更多的配置冲突。


## 配置堆和托管内存(Configure Heap and Managed Memory)

正如前面在总内存描述中提到的，在Flink中设置内存的另一种方法是显式地指定任务堆和托管内存。它为Flink的任务和托管内存提供了对可用JVM堆的更多控制。

其余的内存组件将根据默认值或额外配置的选项自动调整。[下面](https://ci.apache.org/projects/flink/flink-docs-release-1.10/ops/memory/mem_detail.html#detailed-memory-model)是关于其他内存组件的更多细节。



## 任务(操作符)堆内存 (Task (Operator) Heap Memory)

如果您想确保您的用户代码有一定数量的JVM堆可用，您可以显式地设置任务堆内存(taskmanager.memory.task.heap.size).它将被添加到JVM堆大小中，并将专用于运行用户代码的Flink操作符。

## 托管内存 (Managed Memory)

托管内存由Flink管理，并被分配为native memory(堆外).以下工作负载使用托管内存:
* 流作业可以将其用于RocksDB状态后端。
* 批处理作业可以使用它对中间结果进行排序、散列表和缓存。

托管内存的大小可以是
* 可以通过taskmanager.memory. management .size显式配置
* 或通过taskmanager.memory. management .fraction计算total Flink memory的一部分。

如果两个都设置，大小会覆盖占比这个参数。如果大小和占比都没有设置，会采用默认的占比

## 配置堆外内存 (Configure Off-Heap Memory (direct or native))

由用户代码分配的堆外内存应该在task off-heap memory中(taskmanager.memory.task.off-堆.size)。


==note==您还可以调整框架的堆外内存。此选项是高级的，仅当您确定Flink框架需要更多内存时才建议更改.


Flink将framework off-heap memory和task off-heap memory包含到JVM的 direct memory限制中(-XX:MaxDirectMemorySize = Framework + Task Off-Heap + Network Memory)，[JVM参数](https://ci.apache.org/projects/flink/flink-docs-release-1.10/ops/memory/mem_detail.html#jvm-parameters)配置如下。


----------------------


## 详细内存模型 Detailed Memory Model

![image](https://note.youdao.com/yws/api/personal/file/74E93C992E78417FAC24C8E603D2DF72?method=download&shareKey=5f4c2cd2feeec20b8de565f23d0a81e7)




Component  	|  Configuration options  		|  Description  
---|---|---
Framework Heap Memory	|	taskmanager.memory.framework.heap.size	|	JVM heap memory dedicated to Flink framework (advanced option)
Task Heap Memory	|	taskmanager.memory.task.heap.size	|	JVM heap memory dedicated to Flink application to run operators and user code
Managed memory 	|	taskmanager.memory.managed.size<br>	 taskmanager.memory.managed.fraction |	Native memory managed by Flink, reserved for sorting, hash tables, caching of intermediate results and RocksDB state backend
Framework Off-heap Memory 	|	taskmanager.memory.framework.off-heap.size		| Off-heap direct (or native) memory dedicated to Flink framework (advanced option)
Task Off-heap Memory	|	taskmanager.memory.task.off-heap.size	|	Off-heap direct (or native) memory dedicated to Flink application to run operators
Network Memory		| taskmanager.memory.network.min<br> taskmanager.memory.network.max <br> taskmanager.memory.network.fraction		| Direct memory reserved for data record exchange between tasks (e.g. buffering for the transfer over the network), it is a capped fractionated component of the total Flink memory
JVM metaspace		|taskmanager.memory.jvm-metaspace.size		|Metaspace size of the Flink JVM process
JVM Overhead	|	taskmanager.memory.jvm-overhead.min<br> taskmanager.memory.jvm-overhead.max<br> taskmanager.memory.jvm-overhead.fraction	|	Native memory reserved for other JVM overhead: e.g. thread stacks, code cache, garbage collection space etc, it is a capped fractionated component of the total process memory


## Framework Memory

如果没有充分的理由，不应该更改框架堆内存和框架堆外内存选项.只有当您确定Flink需要更多内存来处理某些内部数据结构或操作时，才需要调整它们.它可以与特定的部署环境或作业结构相关，比如高并行性。此外，在某些设置中，诸如Hadoop之类的Flink依赖项可能会消耗更多的直接内存或本机内存。

## 限制分离组件 Capped Fractionated Components

本节描述以下选项的配置细节，这些选项可能是总内存的一部分
* Network memory 是 the total Flink memory的一部分
* JVM overhead 是 the total process memory 的一部分

例如，如果只设置以下内存选项:

```
total Flink memory = 1000Mb,
network min = 64Mb,
network max = 128Mb,
network fraction = 0.1
```

那么network memory 将是1000Mb x 0.1 = 100Mb，在64-128Mb范围内。
注意，如果您配置相同的最大值和最小值，这实际上意味着它的大小固定在该值上。

如果组件内存没有显式配置，那么Flink将使用分数根据总内存计算内存大小。计算值的上限是对应的最小/最大选项

```
total Flink memory = 1000Mb,
network min = 128Mb,
network max = 256Mb,
network fraction = 0.1
```

那么network memory 就会是128Mb，因为分数得出的大小是100Mb，小于最小值。


如果定义了总内存及其其他组件的大小，也可能忽略该分数。在这种情况下network memory是总内存的剩余部分

```
total Flink memory = 1000Mb,
task heap = 100Mb,
network min = 64Mb,
network max = 256Mb,
network fraction = 0.1
```

所有其他组件都有默认值，包括默认的托管内存分数。那么网络内存不是分数占比(1000Mb x 0.1 = 100Mb)，而是整个Flink内存的其余部分，它们要么在64-256Mb范围内，要么失败。


## JVM Parameters


  JVM Arguments |  	  Value 
---|---
-Xmx and -Xms| 	Framework + Task Heap Memory
-XX:MaxDirectMemorySize	| Framework + Task Off-Heap + Network Memory
-XX:MaxMetaspaceSize| 	JVM Metaspace


--------------------

## 内存调优指南 Memory tuning guide

https://ci.apache.org/projects/flink/flink-docs-release-1.10/ops/memory/mem_tuning.html#configure-memory-for-containers


### 为独立部署配置内存 Configure memory for standalone deployment

建议为独立部署配置总Flink内存(taskmanager.memory.flink.size)或它的组件，以便声明给Flink本身多少内存.此外，如果JVM元空间导致[问题](https://ci.apache.org/projects/flink/flink-docs-release-1.10/ops/memory/mem_trouble.html#outofmemoryerror-metaspace)，您可以调整它。

total Process memory是不相关的,因为JVM overhead不是由Flink或部署环境控制的，在这种情况下，只有执行机器的物理资源起作用。

### 为容器配置内存 Configure memory for containers

建议为容器化部署(Kubernetes、Yarn或Mesos)配置总进程内存(taskmanager.memory.process.size).它声明应该为Flink JVM进程分配多少内存，并与请求容器的大小相对应.

==note== 如果配置的是total Flink memory ，那么flink会隐式的加上JVM memory得到一个派生内存，然后以该派生内存去申请容器。

```
Warning:如果Flink或用户代码分配了超出容器大小的off-heap (native) memory，作业可能会失败，因为部署环境可能会结束有问题的容器。

```

请参阅容器内存超出故障的[说明](https://ci.apache.org/projects/flink/flink-docs-release-1.10/ops/memory/mem_trouble.html#container-memory-exceeded)。

### 为状态后端配置内存 Configure memory for state backends

在部署Flink流应用程序时，所使用的状态后端类型将决定集群的最佳内存配置。

#### 堆状态后端 Heap state backend

当运行无状态作业或使用堆状态后端(memorystate后端或fsstate后端)时，将托管内存设置为零。这将确保为JVM上的用户代码分配最大的内存量。

### RocksDB state backend

The RocksDBStateBackend 使用 native memory。默认情况下，将native memory 分配到managed memory的大小的为RocksDB的限制.因此，为您的状态用例保留足够的托管内存非常重要.如果禁用默认的RocksDB内存控制，如果RocksDB分配的内存超过请求的容器大小限制(the total process memory)，任务执行器可能会在容器化部署中被杀死.参见如何[调优RocksDB内存](https://ci.apache.org/projects/flink/flink-docs-release-1.10/ops/state/large_state_tuning.html#tuning-rocksdb-memory)和[state.backend.rocksdb.memory.managed](https://ci.apache.org/projects/flink/flink-docs-release-1.10/ops/config.html#state-backend-rocksdb-memory-managed).


### 为批处理作业配置内存 Configure memory for batch jobs

Flink的批处理操作符利用托管内存来更有效地运行，有些操作可以直接对原始数据执行，而不必反序列化为Java对象.这意味着托管内存配置对应用程序的性能有实际影响。Flink将尝试为批处理作业分配和使用尽可能多的托管内存，但不会超出其限制。这可以防止OutOfMemoryError错误，因为Flink精确地知道它需要利用多少内存。如果托管内存不够，Flink会优雅地溢出到磁盘。


--------------
## Troubleshooting

https://ci.apache.org/projects/flink/flink-docs-release-1.10/ops/memory/mem_trouble.html


### IllegalConfigurationException

如果你看到一个从TaskExecutorProcessUtils抛出的IllegalConfigurationException，它通常表示存在一个无效的配置值(例如负内存大小、大于1的分数等)或者配置冲突。

### OutOfMemoryError: Java heap space


异常通常表示JVM堆太小。您可以尝试通过增加 total memory 或[task heap memory](https://ci.apache.org/projects/flink/flink-docs-release-1.10/ops/memory/mem_setup.html#task-operator-heap-memory)来增加JVM堆大小。

==Note== 您还可以增加framework heap memory ，但是这个选项是高级的，只有在您确定Flink框架本身需要更多内存时才应该更改


### OutOfMemoryError: Direct buffer memory

异常通常表示JVM直接内存限制太小，或者存在直接内存泄漏。检查用户代码或其他外部依赖项是否使用了JVM直接内存.您可以尝试通过调整 direct off-heap memory存来增加它的限制。还请参阅如何配置[堆外内存](https://ci.apache.org/projects/flink/flink-docs-release-1.10/ops/memory/mem_setup.html#configure-off-heap-memory-direct-or-native)和JVM参数(由Flink设置)

### OutOfMemoryError: Metaspace

异常通常表示 JVM metaspace限制配置得太小。您可以尝试增加 JVM metaspace选项。

### IOException: Insufficient number of network buffers


异常通常表示配置的 network memory不够大.您可以尝试通过调整以下选项来增加网络内存:
```
taskmanager.memory.network.min
taskmanager.memory.network.max
taskmanager.memory.network.fraction
```


### Container Memory Exceeded

如果任务执行器容器试图分配超出其请求大小的内存(Yarn, Mesos or Kubernetes)，这通常表示Flink没有保留足够的本机内存。您可以通过使用外部监视系统或在容器被部署环境杀死时从错误消息观察到这一点


如果使用了rocksdbstateback，并且禁用了内存控制，则可以尝试增加托管内存。或者，您可以增加 JVM overhead.

------------
## 迁移指南 Migration Guide

https://ci.apache.org/projects/flink/flink-docs-release-1.10/ops/memory/mem_migration.html

==Note==在1.10版本之前，Flink根本不需要设置与内存相关的选项，因为它们都有默认值.新的内存配置要求至少有一个子集的以下选项是显式配置，否则配置将失败:

```
taskmanager.memory.flink.size
taskmanager.memory.process.size
taskmanager.memory.task.heap.size and taskmanager.memory.managed.size
```


### 配置选项的更改 Changes in Configuration Options



Deprecated option| 	Interpreted as 
---|---
taskmanager.heap.size | taskmanager.memory.flink.size (standalone部署) <br> taskmanager.memory.process.size (容器化部署)
taskmanager.memory.size | taskmanager.memory.managed.size
taskmanager.network.memory.min | taskmanager.memory.network.min
taskmanager.network.memory.max |taskmanager.memory.network.max
taskmanager.network.memory.fraction | taskmanager.memory.network.fraction

### 总内存(以前是堆内存) Total Memory (Previously Heap Memory)

taskmanager.heap.size or taskmanager.heap.mb ，除了命名之外，它们还包括JVM堆和其他堆外内存组件。这些选项已经被弃用


### JVM Heap Memory

以前JVM堆内存包括托管内存(如果配置为在堆上)和其他包括使用堆内存的组件。现在，如果只配置了总Flink内存或总进程内存，那么JVM堆的大小为总内存中减去所有其他组件后剩下的部分。

另外，您现在可以更直接地控制分配给操作符任务(taskmanager.memory.task.heap.size)的JVM堆，参见Task(操作符)堆内存。JVM堆内存也用于堆状态后端(如果选择memorystate后端，则为memorystate后端或fsstate后端，用于流作业)。

### RocksDB state

如果为流作业选择了rocksdbstateback，那么它的本机内存消耗现在应该计入托管内存中。
RocksDB内存分配受到托管内存大小的限制。您可以通过设置state.backend.rocksdb.memory来禁用RocksDB内存控件


### Container Cut-Off Memory

容器的部署，需要预留一部分内存，这部分内存主要是留给非Flink控制的依赖项，例如RocksDB、JVM内部等等。新的内存模型引入了更具体的内存组件(将进一步描述)来解决这些问题

在使用rocksdbstateback的流作业中，RocksDB本机内存消耗现在应该作为托管内存的一部分来考虑。RocksDB内存分配也受到已配置的托管内存大小的限制。


其他直接或本机堆外内存消费者现在可以通过以下新的配置选项来解决:

```
Task off-heap memory (taskmanager.memory.task.off-heap.size)
Framework off-heap memory (taskmanager.memory.framework.off-heap.size)
JVM metaspace (taskmanager.memory.jvm-metaspace.size)
JVM overhead (see also detailed new memory model)
```
