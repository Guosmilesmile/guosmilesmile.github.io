---
title: Flink源码解析 CLI提交job到JobManger到TaskMnager运行job
date: 2019-07-20 16:56:49
tags:
categories:
	- Flink
---

## CLI提交Job

#### 启动Job
```
./bin/flink run examples/streaming/SocketWindowWordCount.jar
```
跟踪Flink的脚本代码就会发现，最终会执行以下命令：
```
exec $JAVA_RUN $JVM_ARGS "${log_setting[@]}" -classpath "`manglePathList "$CC_CLASSPATH:$INTERNAL_HADOOP_CLASSPATHS"`" org.apache.flink.client.CliFrontend "$@"
```

实际上调用了CliFrontend这个类，这个类的main方法，主要是处理接到的参数，根据参数决定要执行上面函数。
```java
/**
	 * Submits the job based on the arguments.
	 */
	public static void main(final String[] args) {
		EnvironmentInformation.logEnvironmentInfo(LOG, "Command Line Client", args);

		// 1. find the configuration directory
		final String configurationDirectory = getConfigurationDirectoryFromEnv();

		// 2. load the global configuration
		final Configuration configuration = GlobalConfiguration.loadConfiguration(configurationDirectory);

		// 3. load the custom command lines
		final List<CustomCommandLine<?>> customCommandLines = loadCustomCommandLines(
			configuration,
			configurationDirectory);

		try {
			final CliFrontend cli = new CliFrontend(
				configuration,
				customCommandLines);

			SecurityUtils.install(new SecurityConfiguration(cli.configuration));
			int retCode = SecurityUtils.getInstalledContext()
					.runSecured(() -> cli.parseParameters(args));
			System.exit(retCode);
		}
		catch (Throwable t) {
			final Throwable strippedThrowable = ExceptionUtils.stripException(t, UndeclaredThrowableException.class);
			LOG.error("Fatal error while running command line interface.", strippedThrowable);
			strippedThrowable.printStackTrace();
			System.exit(31);
		}
	}
```

其中核心的部分在于cli.parseParameters(args)
```java
/**
     * Parses the command line arguments and starts the requested action.
     *
     * @param args command line arguments of the client.
     * @return The return code of the program
     */
    public int parseParameters(String[] args) {

        // check for action
        if (args.length < 1) {
            CliFrontendParser.printHelp();
            System.out.println("Please specify an action.");
            return 1;
        }

        // get action
        String action = args[0];

        // remove action from parameters
        final String[] params = Arrays.copyOfRange(args, 1, args.length);

        // do action
        switch (action) {
            case ACTION_RUN:
                return run(params);
            case ACTION_LIST:
                return list(params);
            case ACTION_INFO:
                return info(params);
            case ACTION_CANCEL:
                return cancel(params);
            case ACTION_STOP:
                return stop(params);
            case ACTION_SAVEPOINT:
                return savepoint(params);
            case "-h":
            case "--help":
                CliFrontendParser.printHelp();
                return 0;
            case "-v":
            case "--version":
                String version = EnvironmentInformation.getVersion();
                String commitID = EnvironmentInformation.getRevisionInformation().commitId;
                System.out.print("Version: " + version);
                System.out.println(!commitID.equals(EnvironmentInformation.UNKNOWN) ? ", Commit ID: " + commitID : "");
                return 0;
            default:
                System.out.printf("\"%s\" is not a valid action.\n", action);
                System.out.println();
                System.out.println("Valid actions are \"run\", \"list\", \"info\", \"savepoint\", \"stop\", or \"cancel\".");
                System.out.println();
                System.out.println("Specify the version option (-v or --version) to print Flink version.");
                System.out.println();
                System.out.println("Specify the help option (-h or --help) to get help on the command.");
                return 1;
        }
    }
```

根据第一个参数作为action，调用不同的方法。jar包的运行的入参是run，往下看run方法。

```java
protected void run(String[] args) throws Exception {
		LOG.info("Running 'run' command.");

		final Options commandOptions = CliFrontendParser.getRunCommandOptions();

		final Options commandLineOptions = CliFrontendParser.mergeOptions(commandOptions, customCommandLineOptions);

		final CommandLine commandLine = CliFrontendParser.parse(commandLineOptions, args, true);

		final RunOptions runOptions = new RunOptions(commandLine);

		// evaluate help flag
		if (runOptions.isPrintHelp()) {
			CliFrontendParser.printHelpForRun(customCommandLines);
			return;
		}

		if (runOptions.getJarFilePath() == null) {
			throw new CliArgsException("The program JAR file was not specified.");
		}

		//创建一个实例：PackagedProgram, 封装入口类、jar文件、classpath路径、用户配置参数
		final PackagedProgram program;
		try {
			LOG.info("Building program from JAR file");
			program = buildProgram(runOptions);
		}
		catch (FileNotFoundException e) {
			throw new CliArgsException("Could not build the program from JAR file.", e);
		}

		final CustomCommandLine<?> customCommandLine = getActiveCustomCommandLine(commandLine);

		try {
			runProgram(customCommandLine, commandLine, runOptions, program);
		} finally {
			program.deleteExtractedLibraries();
		}
	}
```

