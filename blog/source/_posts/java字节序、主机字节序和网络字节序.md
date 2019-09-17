---
title: java字节序、主机字节序和网络字节序
date: 2019-09-17 23:35:48
tags:
categories:
	- Java
---


计算机硬件有两种储存数据的方式：大端字节序（big endian）和小端字节序（little endian）。

举例来说，数值0x2211使用两个字节储存：高位字节是0x22，低位字节是0x11。


```java
大端字节序：高位字节在前，低位字节在后，这是人类读写数值的方法。
小端字节序：低位字节在前，高位字节在后，即以0x1122形式储存。
```

同理，0x1234567的大端字节序和小端字节序的写法如下图。

![image](https://note.youdao.com/yws/api/personal/file/E691216E78614EDE94E60DBCCEDAD627?method=download&shareKey=ead114f4fa539399c0eade3135d0ab6e)

至于计算机到底是BIG-ENDIAN、LITTLE-ENDIAN、跟CPU有关的，一种CPU不是BIG-ENDIAN就是LITTLE-ENDIAN。IA架构(Intel、AMD)的CPU中是Little-Endian，而PowerPC 、SPARC和Motorola处理器是Big-Endian。这其实就是所谓的主机字节序。而网络字节序是指数据在网络上传输时是大头还是小头的，在Internet的网络字节序是BIG-ENDIAN。所谓的JAVA字节序指的是在JAVA虚拟机中多字节类型数据的存放顺序，JAVA字节序也是BIG-ENDIAN。可见网络和JVM都采用的是大字节序，个人感觉就是因为这种字节序比较符合人类的习惯。由于JVM会根据底层的操作系统和CPU自动进行字节序的转换，所以我们使用java进行网络编程，几乎感觉不到字节序的存在。


### 为什么会有小端字节序？
答案是，计算机电路先处理低位字节，效率比较高，因为计算都是从低位开始的。所以，计算机的内部处理都是小端字节序。

但是，人类还是习惯读写大端字节序。所以，除了计算机的内部处理，其他的场合几乎都是大端字节序，比如网络传输和文件储存。


### "只有读取的时候，才必须区分字节序，其他情况都不用考虑。"

处理器读取外部数据的时候，必须知道数据的字节序，将其转成正确的值。然后，就正常使用这个值，完全不用再考虑字节序。

即使是向外部设备写入数据，也不用考虑字节序，正常写入一个值即可。外部设备会自己处理字节序的问题。


### 转换

举例来说，处理器读入一个16位整数。如果是大端字节序，就按下面的方式转成值。

```
x = buf[offset] * 256 + buf[offset+1];
上面代码中，buf是整个数据块在内存中的起始地址，offset是当前正在读取的位置。第一个字节乘以256，再加上第二个字节，就是大端字节序的值，这个式子可以用逻辑运算符改写。


x = buf[offset]<<8 | buf[offset+1];
上面代码中，第一个字节左移8位（即后面添8个0），然后再与第二个字节进行或运算。

如果是小端字节序，用下面的公式转成值。


x = buf[offset+1] * 256 + buf[offset];
32位整数的求值公式也是一样的。


/* 大端字节序 */
i = (data[3]<<0) | (data[2]<<8) | (data[1]<<16) | (data[0]<<24);

/* 小端字节序 */
i = (data[0]<<0) | (data[1]<<8) | (data[2]<<16) | (data[3]<<24);

```

### java中的字节序

那么java里面，怎么判断你的计算机是大端存储、还是小端存储呢？JDK为我们提供一个类ByteOrder，通过以下代码就可以知道机器的字节序
```java
System.out.println(ByteOrder.nativeOrder());
```

### Reference

https://blog.csdn.net/aitangyong/article/details/23204817

http://www.ruanyifeng.com/blog/2016/11/byte-order.html