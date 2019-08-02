---
title: Flink源码解析 资源申请
date: 2019-07-27 11:14:20
tags:
categories:
	- Flink
---






#### 申请资源

申请资源就得从ejv.allocateResourcesForAll 即 ExecutionJobVertex的allocateResourcesForAll 方法说起。这个方法先遍历了每个ExecutionJobVertex中的所有的task，为每一个task申请一个slot。Execution.allocateAndAssignSlotForExecution方法是主要代码。

```java

		//ExecutionJobVertex一个job顶点，包含里头ExecutionVertex的运行和并行度。
		// allocate the slots (obtain all their futures
		for (ExecutionJobVertex ejv : getVerticesTopologically()) {
			// these calls are not blocking, they only return futures
			//如果一个ExecutionJobVertex有n个并行度，返回n个并行度
			Collection<CompletableFuture<Execution>> allocationFutures = ejv.allocateResourcesForAll(
				slotProvider,
				queued,
				LocationPreferenceConstraint.ALL,
				allPreviousAllocationIds,
				timeout);//申请slot

			allAllocationFutures.addAll(allocationFutures);
		}
		
		/**
         * Defines the location preference constraint.
         *
         * <p> Currently, we support that all input locations have to be taken into consideration
         * and only those which are known at scheduling time. Note that if all input locations
         * are considered, then the scheduling operation can potentially take a while until all
         * inputs have locations assigned.
         */
        public enum LocationPreferenceConstraint {
        	ALL, // wait for all inputs to have a location assigned
        	ANY // only consider those inputs who already have a location assigned
        }
```
默认是使用LocationPreferenceConstraint.ALL,，等待这个task的input部署完之后，才分配。



```java
/**
	 * 申请一个slot为这个JobVertex运行.这个方法返回资源和执行尝试次数，减少顶点和执行查询尝试次数的联系
	 * Acquires a slot for all the execution vertices of this ExecutionJobVertex. The method returns
	 * pairs of the slots and execution attempts, to ease correlation between vertices and execution
	 * attempts.
	 *
	 * 如果这个方法抛出异常，会释放之前申请到的所有资源
	 * <p>If this method throws an exception, it makes sure to release all so far requested slots.
	 *
	 * @param resourceProvider The resource provider from whom the slots are requested.
	 * @param queued if the allocation can be queued
	 * @param locationPreferenceConstraint constraint for the location preferences
	 * @param allPreviousExecutionGraphAllocationIds the allocation ids of all previous executions in the execution job graph.
	 * @param allocationTimeout timeout for allocating the individual slots
	 */
	public Collection<CompletableFuture<Execution>> allocateResourcesForAll(
			SlotProvider resourceProvider,
			boolean queued,
			LocationPreferenceConstraint locationPreferenceConstraint,
			@Nonnull Set<AllocationID> allPreviousExecutionGraphAllocationIds,
			Time allocationTimeout) {
		final ExecutionVertex[] vertices = this.taskVertices;

		@SuppressWarnings("unchecked")
		final CompletableFuture<Execution>[] slots = new CompletableFuture[vertices.length];

		// try to acquire a slot future for each execution.
		// we store the execution with the future just to be on the safe side
		for (int i = 0; i < vertices.length; i++) {
			// allocate the next slot (future)
			final Execution exec = vertices[i].getCurrentExecutionAttempt();
			final CompletableFuture<Execution> allocationFuture = exec.allocateAndAssignSlotForExecution(
				resourceProvider,
				queued,
				locationPreferenceConstraint,
				allPreviousExecutionGraphAllocationIds,
				allocationTimeout);
			slots[i] = allocationFuture;
		}

		// all good, we acquired all slots
		return Arrays.asList(slots);
	}
```

