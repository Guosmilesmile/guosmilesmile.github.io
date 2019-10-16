---
title: java堆外内存
date: 2019-09-25 22:33:51
tags:
categories:
	- Java
---


### 堆内内存（on-heap memory）

堆外内存和堆内内存是相对的二个概念，其中堆内内存是我们平常工作中接触比较多的，我们在jvm参数中只要使用-Xms，-Xmx等参数就可以设置堆的大小和最大值
### 堆外内存（off-heap memory）

和堆内内存相对应，堆外内存就是把内存对象分配在Java虚拟机的堆以外的内存，这些内存直接受操作系统管理（而不是虚拟机），这样做的结果就是能够在一定程度上减少垃圾回收对应用程序造成的影响。

DirectByteBuffer类是在Java Heap外分配内存，对堆外内存的申请主要是通过成员变量unsafe来操作，下面介绍构造方法



```java
DirectByteBuffer(int cap) {                 

        super(-1, 0, cap, cap);
        //内存是否按页分配对齐
        boolean pa = VM.isDirectMemoryPageAligned();
        //获取每页内存大小
        int ps = Bits.pageSize();
        //分配内存的大小，如果是按页对齐方式，需要再加一页内存的容量
        long size = Math.max(1L, (long)cap + (pa ? ps : 0));
        //用Bits类保存总分配内存(按页分配)的大小和实际内存的大小
        Bits.reserveMemory(size, cap);

        long base = 0;
        try {
           //在堆外内存的基地址，指定内存大小
            base = unsafe.allocateMemory(size);
        } catch (OutOfMemoryError x) {
            Bits.unreserveMemory(size, cap);
            throw x;
        }
        unsafe.setMemory(base, size, (byte) 0);
        //计算堆外内存的基地址
        if (pa && (base % ps != 0)) {
            // Round up to page boundary
            address = base + ps - (base & (ps - 1));
        } else {
            address = base;
        }
        cleaner = Cleaner.create(this, new Deallocator(base, size, cap));
        att = null;
    }
```
 如果按页对齐     
 address = base + ps - (base & (ps - 1)) 等价于 (base + ps) / ps * ps
 假设page=4，从0开始，分配的base为6，那么计算可得address为8，第三页的头。
 
 ### DirectByteBuffer的回收
 Java的堆外内存回收设计是这样的：当GC发现DirectByteBuffer对象变成垃圾时，会调用Cleaner#clean回收对应的堆外内存，一定程度上防止了内存泄露。当然，也可以手动的调用该方法，对堆外内存进行提前回收。
 
```java
public class Cleaner extends PhantomReference<Object> {
   ...
    private Cleaner(Object referent, Runnable thunk) {
        super(referent, dummyQueue);
        this.thunk = thunk;
    }
    public void clean() {
        if (remove(this)) {
            try {
                //thunk是一个Deallocator对象
                this.thunk.run();
            } catch (final Throwable var2) {
              ...
            }

        }
    }
}

private static class Deallocator
    implements Runnable
    {

        private static Unsafe unsafe = Unsafe.getUnsafe();

        private long address;
        private long size;
        private int capacity;

        private Deallocator(long address, long size, int capacity) {
            assert (address != 0);
            this.address = address;
            this.size = size;
            this.capacity = capacity;
        }

        public void run() {
            if (address == 0) {
                // Paranoia
                return;
            }
            //调用unsafe方法回收堆外内存
            unsafe.freeMemory(address);
            address = 0;
            Bits.unreserveMemory(size, capacity);
        }

    }
```
 
### 堆外内存的优点


* 减少了垃圾回收的工作，因为垃圾回收会暂停其他的工作（可能使用多线程或者时间片的方式，根本感觉不到） 。
* 加快了复制的速度。因为堆内在flush到远程时，会先复制到直接内存（非堆内存），然后在发送；而堆外内存相当于省略掉了这个工作。 (用堆外内存只需拷贝一次，而用堆内存是要拷贝两次（堆->堆外->内核）)
* 可以在进程间共享，减少JVM间的对象复制，使得JVM的分割部署更容易实现。
* 可以扩展至更大的内存空间。
　　


利用nio的DirectByteBuffers实现，比存储到磁盘上快，而且完全不受GC的影响，可以保证响应时间的稳定性；但是direct buffer的在分配上的开销要比heap buffer大，而且要求必须以字节数组方式存储，因此对象必须在存储过程中进行序列化，读取则进行反序列化操作，它的速度大约比堆内存储慢一个数量级。

（注：direct buffer不受GC影响，但是direct buffer归属的的JAVA对象是在堆上且能够被GC回收的，一旦它被回收，JVM将释放direct buffer的堆外空间。）”

##### 堆内在flush到远程时，会先复制到直接内存（非堆内存），然后在发送的说明

HeapByteBuffer与DirectByteBuffer，在原理上，前者可以看出分配的buffer是在heap区域的，其实真正flush到远程的时候会先拷贝得到直接内存，再做下一步操作（考虑细节还会到OS级别的内核区直接内存），其实发送静态文件最快速的方法是通过OS级别的send_file，只会经过OS一个内核拷贝，而不会来回拷贝；在NIO的框架下，很多框架会采用DirectByteBuffer来操作，这样分配的内存不再是在java heap上，而是在C heap上，经过性能测试，可以得到非常快速的网络交互，在大量的网络交互下，一般速度会比HeapByteBuffer要快速好几倍。


### 堆外内存缺点
* 堆外内存难以控制，如果内存泄漏，那么很难排查 
* 堆外内存相对来说，不适合存储很复杂的对象。一般简单的对象或者扁平化的比较适合。



```java
Field f = Unsafe.class.getDeclaredField("theUnsafe");
        f.setAccessible(true);
        Unsafe unsafe = (Unsafe) f.get(null);

        Field ADDRESS_FIELD = java.nio.Buffer.class.getDeclaredField("address");
        ADDRESS_FIELD.setAccessible(true);

        ByteBuffer buffer = ByteBuffer.allocateDirect(8);
        Long address = (Long) ADDRESS_FIELD.get(buffer);
        System.out.println(address);
        buffer.putLong(50000L);
        System.out.println("use ByteBuffer.getLong:" + buffer.getLong(buffer.position() - 8));
        System.out.println(ByteOrder.nativeOrder());
        long aLong = unsafe.getLong(null, address);
        System.out.println("use unsafe.getLong:" + aLong);
        System.out.println("reverseBytes :" + Long.reverseBytes(aLong));
```
```java
493583600
use ByteBuffer.getLong:50000
LITTLE_ENDIAN
use unsafe.getLong:5819495143492812800
 reverseBytes :50000
```

默认的使用的是小端序，所以get出来是5819495143492812800，需要通过Long.reverseBytes后才可以看到人肉眼能识别的结果。
### Reference

https://www.cnblogs.com/duanxz/p/3141647.html

https://www.jianshu.com/p/50be08b54bee

https://www.cnblogs.com/duanxz/p/3141647.html

https://juejin.im/post/5be538fff265da611b57da10