---
title: Flink StreamingFileSink源码分析
date: 2021-08-15 21:10:56
tags:
categories:
	- Flink
---


## 背景

关于StreamFileSink的使用，直接见官网吧，更全

https://ci.apache.org/projects/flink/flink-docs-release-1.13/zh/docs/connectors/datastream/streamfile_sink/


```java
public static StreamingFileSink<OneLogEntity> init(String outputPath) {
        return StreamingFileSink
                .forRowFormat(new Path(outputPath), new SimpleStringEncoder<OneLogEntity>("UTF-8"))
                .withBucketAssigner(new HostDateTimeBucketAssigner())
                .withRollingPolicy(
                        DefaultRollingPolicy.builder()
                                .withRolloverInterval(TimeUnit.MINUTES.toMillis(5)) // 它至少包含5分钟的数据
                                .withInactivityInterval(TimeUnit.MINUTES.toMillis(1)) // 最近1分钟没有收到新的记录
                                .withMaxPartSize(1024 * 1024 * 1024) // 文件大小达到 1GB（写入最后一条记录后）
                                .build())
                .build();
    }
```

or

```java
public static StreamingFileSink<OneLogEntity> init(String outputPath) {
        return StreamingFileSink
                .forRowFormat(new Path(outputPath), new SimpleStringEncoder<OneLogEntity>("UTF-8"))
                .withBucketAssigner(new HostDateTimeBucketAssigner())
                .withRollingPolicy(OnCheckpointRollingPolicy.build())
                .build();
    }
```

写入的文件有三种状态：in-process、in-pending、finshed，invoke方法里面正在写入的文件状态是in-process，当满足滚动策略之后将文件变为in-pending状态，执行sapshotState方法会对in-process状态文件执行commit操作，将缓存的数据刷进磁盘，并且记录其当前offset值，同时会记录in-pending文件的元数据信息，最终在notifyCheckpointComplete方法中将记录的in-pending状态文件转换为finshed的可读文件。如果中间程序出现异常则会通过initializeState完成恢复操作，将in-process文件恢复到记录的offset位置，直接恢复in-pending文件，并且将没有记录的in-pending文件删除、超过offset部分的数据删除，finshed文件无需操作。除了需要对offset、文件元数据信息保存外，还需要保存counter值，partFile命名组成的一部分，如果不保存则会造成重启后文件counter重新从0开始，会覆盖之前写入的文件



今天我们来看下StreamingFileSink的一些不一样的用法和具体源码解析



## 如何自定义路径

根据输入数据自定义文件输出路径，StreamingFileSink中withBucketAssigner方法，只要实现getBucketId方法，返回的BucketId就是对应的路径地址，可以参考源码DateTimeBucketAssigner

如果返回的是xxx/xxxx,那么也会变成多层路径

```java
@Override
    public String getBucketId(OneLogEntity element, BucketAssigner.Context context) {
        if (dateTimeFormatter == null) {
            dateTimeFormatter = DateTimeFormatter.ofPattern(formatString).withZone(zoneId);
        }
        long currentProcessingTime = context.currentProcessingTime();
        String time = dateTimeFormatter.format(Instant.ofEpochMilli(currentProcessingTime/1000 /300 * 300 *1000));
        return time+"/"+time + "-" + element.getHost();
    }
```


```

└── 202108121105
    └── 202108121105-host1
    	├── part-0-0
    	├── part-0-1
    	├── .part-0-2-inprogress
    
    └── 202108121105-host2
    	├── part-0-0
    	├── part-0-1
    	├── .part-0-2-inprogress
	
```

详情见源码

Buckets

```java
    private Path assembleBucketPath(BucketID bucketId) {
        final String child = bucketId.toString();
        if ("".equals(child)) {
            return basePath;
        }
        return new Path(basePath, child);
    }
```


## 回滚策略

* OnCheckpointRollingPolicy   
* DefaultRollingPolicy

OnCheckpointRollingPolicy在每次checkPoint的时候就会从inprogress变成finish。DefaultRollingPolicy是根据大小，时间来判断是否可以从inprogress到finish，一个inprogress中会有多个checkpoint数据



接口RollingPolicy有三个方法

```java
shouldRollOnCheckpoint
shouldRollOnEvent
shouldRollOnProcessingTime
```

OnCheckpointRollingPolicy很明显是shouldRollOnCheckpoint为true，其他皆为false，可以做到每个part文件都是一个checkpoint数据，做到excetly once。

DefaultRollingPolicy有三个回滚策略，一个是根据每个part的大小，这个会在每次写文件和每次chk的时候判断。一个是根据文件的时间，是否大于可以回滚的时候或者时间大于上一条数据写入的时间


```java
  @Override
    public boolean shouldRollOnCheckpoint(PartFileInfo<BucketID> partFileState) throws IOException {
        return partFileState.getSize() > partSize;
    }

    @Override
    public boolean shouldRollOnEvent(PartFileInfo<BucketID> partFileState, IN element)
            throws IOException {
        return partFileState.getSize() > partSize;
    }

    @Override
    public boolean shouldRollOnProcessingTime(
            final PartFileInfo<BucketID> partFileState, final long currentTime) {
        return currentTime - partFileState.getCreationTime() >= rolloverInterval
                || currentTime - partFileState.getLastUpdateTime() >= inactivityInterval;
    }

```




## 源码分析

行编码格式，主要返回的类是DefaultRowFormatBuilder，他的父类是RowFormatBuilder，有这么重要的参数