整体的分配资源的方法，在如下中体现
```java
/**
	 * 从slot提供者获取一个slot给一个execution
	 * Allocates and assigns a slot obtained from the slot provider to the execution.
	 *
	 * @param slotProvider to obtain a new slot from
	 * @param queued if the allocation can be queued
	 * @param locationPreferenceConstraint constraint for the location preferences
	 * @param allPreviousExecutionGraphAllocationIds set with all previous allocation ids in the job graph.
	 *                                                 Can be empty if the allocation ids are not required for scheduling.
	 * @param allocationTimeout rpcTimeout for allocating a new slot
	 * @return Future which is completed with this execution once the slot has been assigned
	 * 			or with an exception if an error occurred.
	 * @throws IllegalExecutionStateException if this method has been called while not being in the CREATED state
	 */
	public CompletableFuture<Execution> allocateAndAssignSlotForExecution(
			SlotProvider slotProvider,
			boolean queued,
			LocationPreferenceConstraint locationPreferenceConstraint,
			@Nonnull Set<AllocationID> allPreviousExecutionGraphAllocationIds,
			Time allocationTimeout) throws IllegalExecutionStateException {

		checkNotNull(slotProvider);

		assertRunningInJobMasterMainThread();

		//获取jobVertex的shareing 组
		final SlotSharingGroup sharingGroup = vertex.getJobVertex().getSlotSharingGroup();
		final CoLocationConstraint locationConstraint = vertex.getLocationConstraint();

		// sanity check
		if (locationConstraint != null && sharingGroup == null) {
			throw new IllegalStateException(
					"Trying to schedule with co-location constraint but without slot sharing allowed.");
		}

		// this method only works if the execution is in the state 'CREATED'
		if (transitionState(CREATED, SCHEDULED)) {

			final SlotSharingGroupId slotSharingGroupId = sharingGroup != null ? sharingGroup.getSlotSharingGroupId() : null;

			ScheduledUnit toSchedule = locationConstraint == null ?
					new ScheduledUnit(this, slotSharingGroupId) :
					new ScheduledUnit(this, slotSharingGroupId, locationConstraint);

			// try to extract previous allocation ids, if applicable, so that we can reschedule to the same slot
			ExecutionVertex executionVertex = getVertex();
			AllocationID lastAllocation = executionVertex.getLatestPriorAllocation();

			//上次分配过的id
			Collection<AllocationID> previousAllocationIDs =
				lastAllocation != null ? Collections.singletonList(lastAllocation) : Collections.emptyList();

			//计算偏好位置，根据input的位置计算，如果上游input多于8个， 忽略。
			// calculate the preferred locations
			final CompletableFuture<Collection<TaskManagerLocation>> preferredLocationsFuture =
				calculatePreferredLocations(locationPreferenceConstraint);

			final SlotRequestId slotRequestId = new SlotRequestId();

			final CompletableFuture<LogicalSlot> logicalSlotFuture =
				preferredLocationsFuture.thenCompose(
					(Collection<TaskManagerLocation> preferredLocations) ->
						slotProvider.allocateSlot(
							slotRequestId,
							toSchedule,
							new SlotProfile(
								ResourceProfile.UNKNOWN,
								preferredLocations,
								previousAllocationIDs,
								allPreviousExecutionGraphAllocationIds),
							queued,
							allocationTimeout));

			// register call back to cancel slot request in case that the execution gets canceled
			releaseFuture.whenComplete(
				(Object ignored, Throwable throwable) -> {
					if (logicalSlotFuture.cancel(false)) {
						slotProvider.cancelSlotRequest(
							slotRequestId,
							slotSharingGroupId,
							new FlinkException("Execution " + this + " was released."));
					}
				});

			// This forces calls to the slot pool back into the main thread, for normal and exceptional completion
			return logicalSlotFuture.handle(
				(LogicalSlot logicalSlot, Throwable failure) -> {

					if (failure != null) {
						throw new CompletionException(failure);
					}

					if (tryAssignResource(logicalSlot)) {
						return this;
					} else {
						// release the slot
						logicalSlot.releaseSlot(new FlinkException("Could not assign logical slot to execution " + this + '.'));
						throw new CompletionException(
							new FlinkException(
								"Could not assign slot " + logicalSlot + " to execution " + this + " because it has already been assigned "));
					}
				});
		} else {
			// call race, already deployed, or already done
			throw new IllegalExecutionStateException(this, CREATED, state);
		}
	}
```


