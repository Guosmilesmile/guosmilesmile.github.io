---
title: BitMap和布隆过滤器(BloomFilter)
date: 2019-05-10 08:16:26
tags:
categories: Java
---

## BitMap

#### 介绍

Bitmap 也称之为 Bitset，它本质上是定义了一个很大的 bit 数组，每个元素对应到 bit 数组的其中一位。

例如有一个集合［2，3，5，8］对应的 Bitmap 数组是［001101001］，集合中的 2 对应到数组 index 为 2 的位置，3 对应到 index 为 3 的位置，下同，得到的这样一个数组，我们就称之为 Bitmap。

本溯源，我们的目的是用更小的存储去表示更多的信息，而在计算机最小的信息单位是 bit，如果能够用一个 bit 来表示集合中的一个元素，比起原始元素，可以节省非常多的存储。

这就是最基础的 Bitmap，我们可以把 Bitmap 想象成一个容器，我们知道一个 Integer 是32位的，如果一个 Bitmap 可以存放最多 Integer.MAX_VALUE 个值，那么这个 Bitmap 最少需要 32 的长度。一个 32 位长度的 Bitmap 占用的空间是512 M （2^32/8/1024/1024），这种 Bitmap 存在着非常明显的问题：这种 Bitmap 中不论只有 1 个元素或者有 40 亿个元素，它都需要占据 512 M 的空间

![image](https://note.youdao.com/yws/api/personal/file/E54DCA09EB6249488E0A59C91F3ED39F?method=download&shareKey=2d3956019a38eea421f6f2324a92f1eb)

#### 字符串映射
BitMap 也可以用来表述字符串类型的数据，但是需要有一层Hash映射，如下图，通过一层映射关系，可以表述字符串是否存在。

![image](https://note.youdao.com/yws/api/personal/file/26443B5D7E464435AC164F0D51329D98?method=download&shareKey=9944c97663076d646a69642f1e702417)

当然这种方式会有数据碰撞的问题，但可以通过 Bloom Filter 做一些优化。

## BloomFilter

### 工作原理

以WEB页面地址的存储为例来说明布隆过滤器的工作原理。


##### 存储过程
假定存储一亿个WEB页面地址，先建立一个2亿字节的向量，即16亿二进制（比特），然后将这16亿个二进制位清零。对于每一个WEB页面地址X，用8个随机数产生器（f1,f2,...,f8）。再用一个随机数产生器G把这8个信息指纹映射到1-16亿中的8个自然数g1,g2,...g8。现在把这8个位置的二进制位都置为1。对着一亿个WEB页面地址都进行这样的处理后，一个针对WEB页面的布隆过滤器就建成了
![image](https://note.youdao.com/yws/api/personal/file/BBAA1734F87A4CC6AD1118656E90754E?method=download&shareKey=9011f286303c81847e6883c89cc5ce77)

用相同的8个随机数生成器（f1,f2,...,f8）对这个WEB网页地址产生8个信息指纹s1,s2,...s8，然后将这8个指纹对应到布隆过滤器的8个二进制位，分别是t1,t2,...,t8。如果Y已被收录，显然t1,t2,...,t8对应的8个二进制位一定是1。通过这样的方式我们能够很快地确定一个WEB页面是否已被我们收录。

![image](https://note.youdao.com/yws/api/personal/file/E628F0685D244863A0DB0E9D5C83989C?method=download&shareKey=01dfca7cf5599bf6ad1c7cccd8b2ccd4)

##### 判断是否存在过程

通过n个hash函数，将需要判断的数据hash出n个数字，在bitMap中，这n个index的数据是否都是1，如果是，那么说明可能存在，如果存在一个0，那么这个数据确定不存在。

##### 优势

如果用哈希表，每存储一亿个 email地址，就需要 1.6GB的内存（用哈希表实现的具体办法是将每一个 email地址对应成一个八字节的信息指纹，然后将这些信息指纹存入哈希表，由于哈希表的存储效率一般只有 50%，因此一个 email地址需要占用十六个字节。一亿个地址大约要 1.6GB，即十六亿字节的内存）。因此存贮几十亿个邮件地址可能需要上百 GB的内存。而Bloom Filter只需要哈希表 1/8到 1/4 的大小就能解决同样的问题。


#### 使用

guava中有现成的工具类

```
    <dependencies>
        <dependency>
            <groupId>com.google.guava</groupId>
            <artifactId>guava</artifactId>
            <version>22.0</version>
        </dependency>
    </dependencies>
```
```java
    
import com.google.common.base.Charsets; 
import com.google.common.hash.BloomFilter; 
import com.google.common.hash.Funnel; 
import com.google.common.hash.Funnels; 
import com.google.common.hash.PrimitiveSink; 
import lombok.AllArgsConstructor; 
import lombok.Builder; 
import lombok.Data; 
import lombok.ToString; 
    
/**
 * BloomFilterTest
 */ 
public class BloomFilterTest { 
        
    public static void main(String[] args) { 
        long expectedInsertions = 10000000; 
        double fpp = 0.00001; 
    
        BloomFilter<CharSequence> bloomFilter = BloomFilter.create(Funnels.stringFunnel(Charsets.UTF_8), expectedInsertions, fpp); 
    
        bloomFilter.put("aaa"); 
        bloomFilter.put("bbb"); 
        boolean containsString = bloomFilter.mightContain("aaa"); 
        System.out.println(containsString); 
    
        BloomFilter<Email> emailBloomFilter = BloomFilter 
                .create((Funnel<Email>) (from, into) -> into.putString(from.getDomain(), Charsets.UTF_8), 
                        expectedInsertions, fpp); 
    
        emailBloomFilter.put(new Email("222.com", "111.com")); 
        boolean containsEmail = emailBloomFilter.mightContain(new Email("222.com", "111.com")); 
        System.out.println(containsEmail); 
    } 
    
    @Data 
    @Builder 
    @ToString 
    @AllArgsConstructor 
    public static class Email { 
        private String userName; 
        private String domain; 
    } 
    
} 
```

### Reference
https://my.oschina.net/LucasZhu/blog/1813110
https://www.cnblogs.com/z941030/p/9218356.html
https://cloud.tencent.com/developer/article/1006113