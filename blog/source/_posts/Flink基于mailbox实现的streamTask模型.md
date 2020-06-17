---
title: Flink源码解析-基于mailbox实现的streamTask模型
date: 2020-06-03 20:38:05
tags:
categories:
	- Flink
---

### AbstractUdfStreamOperator



基本上所有的流式操作，都继承了这个类，如果是单流操作就实现OneInputStreamOperator接口，如果是双流操作就实现TwoInputStreamOperator接口。



AbstractUdfStreamOperator 继承了 AbstractStreamOperator。

AbstractUdfStreamOperator 源码简版：



```java
public abstract class AbstractUdfStreamOperator<OUT, F extends Function>  extends AbstractStreamOperator<OUT> implements OutputTypeConfigurable<OUT> {

     /** The user function. */
     protected final F userFunction;

     public AbstractUdfStreamOperator(F userFunction) {
       this.userFunction = requireNonNull(userFunction);
     }

     public F getUserFunction() {
       return userFunction;
     }
    
    @Override
	public void open() throws Exception {
		super.open();
		FunctionUtils.openFunction(userFunction, new Configuration());
	}

	@Override
	public void close() throws Exception {
		super.close();
		functionsClosed = true;
		FunctionUtils.closeFunction(userFunction);
	}
}
```



AbstractUdfStreamOperator 类中有个泛型 F 类型属性 userFunction，用于保存用户定义的 MapFunction、FlatMapFunction。AbstractUdfStreamOperator 的构造器可以将 userFunction 保存起来。



也提供了open和close方法等，可以在调用上层的close方法后调用用户的close方法。



### AbstractStreamOperator

所有流操作的基类。



```java
public abstract class AbstractStreamOperator<OUT>
		implements StreamOperator<OUT>, SetupableStreamOperator<OUT>, Serializable {
    
    // 该算子的chain策略，ALWAYS可以成chain，NEVER不成chain，HEAD可以称为头部被chain，在各自的
    // 实现类中会被覆盖
    protected ChainingStrategy chainingStrategy = ChainingStrategy.HEAD;
    
    // 包含该算子以及成chain的task
    private transient StreamTask<?, ?> container;
    
    // 第一条流的keySelector，如果是非keyed operator，为null
    private transient KeySelector<?, ?> stateKeySelector1;
    
    // 第二条流的keySelector，如果是非keyed operator，为null
    private transient KeySelector<?, ?> stateKeySelector2;
    
    public void setup(StreamTask<?, ?> containingTask, StreamConfig config, Output<StreamRecord<OUT>> output) {
    }
    
    
    public final void initializeState() throws Exception {
    }
   
}
```



一个算子，可能是keyed，也可能是非keyed，包含了两者都该有的属性，主要负责生命周期相关的操作。





## StreamTask



所有stream task的基本类。一个task 运行一个或者多个StreamOperator（如果成chain）。成chain的算子在同一个线程内同步运行。



每一个 `StreamNode` 在添加到 `StreamGraph` 的时候都会有一个关联的 `jobVertexClass` 属性，这个属性就是该 `StreamNode` 对应的 `StreamTask` 类型；对于一个 `OperatorChain` 而言，它所对应的 `StreamTask` 就是其 head operator 对应的 `StreamTask`





生命周期如下



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

  