计算偏好位置。
```java

	final CompletableFuture<Collection<TaskManagerLocation>> preferredLocationsFuture =
				calculatePreferredLocations(locationPreferenceConstraint);
```

主要是getVertex().getPreferredLocations();

```java

/**
	 * Calculates the preferred locations based on the location preference constraint.
	 *
	 * @param locationPreferenceConstraint constraint for the location preference
	 * @return Future containing the collection of preferred locations. This might not be completed if not all inputs
	 * 		have been a resource assigned.
	 */
	@VisibleForTesting
	public CompletableFuture<Collection<TaskManagerLocation>> calculatePreferredLocations(LocationPreferenceConstraint locationPreferenceConstraint) {
		final Collection<CompletableFuture<TaskManagerLocation>> preferredLocationFutures = getVertex().getPreferredLocations();
        ··········
		return preferredLocationsFuture;
	}
```

如果上游的个数超过8个，那么就不考虑偏好位置的因素
```java
    private static final int MAX_DISTINCT_LOCATIONS_TO_CONSIDER = 8;


	if (inputLocations.size() > MAX_DISTINCT_LOCATIONS_TO_CONSIDER) {
							inputLocations.clear();
							break;
						}
```


slotProvider.allocateSlot。slotProvider的实现类SchedulerImpl.allocateSlot


```java

CompletableFuture<LogicalSlot> allocationFuture = scheduledUnit.getSlotSharingGroupId() == null ?
			allocateSingleSlot(slotRequestId, slotProfile, allowQueuedScheduling, allocationTimeout) :
			allocateSharedSlot(slotRequestId, scheduledUnit, slotProfile, allowQueuedScheduling, allocationTimeout);

```

一般都会有SlotSharingGroupId，因此大部分都是调用allocateSharedSlot(slotRequestId, scheduledUnit, slotProfile, allowQueuedScheduling, allocationTimeout);

```java
	private CompletableFuture<LogicalSlot> allocateSharedSlot(
		SlotRequestId slotRequestId,
		ScheduledUnit scheduledUnit,
		SlotProfile slotProfile,
		boolean allowQueuedScheduling,
		Time allocationTimeout) {
··········
		try {
			if (scheduledUnit.getCoLocationConstraint() != null) {
				multiTaskSlotLocality = allocateCoLocatedMultiTaskSlot(
					scheduledUnit.getCoLocationConstraint(),
					multiTaskSlotManager,
					slotProfile,
					allowQueuedScheduling,
					allocationTimeout);
			} else {
                //生成MultiTaskSlot
				multiTaskSlotLocality = allocateMultiTaskSlot(
					scheduledUnit.getJobVertexId(),
					multiTaskSlotManager,
					slotProfile,
					allowQueuedScheduling,
					allocationTimeout);
			}
············

}		
```
在没有colocation的情况，会调用
```java
	multiTaskSlotLocality = allocateMultiTaskSlot(
					scheduledUnit.getJobVertexId(),
					multiTaskSlotManager,
					slotProfile,
					allowQueuedScheduling,
					allocationTimeout);
```




### LogicalSlot、 MultiTaskSlot 、 SingleTaskSlot
* LogicalSlot:逻辑槽表示TaskManager中可以部署单个任务的资源
* MultiTaskSlot  资源的父节点，下面挂了很多SingleTaskSlot。
* SingleTaskSlot   资源的节点的最小单元，一个。

举例：


```
graph LR
A[source]-->B[map]
A-->C[map]
C-->D[sink]
B-->E[sink]
```

其实会转化为


```
graph LR
A[MultiTaskSlot]-->B[SingleTaskSlot.source]
B-->C[SingleTaskSlot]
C-->D[SingleTaskSlot.sink]
A1[MultiTaskSlot]-->B1[SingleTaskSlot.map]
B1-->C1[SingleTaskSlot.sink]
```



计算资源权重

