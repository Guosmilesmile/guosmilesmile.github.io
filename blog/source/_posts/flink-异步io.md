---
title: Flink 异步io
date: 2019-04-05 19:32:00
tags:
categories:
    - Flink
---


### 流程
异步io就是将一条条的记录同步与外部系统交互，变成并发的访问外部io。并不会将整个拓扑的次序打乱。


![image](https://note.youdao.com/yws/api/personal/file/FBB25A7C831A440E8E277105AC26AB4B?method=download&shareKey=79c7e084232004a26c44e76e38a5e5e5)


#### 重要提示
ResultFuture在第一次通话时完成ResultFuture.complete。所有后续complete调用都将被忽略。

#### 参数

* 超时：超时定义异步请求在被视为失败之前可能需要多长时间。此参数可防止死/失败的请求。
* 容量：此参数定义可能同时有多少异步请求正在进行中。尽管异步I / O方法通常会带来更好的吞吐量，但算子仍然可能成为流应用程序的瓶颈。限制并发请求的数量可确保算子不会累积不断增加的待处理请求积压，但一旦容量耗尽就会触发反压。

#### 超时处理
当异步I / O请求超时时，默认情况下会引发异常并重新启动作业。如果要处理超时，可以覆盖该AsyncFunction#timeout方法。

#### 结果顺序
AsyncDataStream 有两个静态方法，orderedWait 和 unorderedWait，对应了两种输出模式：有序和无序。

* 有序：消息的发送顺序与接受到的顺序相同（包括 watermark ），也就是先进先出。
* 无序：
在 ProcessingTime 的情况下，完全无序，先返回的结果先发送。
在 EventTime 的情况下，watermark 不能超越消息，消息也不能超越 watermark，也就是说 watermark 定义的顺序的边界。在两个 watermark 之间的消息的发送是无序的，但是在watermark之后的消息不能先于该watermark之前的消息发送。

```java
public class AsyncDatabaseRequest extends RichAsyncFunction<PlayerCountEvent, PlayerCountEvent> {

  
    @Override
    public void open(Configuration parameters) throws Exception {
        super.open(parameters);
    }

    @Override
    public void close() throws Exception {
        super.close();
        executorService.shutdown();
    }

    @Override
    public void asyncInvoke(PlayerCountEvent input, ResultFuture<PlayerCountEvent> resultFuture) throws Exception {
        
        CompletableFuture.supplyAsync(new Supplier<PlayerCountEvent>() {
            @Override
            public PlayerCountEvent get() {
                try {
                   // do something in the thread 
                   return input;
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } catch (ExecutionException e) {
                    e.printStackTrace();
                }
                return null;
            }

        }).thenAccept(new Consumer<PlayerCountEvent>() {
            @Override
            public void accept(PlayerCountEvent playerCountEvent) {
                resultFuture.complete(Collections.singleton(playerCountEvent));
            }
        });


    }
}
```

#### 警告
##### AsyncFunction不称为多线程
AsyncFunction是不以多线程方式调用。只存在一个实例，AsyncFunction并且对于流的相应分区中的每个记录顺序调用它
以下模式会导致阻塞asyncInvoke(...)函数，从而使异步行为无效：
* 使用其查找/查询方法调用阻塞的数据库客户端，直到收到结果为止
* 在asyncInvoke(...) 方法内阻塞等待一个异步客户端返回futureObject


### 疑惑自解

异步io与在flatmap中写多线程有什么区别呢？

1. 在flatmap中使用多线程，与异步io的无序的场景结果应该是一致的
2. 如果需要实现有序的场景，flatmap中无法实现。
3. 同时是无序的场景，异步io有限定容量大小与超时时间，可以防止假死，如果处理效率不够会出现反压，如果使用flatmap，无法直观的观察到上述现象。

### 参考

https://blog.csdn.net/rlnlo2pnefx9c/article/details/83829452

https://blog.csdn.net/baifanwudi/article/details/85074112

http://wuchong.me/blog/2017/05/17/flink-internals-async-io/