```java
abstract class StreamTask {
 
    // invoke前调用，主要初始化状态存储后端，初始化算子等。
    private void beforeInvoke() throws Exception {
		disposedOperators = false;
		LOG.debug("Initializing {}.", getName());

		asyncOperationsThreadPool = Executors.newCachedThreadPool(new ExecutorThreadFactory("AsyncOperations", uncaughtExceptionHandler));

		// 创建状态存储后端
		stateBackend = createStateBackend();
		checkpointStorage = stateBackend.createCheckpointStorage(getEnvironment().getJobID());

		// if the clock is not already set, then assign a default TimeServiceProvider
		if (timerService == null) {
			ThreadFactory timerThreadFactory =
				new DispatcherThreadFactory(TRIGGER_THREAD_GROUP, "Time Trigger for " + getName());

			timerService = new SystemProcessingTimeService(
				this::handleTimerException,
				timerThreadFactory);
		}

		// 创建 OperatorChain，会加载每一个 operator，并调用 setup 方法
		operatorChain = new OperatorChain<>(this, recordWriter);
		headOperator = operatorChain.getHeadOperator();

		// 和具体 StreamTask 子类相关的初始化操作
		// task specific initialization
		init();

		// save the work of reloading state, etc, if the task is already canceled
		if (canceled) {
			throw new CancelTaskException();
		}

		// -------- Invoke --------
		LOG.debug("Invoking {}", getName());

		// we need to make sure that any triggers scheduled in open() cannot be
		// executed before all operators are opened
		actionExecutor.runThrowing(() -> {
			// both the following operations are protected by the lock
			// so that we avoid race conditions in the case that initializeState()
			// registers a timer, that fires before the open() is called.

			initializeStateAndOpen();
		});
	}
    
    @Override
	public final void invoke() throws Exception {
		try {
			beforeInvoke();

			// final check to exit early before starting to run
			if (canceled) {
				throw new CancelTaskException();
			}

			// let the task do its work
			isRunning = true;
			runMailboxLoop();

			// if this left the run() method cleanly despite the fact that this was canceled,
			// make sure the "clean shutdown" is not attempted
			if (canceled) {
				throw new CancelTaskException();
			}

			afterInvoke();
		}
		finally {
			cleanUpInvoke();
		}
	}
    
    
    
}
```





在`beforeInvoke`中会做一些初始化工作，包括提取出所有的operator等。
在`runMailboxLoop`中调用task运行
在`afterInvoke`中结束





### StreamTask With Mailbox

