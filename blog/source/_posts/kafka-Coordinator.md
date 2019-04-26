---
title: kafka Coordinator
date: 2019-03-21 22:25:51
tags:
categories: Kafka
---

### Rebalance
rebalance本质上是一种协议，规定了一个consumer group下的所有consumer如何达成一致来分配订阅topic的每个分区。比如某个group下有20个consumer，它订阅了一个具有100个分区的topic。正常情况下，Kafka平均会为每个consumer分配5个分区。这个分配的过程就叫rebalance。

#### 什么时候rebalance？

这也是经常被提及的一个问题。rebalance的触发条件有三种：
* 组成员发生变更(新consumer加入组、已有consumer主动离开组或已有consumer崩溃了——这两者的区别后面会谈到)
* 订阅主题数发生变更——这当然是可能的，如果你使用了正则表达式的方式进行订阅，那么新建匹配正则表达式的topic就会触发rebalance
* 订阅主题的分区数发生变更

#### 如何进行组内分区分配？

之前提到了group下的所有consumer都会协调在一起共同参与分配，这是如何完成的？Kafka新版本consumer默认提供了两种分配策略：range和round-robin。当然Kafka采用了可插拔式的分配策略，你可以创建自己的分配器以实现不同的分配策略。实际上，由于目前range和round-robin两种分配器都有一些弊端，Kafka社区已经提出第三种分配器来实现更加公平的分配策略，只是目前还在开发中。我们这里只需要知道consumer group默认已经帮我们把订阅topic的分区分配工作做好了就行了。

简单举个例子，假设目前某个consumer group下有两个consumer： A和B，当第三个成员加入时，kafka会触发rebalance并根据默认的分配策略重新为A、B和C分配分区，如下图所示：
![image](https://note.youdao.com/yws/api/personal/file/0B40B5900E534F1FB0B680D6728725AF?method=download&shareKey=c1fc693fc38bc82ffebf6d9b4cbdb1b5)

### 谁来执行rebalance和consumer group管理？              
Kafka提供了一个角色：coordinator来执行对于consumer group的管理。坦率说kafka对于coordinator的设计与修改是一个很长的故事。最新版本的coordinator也与最初的设计有了很大的不同。这里我只想提及两次比较大的改变。

首先是0.8版本的coordinator，那时候的coordinator是依赖zookeeper来实现对于consumer group的管理的。Coordinator监听zookeeper的/consumers/<group>/ids的子节点变化以及/brokers/topics/<topic>数据变化来判断是否需要进行rebalance。group下的每个consumer都自己决定要消费哪些分区，并把自己的决定抢先在zookeeper中的/consumers/<group>/owners/<topic>/<partition>下注册。很明显，这种方案要依赖于zookeeper的帮助，而且每个consumer是单独做决定的，没有那种“大家属于一个组，要协商做事情”的精神。

![image](https://note.youdao.com/yws/api/personal/file/BAF03AAB320A40CDACF9300CFE19A089?method=download&shareKey=621a95a9cdac6030e304faad474f76d2)

基于这些潜在的弊端，0.9版本的kafka改进了coordinator的设计，提出了group coordinator——每个consumer group都会被分配一个这样的coordinator用于组管理和位移管理。这个group coordinator比原来承担了更多的责任，比如组成员管理、位移提交保护机制等。当新版本consumer group的第一个consumer启动的时候，它会去和kafka server确定谁是它们组的coordinator。之后该group内的所有成员都会和该coordinator进行协调通信。显而易见，这种coordinator设计不再需要zookeeper了，性能上可以得到很大的提升。后面的所有部分我们都将讨论最新版本的coordinator设计。

### Coordinator介绍
Coordinator一般指的是运行在broker上的group Coordinator，用于管理Consumer Group中各个成员，每个KafkaServer（broker）都有一个GroupCoordinator实例，管理多个消费者组，主要用于offset位移管理和Consumer Rebalance。


###  如何确定coordinator？

那么consumer group如何确定自己的coordinator是谁呢？ 简单来说分为两步：
* 确定consumer group位移信息写入__consumers_offsets的哪个分区。具体计算公式：
　　__consumers_offsets partition# = Math.abs(groupId.hashCode() % groupMetadataTopicPartitionCount)   注意：groupMetadataTopicPartitionCount由offsets.topic.num.partitions指定，默认是50个分区。

* 该分区leader所在的broker就是被选定的coordinator     
###### 等于每一个groupid都有一个自己对应的broker作为coordinator。


#### Rebalance Generation

JVM GC的分代收集就是这个词(严格来说是generational)，我这里把它翻译成“届”好了，它表示了rebalance之后的一届成员，主要是用于保护consumer group，隔离无效offset提交的。比如上一届的consumer成员是无法提交位移到新一届的consumer group中。我们有时候可以看到ILLEGAL_GENERATION的错误，就是kafka在抱怨这件事情。每次group进行rebalance之后，generation号都会加1，表示group进入到了一个新的版本，如下图所示： Generation 1时group有3个成员，随后成员2退出组，coordinator触发rebalance，consumer group进入Generation 2，之后成员4加入，再次触发rebalance，group进入Generation 3.     
![image](https://note.youdao.com/yws/api/personal/file/194D34CFF96A4737932CE3656E2AA265?method=download&shareKey=83275b32c8e5756df4dcb629c99a01f7)


#### Rebalance过程
rebalance的前提是coordinator已经确定。    
rebalance分为2步：Join和Sync
1. Join， 顾名思义就是加入组。这一步中，所有成员都向coordinator发送JoinGroup请求，请求入组。一旦所有成员都发送了JoinGroup请求，coordinator会从中选择一个consumer担任leader的角色，并把组成员信息以及订阅信息发给leader——注意leader和coordinator不是一个概念。leader负责消费分配方案的制定。

![image](https://note.youdao.com/yws/api/personal/file/49232C28D208458A9F383679EEBC38C2?method=download&shareKey=e2cdf39ada37e0170bf703d26edfcb61)

2 Sync，这一步leader开始分配消费方案，即哪个consumer负责消费哪些topic的哪些partition。一旦完成分配，leader会将这个方案封装进SyncGroup请求中发给coordinator，非leader也会发SyncGroup请求，只是内容为空。coordinator接收到分配方案之后会把方案塞进SyncGroup的response中发给各个consumer。这样组内的所有成员就都知道自己应该消费哪些分区了。

![image](https://note.youdao.com/yws/api/personal/file/108C8C6D16564BA9B592927E233E5C23?method=download&shareKey=ba2d9d0149047d57e0b6440dba8df06d)

