---
title: kafka中使用的mmap和page cache
date: 2020-02-11 14:51:33
tags:
categories:
	- Kafka
---
## 传统文件IO操作的多次数据拷贝问题

首先，假设我们有一个程序，这个程序需要对磁盘文件发起IO操作读取他里面的数据到自己这儿来，那么会经过以下一个顺序：



首先从磁盘上把数据读取到内核IO缓冲区里去，然后再从内核IO缓存区里读取到用户进程私有空间里去，然后我们才能拿到这个文件里的数据





![image](https://note.youdao.com/yws/api/personal/file/D02F5B041BF44E9DB3841E76BF5D7BDD?method=download&shareKey=3a7a82823302e54eb4ca1126c3bad64b)

为了读取磁盘文件里的数据，是不是发生了两次数据拷贝？



没错，所以这个就是普通的IO操作的一个弊端，必然涉及到两次数据拷贝操作，对磁盘读写性能是有影响的。



那么如果我们要将一些数据写入到磁盘文件里去呢？



那这个就是一样的过程了，必须先把数据写入到用户进程私有空间里去，然后从这里再进入内核IO缓冲区，最后进入磁盘文件里去



我们看下面的图

![image](https://note.youdao.com/yws/api/personal/file/B75D7781A04D496BBF29C8FC92094794?method=download&shareKey=0f8417c7a3eeb08d19bc51b861f537b2)



在数据进入磁盘文件的过程中，是不是再一次发生了两次数据拷贝？没错，所以这就是传统普通IO的问题，有两次数据拷贝问题。



## mmap

Mmap（Memory Mapped Files，内存映射文件）

Mmap 方法为我们提供了将文件的部分或全部映射到内存地址空间的能力，同当这块内存区域被写入数据之后[dirty]，操作系统会用一定的算法把这些数据写入到文件中

其实有的人可能会误以为是直接把那些磁盘文件里的数据给读取到内存里来了，类似这个意思，但是并不完全是对的。


因为刚开始你建立映射的时候，并没有任何的数据拷贝操作，其实磁盘文件还是停留在那里，只不过他把物理上的磁盘文件的一些地址和用户进程私有空间的一些虚拟内存地址进行了一个映射


**这个mmap技术在进行文件映射的时候，一般有大小限制，在1.5GB~2GB之间**
所以在很多消息中间件，会限制文件的大小。


![image](https://note.youdao.com/yws/api/personal/file/A422A95D5B9C40E6B337EAF27FAC7D48?method=download&shareKey=c17ca215395a115fbc385d62bc487bc5)


**PageCache，实际上在这里就是对应于虚拟内存**


接下来就可以对这个已经映射到内存里的磁盘文件进行读写操作了，比如要写入消息到文件，你先把一文件通过MappedByteBuffer的map()函数映射其地址到你的虚拟内存地址。


接着就可以对这个MappedByteBuffer执行写入操作了，写入的时候他会直接进入PageCache中，然后过一段时间之后，由os的线程异步刷入磁盘中，如下图我们可以看到这个示意。
![image](https://note.youdao.com/yws/api/personal/file/391FE4C329DD4425B7643AA63C7FC110?method=download&shareKey=f6be0ee3616c35be6cd8d383e197acc9)

上面的图里，似乎只有一次数据拷贝的过程，他就是从PageCache里拷贝到磁盘文件里而已！这个就是你使用mmap技术之后，相比于传统磁盘IO的一个性能优化。

而且PageCache技术在加载数据的时候，还会将你加载的数据块的临近的其他数据块也一起加载到PageCache里去。
![image](https://note.youdao.com/yws/api/personal/file/DFF98763013D48B58CCCE9C0A6A27F04?method=download&shareKey=8ab44aa2144ee1555968af8ff69e7264)

所以kafka的顺序读写，在pageCache中可以有很好的利用。在写实时数据和消费实时数据，都可以从内存中直接消费，性能会提高（消费历史数据就不得不从磁盘重新加载到page cache，而且会污染掉实时数据的page cache）

具体事例



```java
(1)RandomAccessFile raf = new RandomAccessFile (File, "rw");

(2)FileChannel channel = raf.getChannel();

(3)MappedByteBuffer buff = channel.map(FileChannel.MapMode.READ_WRITE,startAddr,SIZE);

(4)buff .put((byte)255);

(5)buff.write(byte[] data)
```
```java

/**
 * 使用直接内存映射读取文件
 * @param file
 */
public static void fileReadWithMmap(File file) {

    long begin = System.currentTimeMillis();
    byte[] b = new byte[BUFFER_SIZE];
    int len = (int) file.length();
    MappedByteBuffer buff;
    try (FileChannel channel = new FileInputStream(file).getChannel()) {
        // 将文件所有字节映射到内存中。返回MappedByteBuffer
        buff = channel.map(FileChannel.MapMode.READ_ONLY, 0, channel.size());
        for (int offset = 0; offset < len; offset += BUFFER_SIZE) {
            if (len - offset > BUFFER_SIZE) {
                buff.get(b);
            } else {
                buff.get(new byte[len - offset]);
            }
        }
    } catch (IOException e) {
        e.printStackTrace();
    }
    long end = System.currentTimeMillis();
    System.out.println("time is:" + (end - begin));
}
```

其中最重要的就是那个buff，它是文件在内存中映射的标的物，通过对buff的read/write我们就可以间接实现对于文件的读写操作，当然写操作是操作系统帮忙完成的。


## 总结
 
mmap带来的最大的好处是虚拟内存的映射，较少一次io操作，但是本身也有局限，一般有大小限制，在1.5GB~2GB之间。


对虚拟内存进行读写的时候，会引入page cache的功能。

### Reference

[RocketMQ 如何基于mmap+page cache实现磁盘文件的高性能读写](https://mp.weixin.qq.com/s?__biz=MzU0OTk3ODQ3Ng==&mid=2247487013&idx=1&sn=1d05ed6d7aefe2a76fe024b34050343d&chksm=fba6e626ccd16f30dc19dc66323c39995a9e76e5ca897dbb2a3f97e725ed3faf37eb52e41888&scene=0&xtrack=1#rd)