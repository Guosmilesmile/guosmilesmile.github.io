---
title: Flink 对象的不正确释放和内存泄露（反面教材
date: 2020-06-17 22:28:09
tags:
categories:
	- Flink
---


## 前言


今天在群里看到有人问了这么一个问题，在open里面开了一个定时任务，每隔一段时间更新下数据，手动cancel掉作业后，发现这个定时任务还在执行。

同一个时间，在perfma上看到一个挺有趣的内存泄露的排查记录，连接如下：

https://club.perfma.com/article/1555905


今天就来聊聊flink的对象不正确释放问题。


## TaskManager架构

看下Flink的整体的架构图


![image](https://note.youdao.com/yws/api/personal/file/81D17722B2304165A251569F0CBCB2BA?method=download&shareKey=761e56d216c0832f923753677def7816)


运行用户代码是在Flink的taskManager，TM的模型大体如下


![image](https://note.youdao.com/yws/api/personal/file/2F9F99093B4C44CDBEC9E22B691BFC54?method=download&shareKey=0fa604758cc8fb09e0fcec61a7cbf893)



虽然这种方式可以有效提高 CPU 利用率，但是个人不太喜欢这种设计，因为不仅缺乏资源隔离机制，同时也不方便调试。类似 Storm 的进程模型，一个JVM 中只跑该 Job 的 Tasks 实际应用中更为合理。

在这种模型下，用户代码不好控制，会出现很多神奇的现象。TM运行起来就是一个JVM。




## task的生命周期

在StreamTask的源码可以看到，具体的生命周期如下

```java
*  -- invoke()
*        |
*        +----> Create basic utils (config, etc) and load the chain of operators
*        +----> operators.setup()
*        +----> task specific init()
*        +----> initialize-operator-states()
*        +----> open-operators()
*        +----> run()
* --------------> mailboxProcessor.runMailboxLoop();
* --------------> StreamTask.processInput()
* --------------> StreamTask.inputProcessor.processInput()
* --------------> 间接调用 operator的processElement()和processWatermark()方法
*        +----> close-operators()
*        +----> dispose-operators()
*        +----> common cleanup
*        +----> task specific cleanup()
```


- 创建状态存储后端，为 OperatorChain 中的所有算子提供状态

- 加载 OperatorChain 中的所有算子

- 所有的 operator 调用 `setup`

- task 相关的初始化操作

- 所有 operator 调用 `initializeState` 初始化状态

- 所有的 operator 调用 `open`

- `run` 方法循环处理数据

- 所有 operator 调用 `close`

- 所有 operator 调用 `dispose`

- 通用的 cleanup 操作

- task 相关的 cleanup 操作


从生命周期可以看到，TM会自动调用用户的open和close方法来打来和关闭用户定义的资源和对象。


## 问题分析

再看看问题1，用户只定义了open中开启定时器，没有在close中关闭，哪怕cancel掉job，对于TM只是停止运行对应的用户代码，但是用户另外开启的线程没有办法回收，jvm也不会认为他是需要回收的垃圾。这个时候，需要用户在close方法种定义timer的close动作，将对应的操作关闭。



在看看问题2，内存的dump中出现大量的es client的实例，明显是用户自定义的es sink没有很好的回收掉client导致。参考下flink 原生的essink中的close方法

```java
    @Override
	public void close() throws Exception {
		if (bulkProcessor != null) {
			bulkProcessor.close();
			bulkProcessor = null;
		}

		if (client != null) {
			client.close();
			client = null;
		}

		callBridge.cleanup();

		// make sure any errors from callbacks are rethrown
		checkErrorAndRethrow();
	}
```

如果在job出现npe或者写es rejected等异常时，job会被flink重启，如果用户代码灭有很好的close这些资源，就会频繁调用open构造出新的对象，资源，线程等等。


而且案例中，用户的单例模式还写的有问题。虽然有double check，但是没加volatile，同时锁的是this, 而不是类。


#### 为什么单例模式要使用volatile

因为指令重排

uniqueInstance = new Singleton(); 这段代码其实是分为三步执行：
1. 为 uniqueInstance 分配内存空间
2. 初始化 uniqueInstance
3. 将 uniqueInstance 指向分配的内存地址


但是由于 JVM 具有指令重排的特性，执行顺序有可能变成 1->3->2。指令重排在单线程环境下不会出现问题，但是在多线程环境下会导致一个线程获得还没有初始化的实例。例如，线程 T1 执行了 1 和 3，此时 T2 调用 getUniqueInstance() 后发现 uniqueInstance 不为空，因此返回 uniqueInstance，但此时 uniqueInstance 还未被初始化。

使用 volatile 可以禁止 JVM 的指令重排，保证在多线程环境下也能正常运行。


而且锁的是this，锁的是这个实例，而不是类，那么这个client也会多次创建。



### 多线程模型和多进程模型

多进程模型，每个task是一个进程，如果作业job进程就会跟着kill，可以很好的对资源进行控制和隔离

多线程模型，可以更好的利用cpu，毕竟多进程间的数据共享等等都会产生cpu消耗。但是像flink这样TM一个jvm，如果job混合部署，一个job异常导致jvm崩溃，会连坐另一个job，如果一个job到处创建资源不释放，很容易影响整个集群。因此flink最好部署在yarn或者kubernetes上，一个job一个集群，实现隔离。

### 堆外内存

Flink大量使用堆外内存，堆外内存的使用可以提高计算能力，不用担心gc问题。

Heap 空间可以认为是 JVM 向 OS 申请好的一段连续内存。Java 对象 new 的时候是从这段 JVM 已经申请的内存中划分出一部分，GC
时对象 finalize 也是将内存还给 JVM，并不会真的像 OS 去释放内存。

Direct/Native 内存则是直接向 OS 申请的内存。持有该内存的对象在 finalize 的时候必须向 OS 释放这段内存，否则GC是无法自动释放该内存的，就会造成泄漏。
Direct 内存相比 Native 内存的区别主要有两点，一是申请时 JVM 会检查 MaxDirectMemorySize，二是 JVM 会保证DirectByteBuffer 被销毁的时候会向 OS 去释放这段内存。
Native 内存需要我们自己保证内存的释放，在 Flink 中由于申请到的 Native 内存也是封装在 DirectByteBuffer
里的，所以这部分内存的释放是通过给 DirectByteBuffer 设置 cleaner 实现的。详见
`MemorySegmentFactory#allocateOffHeapUnsafeMemory`


那如果JVM因为异常挂掉或者因为内存超用被kill，会出现无法调用finalize向os释放这部分内存，导致新拉起的TM在该container中的可用堆外内存并没有那么多，资源不够导致雪崩继续挂，再拉起再挂。


资源隔离和资源回收就显得很重要 


### Reference

http://apache-flink.147419.n8.nabble.com/flink-1-10-td2934.html#a2971
