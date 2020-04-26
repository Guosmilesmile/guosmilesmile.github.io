---
title: Flink cluster.evenly-spread-out-slots
date: 2020-04-22 20:18:45
tags:
categories:
	- Flink
---


## Flink调度

Flink调度[^1][^2]的时候，会尽量将同一个Task的多个并行度，分配到同一个TM中。会使得某一些TM的cpu消耗比较大，资源使用不够均衡。





#### 配置项

##### cluster.evenly-spread-out-slots



默认是false。可以配置成true使得task分散分配在不同的TM中。



Enable the slot spread out allocation strategy. This strategy tries to spread out the slots evenly across all available `TaskExecutors`.





### 源码解析



JobMaster初始化的时候，会初始化一个调度工程SchedulerFactory。默认的实现是DefaultSchedulerFactory。





SchedulerFactory调用selectSlotSelectionStrategy通过配置获取是否配置了cluster.evenly-spread-out-slots。

SlotSelectionStrategy用于task选择slot的位置。有两种策略。



LocationPreferenceSlotSelectionStrategy.createDefault()和LocationPreferenceSlotSelectionStrategy.createEvenlySpreadOut()。



#### LocationPreferenceSlotSelectionStrategy的两种实现



对应两个实体：

默认调度：DefaultLocationPreferenceSlotSelectionStrategy

分散调度：EvenlySpreadOutLocationPreferenceSlotSelectionStrategy

![image](https://note.youdao.com/yws/api/personal/file/BEEF0A62B0C14679AA503B2786A2528A?method=download&shareKey=314745bec1663ee815d8df9a74cf13c0)



```
selectWitLocationPreference
```



DefaultLocationPreferenceSlotSelectionStrategy和EvenlySpreadOutLocationPreferenceSlotSelectionStrategy给出了不一样的计算公式



```java
protected Optional<SlotInfoAndLocality> selectWithoutLocationPreference(@Nonnull Collection<SlotInfoAndResources> availableSlots, @Nonnull ResourceProfile resourceProfile) {
		for (SlotInfoAndResources candidate : availableSlots) {
			if (candidate.getRemainingResources().isMatching(resourceProfile)) {
				return Optional.of(SlotInfoAndLocality.of(candidate.getSlotInfo(), Locality.UNCONSTRAINED));
			}
		}
		return Optional.empty();
	}
```



EvenlySpreadOut的分发策略是将有相同task的slot的尽量分配在不同TM上



```java
protected Optional<SlotInfoAndLocality> selectWithoutLocationPreference(@Nonnull Collection<SlotInfoAndResources> availableSlots, @Nonnull ResourceProfile resourceProfile) {
		return availableSlots.stream()
			.filter(slotInfoAndResources -> slotInfoAndResources.getRemainingResources().isMatching(resourceProfile))
			.min(Comparator.comparing(SlotInfoAndResources::getTaskExecutorUtilization))
			.map(slotInfoAndResources -> SlotInfoAndLocality.of(slotInfoAndResources.getSlotInfo(), Locality.UNCONSTRAINED));
	}
```



taskExecutorUtilization[^3]是taskExecutor的利用率，每个JobVertex都有一个groupId，会有多个task，taskExecutor中有n个slot，没有被release的个数为m，其中m中没有分配同一个groupid的slot数量为k，

利用率是(m-k)/m。在[0,1]之间。



min(Comparator.comparing(SlotInfoAndResources::getTaskExecutorUtilization)) 获取最小的同groupId占用率。然后将该Solt返回



### 补充说明



[^1]: ResourceId说明

##### ResourceId

```java
/**
 * Gets the ID of the resource in which the TaskManager is started. The format of this depends
 * on how the TaskManager is started:
 * <ul>
 *     <li>If the TaskManager is started via YARN, this is the YARN container ID.</li>
 *     <li>If the TaskManager is started via Mesos, this is the Mesos container ID.</li>
 *     <li>If the TaskManager is started in standalone mode, or via a MiniCluster, this is a random ID.</li>
 *     <li>Other deployment modes can set the resource ID in other ways.</li>
 * </ul>
 * 
 * @return The ID of the resource in which the TaskManager is started
 */
public ResourceID getResourceID() {
   return resourceID;
}
```



[^2]: host源码

#### host源码

```java
/**
 * Returns the fully-qualified domain name the TaskManager. If the name could not be
 * determined, the return value will be a textual representation of the TaskManager's IP address.
 * 
 * @return The fully-qualified domain name of the TaskManager.
 */
public String getFQDNHostname() {
   return fqdnHostName;
}
```



[^3]:  calculateTaskExecutorUtilization


#### calculateTaskExecutorUtilization
```java
private double calculateTaskExecutorUtilization(Map<AllocationID, MultiTaskSlot> map, AbstractID groupId) {
   int numberValidSlots = 0;
   int numberFreeSlots = 0;

   for (MultiTaskSlot multiTaskSlot : map.values()) {
      if (isNotReleasing(multiTaskSlot)) {
         numberValidSlots++;

         if (doesNotContain(groupId, multiTaskSlot)) {
            numberFreeSlots++;
         }
      }
   }

   return (double) (numberValidSlots - numberFreeSlots) / numberValidSlots;
}
```