```java

    /**
	 * Calculates the candidate's locality score.
	 */
	private static final BiFunction<Integer, Integer, Integer> LOCALITY_EVALUATION_FUNCTION =
		(localWeigh, hostLocalWeigh) -> localWeigh * 10 + hostLocalWeigh;

    /**
	 * 先给所有偏好slot加分，遍历所有的slot，locate和host的权重为10:1，只要存在locate分数，则分配策略为locate
	 * 然后将这个slot和策略组成SlotInfoAndLocality返回
	 * @param availableSlots
	 * @param locationPreferences
	 * @param resourceProfile
	 * @return
	 */
	@Nonnull
	private Optional<SlotInfoAndLocality> selectWitLocationPreference(
		@Nonnull Collection<? extends SlotInfo> availableSlots,
		@Nonnull Collection<TaskManagerLocation> locationPreferences,
		@Nonnull ResourceProfile resourceProfile) {

		// we build up two indexes, one for resource id and one for host names of the preferred locations.
		final Map<ResourceID, Integer> preferredResourceIDs = new HashMap<>(locationPreferences.size());
		final Map<String, Integer> preferredFQHostNames = new HashMap<>(locationPreferences.size());

		for (TaskManagerLocation locationPreference : locationPreferences) {
			preferredResourceIDs.merge(locationPreference.getResourceID(), 1, Integer::sum);
			preferredFQHostNames.merge(locationPreference.getFQDNHostname(), 1, Integer::sum);
		}

		SlotInfo bestCandidate = null;
		Locality bestCandidateLocality = Locality.UNKNOWN;
		int bestCandidateScore = Integer.MIN_VALUE;

		for (SlotInfo candidate : availableSlots) {

			if (candidate.getResourceProfile().isMatching(resourceProfile)) {

				// this gets candidate is local-weigh
				Integer localWeigh = preferredResourceIDs.getOrDefault(
					candidate.getTaskManagerLocation().getResourceID(), 0);

				// this gets candidate is host-local-weigh
				Integer hostLocalWeigh = preferredFQHostNames.getOrDefault(
					candidate.getTaskManagerLocation().getFQDNHostname(), 0);

				int candidateScore = LOCALITY_EVALUATION_FUNCTION.apply(localWeigh, hostLocalWeigh);
				if (candidateScore > bestCandidateScore) {
					bestCandidateScore = candidateScore;
					bestCandidate = candidate;
					bestCandidateLocality = localWeigh > 0 ?
						Locality.LOCAL : hostLocalWeigh > 0 ?
						Locality.HOST_LOCAL : Locality.NON_LOCAL;
				}
			}
		}

		// at the end of the iteration, we return the candidate with best possible locality or null.
		return bestCandidate != null ?
			Optional.of(SlotInfoAndLocality.of(bestCandidate, bestCandidateLocality)) :
			Optional.empty();
	}
```


回到申请MultiTaskSlot.首先会获取已经分配的root资源槽信息.并且这个资源不含有相同的操作,如果含有来自同一个jobVertex的，会被过滤（两个map的subTask不会在同一个slot中）。然后获取计算出现有资源的最佳槽，如果有，直接分配这个root（MultiTaskSlot），并且将SingTaskSlot注册在这个root下。

如果资源不够，则想resourceManager申请，通过rpc方法(slotPool.requestNewAllocatedSlot).如果已有的TaskManager没有足够的Slot，SlotManager会向ResourceManager申请新的TaskManager（在启动或者某个TaskExecutor挂掉的情况下会出现这种情况，或者yarn上再创建新的TM)。





