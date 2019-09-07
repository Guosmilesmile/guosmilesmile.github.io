---
title: Flink源码解析 TaskManager启动与运行Task
date: 2019-08-28 21:36:39
tags:
categories:
	- Flink
---

```shell
./taskmanager.sh start
```
实际上，调用了如下语句
```
/usr/local/flink/flink-1.7.2/bin/flink-daemon.sh start taskexecutor --configDir /usr/local/flink/flink-1.7.2/conf
```

在flink-daemon.sh脚本中
```
org.apache.flink.runtime.taskexecutor.TaskManagerRunner
```
调用的类是这个。

```
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.65-3.b17.el7.x86_64/bin/java -Dlog.file=/cache1/flink/log/flink-root-taskexecutor-4-PShnczsjzxvp26.log -Dlog4j.configuration=file:/usr/local/flink/flink-1.7.2/conf/log4j.properties -Dlogback.configurationFile=file:/usr/local/flink/flink-1.7.2/conf/logback.xml -classpath /usr/local/flink/flink-1.7.2/lib/flink-python_2.11-1.7.2.jar:/usr/local/flink/flink-1.7.2/lib/flink-queryable-state-runtime_2.11-1.7.2.jar:/usr/local/flink/flink-1.7.2/lib/flink-shaded-hadoop2-uber-1.7.2.jar:/usr/local/flink/flink-1.7.2/lib/log4j-1.2.17.jar:/usr/local/flink/flink-1.7.2/lib/slf4j-log4j12-1.7.15.jar:/usr/local/flink/flink-1.7.2/lib/flink-dist_2.11-1.7.2.jar::/etc/hadoop/: org.apache.flink.runtime.taskexecutor.TaskManagerRunner --configDir /usr/local/flink/flink-1.7.2/conf

```
在main方法中，通过入参的configDir路径，获取对应的配置文件，然后调用runTaskManager方法，在这个方法中主要是生成TaskManagerRunner，调用start方法，start方法主要是调用TaskExecutor.start。所以主要是看TaskExecutor这个类。
```
public static void runTaskManager(Configuration configuration, ResourceID resourceId) throws Exception {
		final TaskManagerRunner taskManagerRunner = new TaskManagerRunner(configuration, resourceId);

		taskManagerRunner.start();
	}
```
先看下下面的类关系。

![image](
https://note.youdao.com/yws/api/personal/file/D3D3E282328F4363954EB769A0193915?method=download&shareKey=1d336f5a86282de1ec55086c9f13eae6)

* TaskManagerServices ： TaskExecutor服务（内存管理，io管理等等）的持有类
* TaskManagerLocation : 这个类封装了TaskManager的连接信息
* MemoryManager : 内存管理
* IOManager ：提供IO服务
* NetworkEnvironment ： 网络服务包含跟踪所有中间结果和所有数据交换的数据结构
* BroadcastVariableManager : 广播变量服务
* TaskSlotTable :TaskSlot的持有者
* TaskSlot ： Task的持有者，一个TaskSlot可以拥有多个Task
* Task：表示在TaskManager上执行并行子任务
* JobManagerTable：将JobManagerConnection与JobId关联
* JobLeaderService: 监控job对应的leader
* TaskExecutorLocalStateStoresManager：持有所有的TaskLocalStateStoreImpl实例

TaskMangerRunner.java 实际调用的是rpcServer.start();
```java
    //实际调用rpcServer.start();
	public void start() throws Exception {
		taskManager.start();
	}
```

当提交作业调用RPC服务的时候，通过rpc服务到taskManger调用TaskExecutor.submitTask，获取提交的Task。


### Task


![image](https://note.youdao.com/yws/api/personal/file/4F68C2B40D6A4A4DA0620B67DD710F37?method=download&shareKey=c9a775082d067758b5e2c868dc4fb9ba)



* TaskInfo：task信息，包括名字，子任务的系列号，并行度和重试次数
* ResultPartition：单个任务生成的数据的结果分区,是buffer实例们的集合。
* SingleInputGate：消费一个或者多个分区的数据

其他类就先忽略。

![image](https://note.youdao.com/yws/api/personal/file/56744FB7B89448F2A58698D5D6151587?method=download&shareKey=c4dddaf0898abdac28d4123cd0a4d33c)



### Task运行


状态初始化
* 死循环等待状态从CREATED修改为DEPLOYING成功，修改成功后退出
* 如果task当前状态不是CREATED则退出run方法

##### 启动和运行
* 创建一个jobId粒度的class loader并下载缺失的jar files；基于不同的class loader的类加载隔离机制可以在JVM进程内隔离不同的task运行环境
* 将当前task实例注册到network stack，如果可用内存不足，注册可能会失败（networ.registerTask  主要是注册inputGate和resultPartition 需要细看）
* 后台拷贝分布式缓存文件
* 加载和初始化任务的invokable代码（用户代码）

```java
invokable  = loadAndInstantiateInvokable()
```

invokable会实例化返回一个StreamTask，这个类只是JobVertex的head节点，例如StreamSourceTask或者每条链路的head，在StreamTask这个类的invoke()方法中，new OperatorChain的时候，会调用createChainedOperator创建出这个JobVertex中的其他StreamNode的函数，从而完成这个Task中整个链路的所有user function.



逐一调用每个结果分区的finish方法，subtask状态从RUNNING切换到FINISHED



### 流式Task

在流式的Task中，在调用在StreamTask这个类的invoke()方法中，会调用到run(),是一个while(running&&inputProcessor.processInput)实现数据不同的往下发。接下来我们分析一下StreamTask.invoke()


```java
public final void invoke() throws Exception {

    //--------------初始化各种-----------------
    
    
    //获取这个task的整个操作链和headOperator,初始化output
    operatorChain = new OperatorChain<>(this, recordWriters);
    headOperator = operatorChain.getHeadOperator();
    
    //初始化
    init();
    
    
    //-------invoke-------------
    
    
    // executed before all operators are opened
		synchronized (lock) {

		// both the following operations are protected by the lock
		// so that we avoid race conditions in the case that initializeState()
		// registers a timer, that fires before the open() is called.
        initializeState();//初始化状态
		openAllOperators();//执行这个Task下的所有操作符的open方法
		
		// let the task do its work
			isRunning = true;
			run();

	}
    

}



```




init()如果是SourceStreamTask，没有初始化inputGate（他没有上游），如果是OneInputStreamTask，会初始化inputProcessor




run()

如果是SourceStreamTask，run会调用streamSource.run,获取SourceContext然后执行用户函数，传递数据

```java

public void run(final Object lockingObject,
			final StreamStatusMaintainer streamStatusMaintainer,
			final Output<StreamRecord<OUT>> collector) throws Exception {
			
			//----------------
			
        this.ctx = StreamSourceContexts.getSourceContext(
			timeCharacteristic,
			getProcessingTimeService(),
			lockingObject,
			streamStatusMaintainer,
			collector,
			watermarkInterval,
			-1);


		try {
			userFunction.run(ctx);
			
			
			//----------------
		

```


如果是OneInputStreamTask，run如下

```java
@Override
	protected void run() throws Exception {
		// cache processor reference on the stack, to make the code more JIT friendly
		final StreamInputProcessor<IN> inputProcessor = this.inputProcessor;

		while (running && inputProcessor.processInput()) {
			// all the work happens in the "processInput" method
		}
	}
```






### Reference
https://blog.csdn.net/a860MHz/article/details/91877325