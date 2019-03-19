---
title: zero copy
date: 2019-03-19 22:47:19
tags:
categories: kafka
---



#### Kafka在什么场景下使用该技术

使用Zero copy方式在内核层直接将文件内容传送给网络Socket，避免应用层数据拷贝，减小IO开销。

**消息消费的时候**

包括外部Consumer以及Follower 从partiton Leader同步数据，都是如此。简单描述就是：

Consumer从Broker获取文件数据的时候，直接通过下面的方法进行channel到channel的数据传输。

```
java.nio.FileChannel.transferTo(
long position, 
long count,                                
WritableByteChannel target)`
```
也就是说你的数据源是一个Channel,数据接收端也是一个Channel(SocketChannel),则通过该方式进行数据传输，是直接在内核态进行的，避免拷贝数据导致的内核态和用户态的多次切换。




#### zero-copy介绍
data从硬盘读出之后，原封不动的通过socket传输给用户。这种操作看起来可能不会怎么消耗CPU，但是实际上它是低效的：kernal把数据从disk读出来，然后把它传输给user级的application，然后application再次把同样的内容再传回给处于kernal级的socket。这种场景下，application实际上只是作为一种低效的中间介质，用来把disk file的data传给socket。


data每次穿过user-kernel boundary，都会被copy，这会消耗cpu，并且占用RAM的带宽。幸运的是，你可以用一种叫做Zero-Copy的技术来去掉这些无谓的copy。应用程序用zero copy来请求kernel直接把disk的data传输给socket，而不是通过应用程序传输。Zero copy大大提高了应用程序的性能，并且减少了kernel和user模式的上下文切换。


考虑一下这个场景，通过网络把一个文件传输给另一个程序。这个操作的核心代码就是下面的两个函数：

```java
File.read(fileDesc, buf, len);
Socket.send(socket, buf, len);
```

尽管看起来很简单，但是在OS的内部，这个copy操作要经历四次user mode和kernel mode之间的上下文切换，甚至连数据都被拷贝了四次！Figure 1描述了data是怎么移动的。



![image](https://note.youdao.com/yws/api/personal/file/AFEAC74A7C164243BAFF4F4897B4983C?method=download&shareKey=d827066e4f9c6d1e88eb0f94ef7d6b34)


Figure 2 描述了上下文切换


![image](https://note.youdao.com/yws/api/personal/file/3D77500E9C944F56ACC75E8BC39E01E2?method=download&shareKey=54c9fa6b44066d50a9e106c8fd2250ca)

其中的步骤如下：


read() 引入了一次从user mode到kernel mode的上下文切换。实际上调用了sys_read() 来从文件中读取data。第一次copy由DMA完成，将文件内容从disk读出，存储在kernel的buffer中。
然后data被copy到user buffer中，此时read()成功返回。这是触发了第二次context switch: 从kernel到user。至此，数据存储在user的buffer中。
send() socket call 带来了第三次context switch，这次是从user mode到kernel mode。同时，也发生了第三次copy：把data放到了kernel adress space中。当然，这次的kernel buffer和第一步的buffer是不同的两个buffer。
最终 send() system call 返回了，同时也造成了第四次context switch。同时第四次copy发生，DMA将data从kernel buffer拷贝到protocol engine中。第四次copy是独立而且异步的。
使用kernel buffer做中介(而不是直接把data传到user buffer中)看起来比较低效(多了一次copy)。然而实际上kernel buffer是用来提高性能的。在进行读操作的时候，kernel buffer起到了预读cache的作用。当写请求的data size比kernel buffer的size小的时候，这能够显著的提升性能。在进行写操作时，kernel buffer的存在可以使得写请求完全异步。


悲剧的是，当请求的data size远大于kernel buffer size的时候，这个方法本身变成了性能的瓶颈。因为data需要在disk，kernel buffer，user buffer之间拷贝很多次(每次写满整个buffer)。


而Zero copy正是通过消除这些多余的data copy来提升性能。
如果重新检查一遍traditional approach，你会注意到实际上第二次和第三次copy是毫无意义的。应用程序仅仅缓存了一下data就原封不动的把它发回给socket buffer。实际上，data应该直接在read buffer和socket buffer之间传输。transferTo()方法正是做了这样的操作。              

![image](https://note.youdao.com/yws/api/personal/file/A6156D3D90584AB9AF915574F5B851F1?method=download&shareKey=9533adac601c1e15590212f75a3d55c5)

Figure 4 展示了在使用transferTo()之后的上下文切换         
![image](https://note.youdao.com/yws/api/personal/file/07A04527FCD54C8BBC9292014AF74336?method=download&shareKey=a77251bb23679e382e54004a79bd4b05)

transferTo()方法使得文件内容被DMA engine直接copy到一个read buffer中。然后数据被kernel再次拷贝到和output socket相关联的那个kernel buffer中去。
第三次拷贝由DMA engine完成，它把kernel buffer中的data拷贝到protocol engine中。
这是一个很明显的进步：我们把context switch的次数从4次减少到了2次，同时也把data copy的次数从4次降低到了3次(而且其中只有一次占用了CPU，另外两次由DMA完成)。但是，要做到zero copy，这还差得远。如果网卡支持 gather operation，我们可以通过kernel进一步减少数据的拷贝操作。在2.4及以上版本的linux内核中，开发者修改了socket buffer descriptor来适应这一需求。这个方法不仅减少了context switch，还消除了和CPU有关的数据拷贝。user层面的使用方法没有变，但是内部原理却发生了变化：
transferTo()方法使得文件内容被copy到了kernel buffer，这一动作由DMA engine完成。
没有data被copy到socket buffer。取而代之的是socket buffer被追加了一些descriptor的信息，包括data的位置和长度。然后DMA engine直接把data从kernel buffer传输到protocol engine，这样就消除了唯一的一次需要占用CPU的拷贝操作。
Figure 5描述了新的transferTo()方法中的data copy:        
![image](https://note.youdao.com/yws/api/personal/file/5DF3DA7C08CA4130B42C7DDDE783D0CB?method=download&shareKey=0c8032eac932d4dc1fb8fb74572a009b)