核心代码主要是runProgram(customCommandLine, commandLine, runOptions, program);
而runProgram中调用了executeProgram(program, client, userParallelism);这个函数，这个函数主要是与JobManager进行交互。



```java
protected void executeProgram(PackagedProgram program, ClusterClient<?> client, int parallelism) throws ProgramMissingJobException, ProgramInvocationException {
		logAndSysout("Starting execution of program");

		final JobSubmissionResult result = client.run(program, parallelism);

		if (null == result) {
			throw new ProgramMissingJobException("No JobSubmissionResult returned, please make sure you called " +
				"ExecutionEnvironment.execute()");
		}

		if (result.isJobExecutionResult()) {
			logAndSysout("Program execution finished");
			JobExecutionResult execResult = result.getJobExecutionResult();
			System.out.println("Job with JobID " + execResult.getJobID() + " has finished.");
			System.out.println("Job Runtime: " + execResult.getNetRuntime() + " ms");
			Map<String, Object> accumulatorsResult = execResult.getAllAccumulatorResults();
			if (accumulatorsResult.size() > 0) {
				System.out.println("Accumulator Results: ");
				System.out.println(AccumulatorHelper.getResultsFormatted(accumulatorsResult));
			}
		} else {
			logAndSysout("Job has been submitted with JobID " + result.getJobID());
		}
	}
```
与jobManager的交互结果全部都封装在了JobSubmissionResult实体中，
client.run的主要代码如下

```java

	 * 在client连接的Flink集群运行一个程序，这个调用会等待运行完成并返回结果。
	 * Runs a program on the Flink cluster to which this client is connected. The call blocks until the
	 * execution is complete, and returns afterwards.
	 *
	 * @param jobWithJars The program to be executed.
	 * @param parallelism The default parallelism to use when running the program. The default parallelism is used
	 *                    when the program does not set a parallelism by itself.
	 *
	 * @throws CompilerException Thrown, if the compiler encounters an illegal situation.
	 * @throws ProgramInvocationException Thrown, if the program could not be instantiated from its jar file,
	 *                                    or if the submission failed. That might be either due to an I/O problem,
	 *                                    i.e. the job-manager is unreachable, or due to the fact that the
	 *                                    parallel execution failed.
	 */
	public JobSubmissionResult run(JobWithJars jobWithJars, int parallelism, SavepointRestoreSettings savepointSettings)
			throws CompilerException, ProgramInvocationException {
		ClassLoader classLoader = jobWithJars.getUserCodeClassLoader();
		if (classLoader == null) {
			throw new IllegalArgumentException("The given JobWithJars does not provide a usercode class loader.");
		}
		//  获取优化计划JobGraph
		OptimizedPlan optPlan = getOptimizedPlan(compiler, jobWithJars, parallelism);
		return run(optPlan, jobWithJars.getJarFiles(), jobWithJars.getClasspaths(), classLoader, savepointSettings);
	}
```

run(optPlan, jobWithJars.getJarFiles(), jobWithJars.getClasspaths(), classLoader, savepointSettings)中会调用submitJob方法，这个方法实现有两个MiniCluserClient.submitJob和RestClusterClient.submitJob。

MiniCluserClient.submitJob通过miniCluster.submitJob(jobGraph)提交作业，以本地模式运行。

```java

@Override
	public CompletableFuture<JobSubmissionResult> submitJob(@Nonnull JobGraph jobGraph) {
		return miniCluster.submitJob(jobGraph);
	}
	
```

RestClusterClient.submitJob发送RPC请求到JobManager

到这里，已经完成了通过CLI提交作业的过程，CLI通过RCP的方式与JobManger交互，实现job的提交。

### TaskManger接收job submit请求