![](https://img-blog.csdnimg.cn/20200422151623683.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3l1Y2h1YW5jaGVu,size_16,color_FFFFFF,t_70)

读过Flink源码的都见过checkpointLock，这个主要是用来隔离不同线程对状态的操作，但是这种使用object lock的做法，使得main thread不得不将这个object传递给所有需要操作状态的线程，一来二去，就会发现源代码里出现大量的synchronize(lock)。这样对于开发和调试和源码阅读，都是及其不方便的。目前checkpoint lock主要使用在以下三个地方：



- Event Processing: Operator在初始化时使用lock来隔离TimerService的触发，在process element时隔离异步snapshot线程对状态的干扰，总之，很多地方都使用了这个checkpoint lock。
- Checkpoint: 自然不用说，在performCheckpoint中使用了lock。
- Processing Time Timers: 这个线程触发的callback常常对状态进行操作，所以也需要获取lock。



在重构这里，社区提议使用Mailbox加单线程的方式来替代checkpoint lock。那么Mailbox就成为了StreamTask的信息来源，（大多数情况下）替代了StreamTask#run():



```java

BlockingQueue<Runnable> mailbox = ...

void runMailboxProcessing() {
    //TODO: can become a cancel-event through mailbox eventually
    Runnable letter;
    while (isRunning()) { 
        while ((letter = mailbox.poll()) != null) {
            letter.run();
        }

        defaultAction();
    }
}

void defaultAction() {
    // e.g. event-processing from an input
}

```

那么对于上面三者，异步的checkpoint和processing timer会将checkpoint lock中的逻辑变成一个Runnable，放入到Mailbox中，这样我们就将并发变成了基于Mailbox的单线程模型，整个StreamTask看起来会更加轻量。





这个是任务运行的核心，即这里会产生action交由`MailboxProcessor`执行。
`processInput`方法处理输入，是task的默认action，在输入上处理一个事件（event）。

```java
public void runMailboxLoop() throws Exception {

		final TaskMailbox localMailbox = mailbox;

		Preconditions.checkState(
			localMailbox.isMailboxThread(),
			"Method must be executed by declared mailbox thread!");

		assert localMailbox.getState() == TaskMailbox.State.OPEN : "Mailbox must be opened!";

		final MailboxController defaultActionContext = new MailboxController(this);

    	// 如果有 mail 需要处理，这里会进行相应的处理，处理完才会进行下面的 event processing
		while (processMail(localMailbox)) {
            // 进行 task 的 default action，也就是调用 processInput()
			mailboxDefaultAction.runDefaultAction(defaultActionContext); // lock is acquired inside default action as needed
		}
	}
```





上面的方法中，最关键的有两个地方：

**processMail()**: 它会检测 *MailBox* 中是否有 mail 需要处理，如果有的话，就做相应的处理，**一直将全部的 mail 处理完才会返回**，只要 loop 还在进行，这里就会返回 true，否则会返回 false

**runDefaultAction()**: 这个最终调用的 *StreamTask* 的 *processInput()* 方法，event-processing 的处理就是在这个方法中进行的



### process-mail 处理



它会检测 *MailBox* 中是否有 mail 需要处理，如果有的话，就做相应的处理，**一直将全部的 mail 处理完才会返回**，只要 loop 还在进行，这里就会返回 true，否则会返回 false。



```java
private boolean processMail(TaskMailbox mailbox) throws Exception {

		// Doing this check is an optimization to only have a volatile read in the expected hot path, locks are only
		// acquired after this point.
        // taskmailbox 会将 queue 中的消息移到 batch，然后从 batch queue 中依次 take；新 mail 写入 queue。从 batch take 时避免加锁
		if (!mailbox.createBatch()) {
			// We can also directly return true because all changes to #isMailboxLoopRunning must be connected to
			// mailbox.hasMail() == true.
            // 消息为空时直接返回
			return true;
		}

		// Take mails in a non-blockingly and execute them.
		Optional<Mail> maybeMail;
		while (isMailboxLoopRunning() && (maybeMail = mailbox.tryTakeFromBatch()).isPresent()) {
            // 从 batch 获取 mail 执行，直到 batch 中的 mail 处理完
			maybeMail.get().run();
		}

		// If the default action is currently not available, we can run a blocking mailbox execution until the default
		// action becomes available again.
		while (isDefaultActionUnavailable() && isMailboxLoopRunning()) {
			mailbox.take(MIN_PRIORITY).run();
		}

		return isMailboxLoopRunning();
	}

```





mail的类型:

Checkpoint Trigger 



我们知道 TaskManager 收到 JM 的 triggerCheckpoint 消息后，会调用 SourceStreamTask 的 triggerCheckpointAsync 方法，对于非 *ExternallyInducedSource*（该类用于外部测试触发 checkpoint 使用），会直接调用 Streamtask 的 *triggerCheckpointAsync* 方法，实现如下：



```java
	// StreamTask.java
	public Future<Boolean> triggerCheckpointAsync(
			CheckpointMetaData checkpointMetaData,
			CheckpointOptions checkpointOptions,
			boolean advanceToEndOfEventTime) {

        // checkpoint 触发时，将触发 checkpoint 的动作发送到 mail
		return mailboxProcessor.getMainMailboxExecutor().submit(
				() -> triggerCheckpoint(checkpointMetaData, checkpointOptions, advanceToEndOfEventTime),
				"checkpoint %s with %s",
			checkpointMetaData,
			checkpointOptions);
	}
```



processTimer

```java
public ProcessingTimeService getProcessingTimeService(int operatorIndex) {
		Preconditions.checkState(timerService != null, "The timer service has not been initialized.");
		MailboxExecutor mailboxExecutor = mailboxProcessor.getMailboxExecutor(operatorIndex);
		return new ProcessingTimeServiceImpl(timerService, callback -> deferCallbackToMailbox(mailboxExecutor, callback));
	}
```





### event-processing 处理



*event-processing* 现在是在 *processInput()* 方法中实现的





```java

protected void processInput(MailboxDefaultAction.Controller controller) throws Exception {
    
        // event 处理
		InputStatus status = inputProcessor.processInput();
		if (status == InputStatus.MORE_AVAILABLE && recordWriter.isAvailable()) {
            // 如果输入还有数据，并且 writer 是可用的，这里就直接返回了
			return;
		}
		if (status == InputStatus.END_OF_INPUT) {
            // 输入已经处理完了，会调用这个方法
			controller.allActionsCompleted();
			return;
		}
        // 代码进行到这里说明 input 或 output 没有准备好（比如当前流中没有数据）
		CompletableFuture<?> jointFuture = getInputOutputJointFuture(status);
        // 告诉 MailBox 先暂停 loop
		MailboxDefaultAction.Suspension suspendedDefaultAction = controller.suspendDefaultAction();
		jointFuture.thenRun(suspendedDefaultAction::resume);
	}


```

判断当前状态来决定是否要继续这个action：如果当前有更多输入，且输出（`recordWriter`）就绪，那么直接返回（因为还有更多的输入，因此不结束action）；如果输入已经结束，标记一下action为结束状态，直接返回；否则将当前的action暂停，直到有输入且输出（`recordWriter`）就绪的时候恢复执行（异步等待)