```java
private SlotSharingManager.MultiTaskSlotLocality allocateMultiTaskSlot(
		AbstractID groupId,
		SlotSharingManager slotSharingManager,
		SlotProfile slotProfile,
		boolean allowQueuedScheduling,
		Time allocationTimeout) throws NoResourceAvailableException {

		//获取已经分配的root资源槽信息.并且这个资源不含有相同的操作,如果是一个jobVertex，会被过滤
		Collection<SlotInfo> resolvedRootSlotsInfo = slotSharingManager.listResolvedRootSlotInfo(groupId);

		//选取最佳slot
		SlotSelectionStrategy.SlotInfoAndLocality bestResolvedRootSlotWithLocality =
			slotSelectionStrategy.selectBestSlotForProfile(resolvedRootSlotsInfo, slotProfile).orElse(null);

		final SlotSharingManager.MultiTaskSlotLocality multiTaskSlotLocality = bestResolvedRootSlotWithLocality != null ?
			new SlotSharingManager.MultiTaskSlotLocality(
				slotSharingManager.getResolvedRootSlot(bestResolvedRootSlotWithLocality.getSlotInfo()),
				bestResolvedRootSlotWithLocality.getLocality()) :
			null;

		if (multiTaskSlotLocality != null && multiTaskSlotLocality.getLocality() == Locality.LOCAL) {
			return multiTaskSlotLocality;
		}

		final SlotRequestId allocatedSlotRequestId = new SlotRequestId();
		final SlotRequestId multiTaskSlotRequestId = new SlotRequestId();
		//从资源池中获取可以获取的资源，如果没有再去向sourceManger申请
		Optional<SlotAndLocality> optionalPoolSlotAndLocality = tryAllocateFromAvailable(allocatedSlotRequestId, slotProfile);

		if (optionalPoolSlotAndLocality.isPresent()) {
			SlotAndLocality poolSlotAndLocality = optionalPoolSlotAndLocality.get();
			if (poolSlotAndLocality.getLocality() == Locality.LOCAL || bestResolvedRootSlotWithLocality == null) {

				final PhysicalSlot allocatedSlot = poolSlotAndLocality.getSlot();
				final SlotSharingManager.MultiTaskSlot multiTaskSlot = slotSharingManager.createRootSlot(
					multiTaskSlotRequestId,
					CompletableFuture.completedFuture(poolSlotAndLocality.getSlot()),
					allocatedSlotRequestId);

				if (allocatedSlot.tryAssignPayload(multiTaskSlot)) {
					return SlotSharingManager.MultiTaskSlotLocality.of(multiTaskSlot, poolSlotAndLocality.getLocality());
				} else {
					multiTaskSlot.release(new FlinkException("Could not assign payload to allocated slot " +
						allocatedSlot.getAllocationId() + '.'));
				}
			}
		}

		if (multiTaskSlotLocality != null) {
			// prefer slot sharing group slots over unused slots
			if (optionalPoolSlotAndLocality.isPresent()) {
				slotPool.releaseSlot(
					allocatedSlotRequestId,
					new FlinkException("Locality constraint is not better fulfilled by allocated slot."));
			}
			return multiTaskSlotLocality;
		}

		if (allowQueuedScheduling) {
			//获取一个未包含grouopid的未解析的 slot
			// there is no slot immediately available --> check first for uncompleted slots at the slot sharing group
			SlotSharingManager.MultiTaskSlot multiTaskSlot = slotSharingManager.getUnresolvedRootSlot(groupId);

			if (multiTaskSlot == null) {
				// 没有可以获取的slot了，需要从resource manager申请新的slot。
				// it seems as if we have to request a new slot from the resource manager, this is always the last resort!!!
				final CompletableFuture<PhysicalSlot> slotAllocationFuture = slotPool.requestNewAllocatedSlot(
					allocatedSlotRequestId,
					slotProfile.getResourceProfile(),
					allocationTimeout);

				//创建一个root slot
				multiTaskSlot = slotSharingManager.createRootSlot(
					multiTaskSlotRequestId,
					slotAllocationFuture,
					allocatedSlotRequestId);

				slotAllocationFuture.whenComplete(
					(PhysicalSlot allocatedSlot, Throwable throwable) -> {

						final SlotSharingManager.TaskSlot taskSlot = slotSharingManager.getTaskSlot(multiTaskSlotRequestId);

						if (taskSlot != null) {
							// still valid
							if (!(taskSlot instanceof SlotSharingManager.MultiTaskSlot) || throwable != null) {
								taskSlot.release(throwable);
							} else {
								//slot申请下来后进行任务分配
								if (!allocatedSlot.tryAssignPayload(((SlotSharingManager.MultiTaskSlot) taskSlot))) {
									taskSlot.release(new FlinkException("Could not assign payload to allocated slot " +
										allocatedSlot.getAllocationId() + '.'));
								}
							}
						} else {
							slotPool.releaseSlot(
								allocatedSlotRequestId,
								new FlinkException("Could not find task slot with " + multiTaskSlotRequestId + '.'));
						}
					});
			}

			return SlotSharingManager.MultiTaskSlotLocality.of(multiTaskSlot, Locality.UNKNOWN);
		}

		throw new NoResourceAvailableException("Could not allocate a shared slot for " + groupId + '.');
	}
```