* long bucketCheckInterval 
* Path basePath 文件的落地路径
* Encoder encoder  文件落地的编码
* BucketAssigner bucketAssigner bucket的策略构造
* RollingPolicy rollingPolicy 文件的回滚策略
* BucketFactory bucketFactory 
* OutputFileConfig outputFileConfig 落地文件的名字前缀和后缀配置


StreamingFileSink基本都会相关的操作集合到StreamingFileSinkHelper中，而其实helper的操作全部集中到了Buckets中，Buckets用于管理所有bucket，一个bucket对应一个细致的路径。


首先StreamingFileSinkHelper继承了ProcessingTimeCallback，用于注册间隔bucketCheckInterval的processingTime触发任务。

```java
@Override
    public void onProcessingTime(long timestamp) throws Exception {
        final long currentTime = procTimeService.getCurrentProcessingTime();
        buckets.onProcessingTime(currentTime);
        procTimeService.registerTimer(currentTime + bucketCheckInterval, this);
    }
```

针对数据的invoke需要调用onElement，snapshotState调用snapshotState，notifyCheckpointComplete调用commitUpToCheckpoint。


### onElement

Buckets.java
```java
public Bucket<IN, BucketID> onElement(
            final IN value,
            final long currentProcessingTime,
            @Nullable final Long elementTimestamp,
            final long currentWatermark)
            throws Exception {
        // setting the values in the bucketer context
        bucketerContext.update(elementTimestamp, currentWatermark, currentProcessingTime);

        final BucketID bucketId = bucketAssigner.getBucketId(value, bucketerContext);
        final Bucket<IN, BucketID> bucket = getOrCreateBucketForBucketId(bucketId);
        bucket.write(value, currentProcessingTime);

        // we update the global max counter here because as buckets become inactive and
        // get removed from the list of active buckets, at the time when we want to create
        // another part file for the bucket, if we start from 0 we may overwrite previous parts.

        this.maxPartCounter = Math.max(maxPartCounter, bucket.getPartCounter());
        return bucket;
    }
```

先通过bucketAssigner获取这个数据的bucketId，通过id获取bucket，然后将数据写入bucket中，


### onProcessingTime

Buckets.java
```java
public void onProcessingTime(long timestamp) throws Exception {
        for (Bucket<IN, BucketID> bucket : activeBuckets.values()) {
            bucket.onProcessingTime(timestamp);
        }
    }
```

每个bucket路径，都要触发一次onProcessingTime,调用rollingPolicy.shouldRollOnProcessingTime方法，判断每个inProgress的part是否可以roll了，如果可以roll的part，就会调用closePartFile


Bucket.java
```java
void onProcessingTime(long timestamp) throws IOException {
        if (inProgressPart != null
                && rollingPolicy.shouldRollOnProcessingTime(inProgressPart, timestamp)) {
            if (LOG.isDebugEnabled()) {
                LOG.debug(
                        "Subtask {} closing in-progress part file for bucket id={} due to processing time rolling policy "
                                + "(in-progress file created @ {}, last updated @ {} and current time is {}).",
                        subtaskIndex,
                        bucketId,
                        inProgressPart.getCreationTime(),
                        inProgressPart.getLastUpdateTime(),
                        timestamp);
            }
            closePartFile();
        }
    }
```

### snapshotState

Buckets.java
```java

public void snapshotState(
            final long checkpointId,
            final ListState<byte[]> bucketStatesContainer,
            final ListState<Long> partCounterStateContainer)
            throws Exception {

        Preconditions.checkState(
                bucketWriter != null && bucketStateSerializer != null,
                "sink has not been initialized");

        LOG.info(
                "Subtask {} checkpointing for checkpoint with id={} (max part counter={}).",
                subtaskIndex,
                checkpointId,
                maxPartCounter);

        bucketStatesContainer.clear();
        partCounterStateContainer.clear();

        snapshotActiveBuckets(checkpointId, bucketStatesContainer);
        partCounterStateContainer.add(maxPartCounter);
    }
```

将当前active的bucket存入状态，和当前maxPartCounter存入状态，真是调用的是bucket.onReceptionOfCheckpoint(checkpointId)，将InProgressFileRecoverable存入状态，这个类中保存了文件的各种信息。

InProgressFileRecoverable的实现OutputStreamBasedInProgressFileRecoverable，有一个成员变量ResumeRecoverable，他有两个实现，HadoopFsRecoverable和LocalRecoverable。

```java
/** The file path for the final result file. */
    private final File targetFile;

    /** The file path of the staging file. */
    private final File tempFile;

    /** The position to resume from. */
    private final long offset;
```
通过上面三个变量，可以做到文件回放和回滚

### notifyCheckpointComplete


```java
public void commitUpToCheckpoint(long checkpointId) throws Exception {
        buckets.commitUpToCheckpoint(checkpointId);
    }
```

```java
bucket.onSuccessfulCompletionOfCheckpoint(checkpointId);
```

调用HadoopFsRecoverable和LocalRecoverable的具体实现
```
bucketWriter.recoverPendingFile(pendingFileRecoverable).commit();
```

把inProgrss转为finish文件





### Reference
https://ci.apache.org/projects/flink/flink-docs-release-1.13/zh/docs/connectors/datastream/streamfile_sink/

https://www.cnblogs.com/Springmoon-venn/p/11198196.html

https://blog.csdn.net/m0_49834705/article/details/114577330

http://shzhangji.com/cnblogs/2018/12/30/real-time-exactly-once-etl-with-apache-flink/


https://www.programmersought.com/article/97513705665/

https://toutiao.io/posts/a19c9s9/preview




https://www.cnblogs.com/Springmoon-venn/p/11198196.html
