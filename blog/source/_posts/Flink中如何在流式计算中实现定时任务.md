---
title: Flink中如何在流式计算中实现定时任务
date: 2020-02-06 17:01:23
tags:
categories:
	- Flink
---

## 背景

流式数据需要跟数据库中的数据进行join，也就是join维表。一般维表是不怎么变化的，不变化的维表可以通过全量加载到内存中直接进行关联，也可以通过异步io的形式访问外部存储，如果担心外部存储的并发压力可以选择第一种。

在我们这边的场景中，这份维表是一份1分钟变化一次的数据，需要每分钟去获取。

一种做法是写一个定时job讲数据库的维表发到kafka中，流式作业直接从kafka中获取维表join，带来的是增加一个作业的维护的部署。第二种就是下面的方法，自定义个source，这个source使用quartz定时触发。

### 例子

```java
public class QuartzSource extends RichSourceFunction {

    private static final Logger LOGGER = LoggerFactory.getLogger(QuartzSource.class);

    private Class clazz;

    private Scheduler scheduler;

    private Map<String, Object> parameter;

    private boolean cancel = false;

    public QuartzSource(Class clazz, Map<String, Object> parameter) {
        this.clazz = clazz;
        this.parameter = parameter;
        Class[] interfaces = clazz.getInterfaces();
        boolean findClass = false;
        for (Class anInterface : interfaces) {
            if (anInterface.equals(Job.class)) {
                findClass = true;
            }
        }
        if (!findClass) {
            throw new RuntimeException("Quartz job class is need!");
        }
    }


    @Override
    public void open(Configuration parameters) throws Exception {
        super.open(parameters);
        InitUtils.initSpringContext();
        StdSchedulerFactory schedFact = new StdSchedulerFactory();
        Properties props = new Properties();
        props.put("org.quartz.scheduler.instanceName", this.clazz.getName());//默认使用default，如果在一个工程中使用两个该source会有问题，需要自定义名称，以类名命名。
        props.put("org.quartz.threadPool.threadCount", "10");
        schedFact.initialize(props);
        try {
            scheduler = schedFact.getScheduler();
        } catch (SchedulerException e) {
            LOGGER.error("Init quartz failed.", e);
        }
    }


    @Override
    public void run(SourceContext ctx) throws Exception {
        try {
            while (true) {
                if (cancel) {
                    break;
                }
                if (scheduler == null) {
                    System.out.println();
                }
                if (!scheduler.isStarted()) {
                    JobDataMap map = new JobDataMap();
                    map.put("context", ctx);
                    map.put("parameter", parameter);
                    JobDetail job = newJob(this.clazz).setJobData(map).build();
                    LOGGER.info("className: " + this.getClass() + ", cron: {}, id of bolt: {}", ConstConfig.CRON_RULE, job.getKey());
                    Trigger trigger = newTrigger().startNow().withSchedule(cronSchedule(ConstConfig.CRON_RULE)).build();
                    scheduler.scheduleJob(job, trigger);
                    scheduler.start();
                }
                LOGGER.info("sched isStarted:" + scheduler.isStarted() + "," + scheduler.getCurrentlyExecutingJobs());
                Thread.sleep(10000);
            }
        } catch (Exception e) {
            LOGGER.error("job exception", e);
        }
    }

    @Override
    public void cancel() {
        cancel = true;
        try {
            scheduler.shutdown();
        } catch (SchedulerException e) {
            LOGGER.error("scheduler clear error!", e);
        }
    }
}


```

需要配置cancel信号量，在job停止的时候断开死循环，不然会出现问题。

具体的job可以按如下方式写

```java
public class QuatzSourceJob implements Job, Serializable {

private static final Logger LOGGER = LoggerFactory.getLogger(QuatzSourceJob.class);

  @Override
    public void execute(JobExecutionContext context) throws JobExecutionException {
        JobDataMap map = context.getJobDetail().getJobDataMap();
        SourceFunction.SourceContext collector = (SourceFunction.SourceContext) map.get("context");

        //do job
    }
}
```

可以通过如下方式使用

```java
Map<String,Object> paramter = new HashMap<>();

DataStream<Object> stream = env.addSource(new QuartzSource(QuatzSourceJob.class,parameter));


```