slotPoolImpl.requestNewAllocatedSlot-->requestNewAllocatedSlotInternal

```java
private CompletableFuture<AllocatedSlot> requestNewAllocatedSlotInternal(
		@Nonnull SlotRequestId slotRequestId,
		@Nonnull ResourceProfile resourceProfile,
		@Nonnull Time timeout) {

		componentMainThreadExecutor.assertRunningInMainThread();

		final PendingRequest pendingRequest = new PendingRequest(
			slotRequestId,
			resourceProfile);

		// register request timeout
		FutureUtils
			.orTimeout(
				pendingRequest.getAllocatedSlotFuture(),
				timeout.toMilliseconds(),
				TimeUnit.MILLISECONDS,
				componentMainThreadExecutor)
			.whenComplete(
				(AllocatedSlot ignored, Throwable throwable) -> {
					if (throwable instanceof TimeoutException) {
						timeoutPendingSlotRequest(slotRequestId);
					}
				});

		if (resourceManagerGateway == null) {
			stashRequestWaitingForResourceManager(pendingRequest);//本地调用
		} else {
			requestSlotFromResourceManager(resourceManagerGateway, pendingRequest);//从resourceManager申请slot
		}

		return pendingRequest.getAllocatedSlotFuture();
	}
```

requestSlotFromResourceManager中调用resourceManagerGateway.requestSlot发起rpc请求申请资源

```java
private void requestSlotFromResourceManager(
			final ResourceManagerGateway resourceManagerGateway,
			final PendingRequest pendingRequest) {

		checkNotNull(resourceManagerGateway);
		checkNotNull(pendingRequest);

		log.info("Requesting new slot [{}] and profile {} from resource manager.", pendingRequest.getSlotRequestId(), pendingRequest.getResourceProfile());

		final AllocationID allocationId = new AllocationID();

		pendingRequests.put(pendingRequest.getSlotRequestId(), allocationId, pendingRequest);

		pendingRequest.getAllocatedSlotFuture().whenComplete(
			(AllocatedSlot allocatedSlot, Throwable throwable) -> {
				if (throwable != null || !allocationId.equals(allocatedSlot.getAllocationId())) {
					// cancel the slot request if there is a failure or if the pending request has
					// been completed with another allocated slot
					resourceManagerGateway.cancelSlotRequest(allocationId);
				}
			});

		//申请资源  rpc向resourceManager
		CompletableFuture<Acknowledge> rmResponse = resourceManagerGateway.requestSlot(
			jobMasterId,
			new SlotRequest(jobId, allocationId, pendingRequest.getResourceProfile(), jobManagerAddress),
			rpcTimeout);

		FutureUtils.whenCompleteAsyncIfNotDone(
			rmResponse,
			componentMainThreadExecutor,
			(Acknowledge ignored, Throwable failure) -> {
				// on failure, fail the request future
				if (failure != null) {
					slotRequestToResourceManagerFailed(pendingRequest.getSlotRequestId(), failure);
				}
			});
	}
```

rpc的最后会调用到YarnResourceManager.startNewWorker(如果是yarn)启动一个新的容器

```java
@Override
	public Collection<ResourceProfile> startNewWorker(ResourceProfile resourceProfile) {
		Preconditions.checkArgument(
			ResourceProfile.UNKNOWN.equals(resourceProfile),
			"The YarnResourceManager does not support custom ResourceProfiles yet. It assumes that all containers have the same resources.");
		requestYarnContainer();

		return slotsPerWorker;
	}
```

### Reference

https://blog.csdn.net/qq475781638/article/details/90923673

https://www.cnblogs.com/andyhe/p/10633692.html

https://zhuanlan.zhihu.com/p/36525639


https://blog.csdn.net/qq_36421826/article/details/80769016