### StreamInputProcessor

`StreamInputProcessor`有三个实现类，分别是：

```java
StreamOneInputProcessor
StreamTwoInputProcessor
StreamMultipleInputProcessor
```

这三个实现类都有一个成员变量：

```
private final OperatorChain<?, ?> operatorChain;
```

配套这个成员变量的还有两组成员变量，配套的意思是如果是`StreamTwoInputProcessor`，那么下面就有两组：

```java
private final StreamTaskInput<IN> input;
private final DataOutput<IN> output;
```

这里的input负责读，读到ouput中，调用ouput的方法，例如`emitRecord`，这个方法的实现类一般是某个`StreamTask`子类的实现类，在这里会开始处理这个输入数据，例如`OneInputStreamTask`的内部类中的一个实现：

```java
@Override
public void emitRecord(StreamRecord<IN> record) throws Exception {
    numRecordsIn.inc();
    operator.setKeyContextElement1(record);
    operator.processElement(record);
}
```



再结合 *MailboxProcessor* 中的 *runMailboxLoop()* 实现一起看，其操作的流程是：

1.首先通过 *processMail()* 方法处理 MailBox 中的 mail：

- 如果没有 mail 要处理，这里直接返回；
- 先将 MailBox 中当前现存的 mail 全部处理完；
- 通过 *isDefaultActionUnavailable()* 做一个状态检查（目的是提供一个接口方便上层控制调用，这里把这个看作一个状态检查方便讲述），如果是 true 的话，会在这里一直处理 mail 事件，不会返回，除非状态改变；

2.然后再调用 *StreamTask* 的 *processInput()* 方法来处理 event:

- 先调用 *StreamInputProcessor* 的 *processInput()* 方法来处理 event；
- 如果上面处理结果返回的状态是 *MORE_AVAILABLE*（表示还有可用的数据等待处理）并且 *recordWriter* 可用（之前的异步操作已经处理完成），就会立马返回；
- 如果上面处理结果返回的状态是 *END_OF_INPUT*，它表示数据处理完成，这里就会告诉 MailBox 数据已经处理完成了；
- 否则的话，这里会等待，直到有可用的数据到来及 *recordWriter* 可用。



### Reference



http://www.liaojiayi.com/streamtask/

https://www.jishuwen.com/d/27Oh

https://blog.csdn.net/yuchuanchen/article/details/105677408

[https://caesarroot.github.io/2020/03/15/flink%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB%E5%92%8C%E8%A7%A3%E6%9E%90%E7%AC%94%E8%AE%B0-%E4%BB%BB%E5%8A%A1%E6%89%A7%E8%A1%8C/](https://caesarroot.github.io/2020/03/15/flink源码阅读和解析笔记-任务执行/)

https://www.cnblogs.com/Leo_wl/p/11413499.html