* 本地环境下，MiniCluster完成了大部分任务，直接把任务委派给了MiniDispatcher；
* 远程环境下，启动了一个RestClusterClient，这个类会以HTTP Rest的方式把用户代码提交到集群上；
* 远程环境下，请求发到集群上之后，必然有个handler去处理，在这里是JobSubmitHandler。这个类接手了请求后，委派StandaloneDispatcher启动job，到这里之后，本地提交和远程提交的逻辑往后又统一了；
* Dispatcher接手job之后，会实例化一个JobManagerRunner，然后用这个runner启动job；
* JobManagerRunner接下来把job交给了JobMaster去处理；
* JobMaster使用ExecutionGraph的方法启动了整个执行图；整个任务就启动起来了。

#### attention

JobManagerRunner负责作业的运行，和对JobMaster的管理。 不是集群级别的JobManager。这个是历史原因。JobManager是老的runtime框架，JobMaster是社区 flip-6引入的新的runtime框架。目前起作用的应该是JobMaster。因此这个类应该叫做和对JobMasterRunner比较合适。



首先来看下Flink的四种图 

![image](https://note.youdao.com/yws/api/personal/file/56744FB7B89448F2A58698D5D6151587?method=download&shareKey=c4dddaf0898abdac28d4123cd0a4d33c)


flink总共提供了三种图的抽象，StreamGraph和JobGraph，还有一种是ExecutionGraph，是用于调度的基本数据结构。 

在JobGraph转换到ExecutionGraph的过程中，主要发生了以下转变： 
* 加入了并行度的概念，成为真正可调度的图结构
* 生成了与JobVertex对应的ExecutionJobVertex，ExecutionVertex，与IntermediateDataSet对应的IntermediateResult和IntermediateResultPartition等，并行将通过这些类实现

ExecutionGraph已经可以用于调度任务。我们可以看到，flink根据该图生成了一一对应的Task，每个task对应一个ExecutionGraph的一个Execution。Task用InputGate、InputChannel和ResultPartition对应了上面图中的IntermediateResult和ExecutionEdge。



Dispatcher.submitJob是提交job的主要处理方法
```java
@Override
	public CompletableFuture<Acknowledge> submitJob(JobGraph jobGraph, Time timeout) {
		log.info("Received JobGraph submission {} ({}).", jobGraph.getJobID(), jobGraph.getName());

		try {
			if (isDuplicateJob(jobGraph.getJobID())) {
				return FutureUtils.completedExceptionally(
					new JobSubmissionException(jobGraph.getJobID(), "Job has already been submitted."));
			} else {
				return internalSubmitJob(jobGraph);
			}
		} catch (FlinkException e) {
			return FutureUtils.completedExceptionally(e);
		}
	}
```

主要运行的是Dispatcher中的submitJob--->internalSubmitJob--->persistAndRunJob  --> runJob-->createJobManagerRunner-->startJobManagerRunner

下面是runJob方法
```java
private CompletableFuture<Void> runJob(JobGraph jobGraph) {
		//判断这个job是否已经运行，避免重复提交
		Preconditions.checkState(!jobManagerRunnerFutures.containsKey(jobGraph.getJobID()));

		final CompletableFuture<JobManagerRunner> jobManagerRunnerFuture = createJobManagerRunner(jobGraph);

		jobManagerRunnerFutures.put(jobGraph.getJobID(), jobManagerRunnerFuture);

		return jobManagerRunnerFuture
			.thenApply(FunctionUtils.nullFn())
			.whenCompleteAsync(
				(ignored, throwable) -> {
					if (throwable != null) {
						jobManagerRunnerFutures.remove(jobGraph.getJobID());
					}
				},
				getMainThreadExecutor());
	}
```
在createJobManagerRunner中会调用
DefaultJobManagerRunnerFactory.createJobManagerRunner创建一个JobManagerRunner，调用Dispatcher.startJobManagerRunner;-->jobManagerRunner.start();


jobManagerRunner.start()中调用了StandaloneHaServices.start

```java
public void start() throws Exception {
		try {
			leaderElectionService.start(this);//======>StandaloneHaServices
		} catch (Exception e) {
			log.error("Could not start the JobManager because the leader election service did not start.", e);
			throw new Exception("Could not start the leader election service.", e);
		}
	}
```

StandaloneLeaderElectionService实际调用了JobManagerRunner.grantLeadership

```java

public void start(LeaderContender newContender) throws Exception {
		if (contender != null) {
			// Service was already started
			throw new IllegalArgumentException("Leader election service cannot be started multiple times.");
		}

		contender = Preconditions.checkNotNull(newContender);

		// directly grant leadership to the given contender
		contender.grantLeadership(HighAvailabilityServices.DEFAULT_LEADER_ID);//==> 调用JobManagerRunner.grantLeadership
	}

```

JobManagerRunner.grantLeadership中的核心部分为verifyJobSchedulingStatusAndStartJobManager

```java
private CompletableFuture<Void> verifyJobSchedulingStatusAndStartJobManager(UUID leaderSessionId) {
		final CompletableFuture<JobSchedulingStatus> jobSchedulingStatusFuture = getJobSchedulingStatus();

		return jobSchedulingStatusFuture.thenCompose(
			jobSchedulingStatus -> {
				if (jobSchedulingStatus == JobSchedulingStatus.DONE) {//任务已完成
					return jobAlreadyDone();
				} else {
					return startJobMaster(leaderSessionId);//启动任务
				}
			});
	}
```


```java
private CompletionStage<Void> startJobMaster(UUID leaderSessionId) {
		log.info("JobManager runner for job {} ({}) was granted leadership with session id {} at {}.",
			jobGraph.getName(), jobGraph.getJobID(), leaderSessionId, getAddress());

		try {
			runningJobsRegistry.setJobRunning(jobGraph.getJobID());
		} catch (IOException e) {
			return FutureUtils.completedExceptionally(
				new FlinkException(
					String.format("Failed to set the job %s to running in the running jobs registry.", jobGraph.getJobID()),
					e));
		}

		final CompletableFuture<Acknowledge> startFuture;
		try {
			//实际调用JobMaster.start
			startFuture = jobMasterService.start(new JobMasterId(leaderSessionId));
		} catch (Exception e) {
			return FutureUtils.completedExceptionally(new FlinkException("Failed to start the JobMaster.", e));
		}

		final CompletableFuture<JobMasterGateway> currentLeaderGatewayFuture = leaderGatewayFuture;
		return startFuture.thenAcceptAsync(
			(Acknowledge ack) -> confirmLeaderSessionIdIfStillLeader(leaderSessionId, currentLeaderGatewayFuture),
			executor);
	}
```
接着JobMaster的启动，继续往下看

JobMaster.start ===> startJobExecution
```java
public CompletableFuture<Acknowledge> start(final JobMasterId newJobMasterId) throws Exception {
		// make sure we receive RPC and async calls
		start();

		return callAsyncWithoutFencing(() -> startJobExecution(newJobMasterId), RpcUtils.INF_TIMEOUT);
	}
```

```java
//-- job starting and stopping  -----------------------------------------------------------------

	private Acknowledge startJobExecution(JobMasterId newJobMasterId) throws Exception {

		validateRunsInMainThread();

		checkNotNull(newJobMasterId, "The new JobMasterId must not be null.");

		if (Objects.equals(getFencingToken(), newJobMasterId)) {
			log.info("Already started the job execution with JobMasterId {}.", newJobMasterId);

			return Acknowledge.get();
		}

		setNewFencingToken(newJobMasterId);

		//包含了slotPoll启动 resourceManager的连接(后续用于request slot)
		startJobMasterServices();

		log.info("Starting execution of job {} ({}) under job master id {}.", jobGraph.getName(), jobGraph.getJobID(), newJobMasterId);

		//执行job
		resetAndScheduleExecutionGraph();

		return Acknowledge.get();
	}
```

这里将JobMastert中的slotpool启动，并和JM的ResourceManager通信
```java
private void startJobMasterServices() throws Exception {
		// start the slot pool make sure the slot pool now accepts messages for this leader
		slotPool.start(getFencingToken(), getAddress(), getMainThreadExecutor());//slotPool是一个Rpc服务
		scheduler.start(getMainThreadExecutor());

		//TODO: Remove once the ZooKeeperLeaderRetrieval returns the stored address upon start
		// try to reconnect to previously known leader
		reconnectToResourceManager(new FlinkException("Starting JobMaster component."));//连接resourceManager

		// job is ready to go, try to establish connection with resource manager
		//   - activate leader retrieval for the resource manager
		//   - on notification of the leader, the connection will be established and
		//     the slot pool will start requesting slots
		resourceManagerLeaderRetriever.start(new ResourceManagerLeaderListener());//告知resourceManager启动正常
	}
```

在slotPool和resourcemanager通信完毕后 开始执行job ，resetAndScheduleExecutionGraph();//执行job

这里会将JobGraph转为ExecutionGraph并执行
===>
scheduleExecutionGraph()
===>ExecutionGraph.scheduleForExecution();

```java

	private void resetAndScheduleExecutionGraph() throws Exception {
		validateRunsInMainThread();

		final CompletableFuture<Void> executionGraphAssignedFuture;

		if (executionGraph.getState() == JobStatus.CREATED) {
			executionGraphAssignedFuture = CompletableFuture.completedFuture(null);
			executionGraph.start(getMainThreadExecutor());
		} else {
			suspendAndClearExecutionGraphFields(new FlinkException("ExecutionGraph is being reset in order to be rescheduled."));
			final JobManagerJobMetricGroup newJobManagerJobMetricGroup = jobMetricGroupFactory.create(jobGraph);
			final ExecutionGraph newExecutionGraph = createAndRestoreExecutionGraph(newJobManagerJobMetricGroup);

			executionGraphAssignedFuture = executionGraph.getTerminationFuture().handle(
				(JobStatus ignored, Throwable throwable) -> {
					newExecutionGraph.start(getMainThreadExecutor());
					assignExecutionGraph(newExecutionGraph, newJobManagerJobMetricGroup);
					return null;
				});
		}

		executionGraphAssignedFuture.thenRun(this::scheduleExecutionGraph);//执行executionGraph
	}
	
	
	private void scheduleExecutionGraph() {
		checkState(jobStatusListener == null);
		// register self as job status change listener
		jobStatusListener = new JobManagerJobStatusListener();
		executionGraph.registerJobStatusListener(jobStatusListener);

		try {
			executionGraph.scheduleForExecution();
		}
		catch (Throwable t) {
			executionGraph.failGlobal(t);
		}
	}
```

==>scheduleEager(slotProvider, allocationTimeout);//立即执行
===>执行任务的核心方法
申请资源。

```java
private CompletableFuture<Void> scheduleEager(SlotProvider slotProvider, final Time timeout) {

..............

for (ExecutionJobVertex ejv : getVerticesTopologically()) {
			// these calls are not blocking, they only return futures
			Collection<CompletableFuture<Execution>> allocationFutures = ejv.allocateResourcesForAll(
				slotProvider,
				queued,
				LocationPreferenceConstraint.ALL,
				allPreviousAllocationIds,
				timeout);//申请slot

			allAllocationFutures.addAll(allocationFutures);
		}

.............


```

execution.deploy();//任务触发执行

```java
public void deploy() throws JobException {

................

		final TaskDeploymentDescriptor deployment = vertex.createDeploymentDescriptor(
				attemptId,
				slot,
				taskRestore,
				attemptNumber);


...............

}

```

```java
taskManagerGateway.submitTask(deployment, rpcTimeout)////==》AkkaInvocationHandler通过rpc调用，然后TM收到服务调用RpcTaskManagerGateway.submitTask
```

到这里JobManger通过AkkaInvocationHandler调用rpc服务将任务提交到分配的TM上。

### TM接收submitTask请求运行Task

AkkaInvocationHandler通过rpc调用，然后TM收到服务调用RpcTaskManagerGateway.submitTask



RpcTaskManagerGateway.submitTask

```java
@Override
	public CompletableFuture<Acknowledge> submitTask(TaskDeploymentDescriptor tdd, Time timeout) {
		return taskExecutorGateway.submitTask(tdd, jobMasterId, timeout);//==>TaskExecutor.submitTask
	}
```

TaskExecutor.submitTask


```java
	Task task = new Task(
				jobInformation,
				taskInformation,
				tdd.getExecutionAttemptId(),
				tdd.getAllocationId(),
				tdd.getSubtaskIndex(),
				tdd.getAttemptNumber(),
				tdd.getProducedPartitions(),
				tdd.getInputGates(),
				tdd.getTargetSlotNumber(),
				taskExecutorServices.getMemoryManager(),
				taskExecutorServices.getIOManager(),
				taskExecutorServices.getNetworkEnvironment(),
				taskExecutorServices.getBroadcastVariableManager(),
				taskStateManager,
				taskManagerActions,
				inputSplitProvider,
				checkpointResponder,
				aggregateManager,
				blobCacheService,
				libraryCache,
				fileCache,
				taskManagerConfiguration,
				taskMetricGroup,
				resultPartitionConsumableNotifier,
				partitionStateChecker,
				getRpcService().getExecutor());
				
				
    	task.startTaskThread();//任务真正执行

```



























#### 申请资源

申请资源就得从ejv.allocateResourcesForAll 即 ExecutionJobVertex的allocateResourcesForAll 方法说起。


未完待续



	
### Reference
https://www.jianshu.com/p/4a5017f20641

https://www.jianshu.com/p/7b4af44eb3f3


https://blog.csdn.net/qq475781638/article/details/90923673

https://www.cnblogs.com/ooffff/p/9486451.html


https://www.cnblogs.com/bethunebtj/p/9168274.html#3-%E4%BB%BB%E5%8A%A1%E7%9A%84%E8%B0%83%E5%BA%A6%E4%B8%8E%E6%89%A7%E8%A1%8C