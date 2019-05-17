---
title: Assemble（系列一）大数据
date: 2019-05-01 10:24:31
tags:
categories: Assemble
---


## 常见压缩算法
Algorithm | % remaining| Encoding |Decoding
---|---|---|---|---
GZIP|13.4% |21 MB/s|118 MB/s
LZO |20.5%|135 MB/s|410 MB/s
Zippy/Snappy|22.2%|172 MB/s|409 MB/s
Lz4 | 大约等于snappy | snappy两倍 | snappy两倍

* GZIP的压缩率最高，但是是cpu密集型，对cpu的消耗要大很多，压缩也解压速度都满
* L4Z的压缩率和snappy差不多，但是速度是snappy的两倍。

## 列式存储压缩算法

1. 字典编码
将相同的值提取出来生成符号表（感觉就是建立索引表），每个列值则直接存储该值映射成的符号表值id（通过索引id的短来减少存储），但是如果量很大的时候，索引也会很大，等于没效果
![image](https://note.youdao.com/yws/api/personal/file/B85F71CC9F744B09A01F87975EB9036C?method=download&shareKey=2895c21b23226c54a0ac15c25f7600e5)
2. 常量编码
当区内的数据大部分的数据相同，只有少数不同时，可以采用常量编码。该编码将区内数据出现最多的一个值作为常量值，其他值作为异常值。异常值使用<行号+值>的方式存储。（将不一样的单独拎出来，加以行号）

![image](https://note.youdao.com/yws/api/personal/file/B92B6C9AA6A84FDA9C3D1BE986FF9871?method=download&shareKey=22fd310560a29024b5084b7771dfd214)
3. RLE编码（Run-Length Encoding）
当区内的数据存在大量的相同值，每个不同值的个数比较均匀，且连续出现时，可以使用RLE编码。（其核心思想是将一个有序列中相同的列属性值转化为三元组（列属性值，在列中第一次出现的位置，出现次数）
![image](https://note.youdao.com/yws/api/personal/file/8BE0BC6A377948B79240879DA6769877?method=download&shareKey=4a64a53555bc826a6977311c18480257)

![image](https://note.youdao.com/yws/api/personal/file/DE74F5653F89495BAD3D726E2EBEE4C7?method=download&shareKey=788642aba329f230d7c884ec1cc1cdb8)
4. 序列编码
当区内的数据差值成等差数列，或者存在一定的代数关系，则可以使用序列编码。
![image](https://note.youdao.com/yws/api/personal/file/AC551EF155914DDE8F3AFB4548B2E02E?method=download&shareKey=67752b545ab2722563b497965b603eed)

5. Bit-Vector Encoding
其核心思想是将一个列中所有相同列属性的值转化为二元组（列属性值，该列属性值出现在列中位置的Bitmap[Bitmap就是一个很大的数组，以01表示对应的数是否存在，海量数据的排序很有效，占用内存固定]）

![image](https://note.youdao.com/yws/api/personal/file/9BC6BA03C3424B4DBD81407088435DC1?method=download&shareKey=4ef9bcbc584bebcf254a26c4d1a4d6fa)

##### Reference
http://www.cnblogs.com/23lalala/p/5643541.html
https://blog.csdn.net/bitcarmanlee/article/details/50938970

## 缓存系列

简单缓存逻辑
![image](http://wx4.sinaimg.cn/large/8b2dfbcaly1g2nddmg0uxj20bh09kmy0.jpg)

#### 缓存穿透

如果去请求一条不存在的key，那么缓存和数据库都不存在这条记录，每次请求都会打到数据库上，这叫做缓存穿透（可以用来攻击）。

##### 避免
1. 缓存空值
可以为这些key对应的值设置为null 丢到缓存里面去。后面再出现查询这个key 的请求的时候，直接返回null 。
2. BloomFilter
在海量数据中，布隆过滤器里头可以选缓存数据库到底有什么key。
在缓存之前在加一层 BloomFilter ，在查询的时候先去 BloomFilter 去查询 key 是否存在，如果不存在就直接返回，存在再走查缓存 -> 查 DB

#### 缓存击穿

在高并发系统中，大量请求查询一个key，这个key又刚好失效，那么就会有大量的数据打到数据库中。

##### 解决
可以在第一个查询数据的请求上使用一个 互斥锁来锁住它。

其他的线程走到这一步拿不到锁就等着，等第一个线程查询到了数据，然后做缓存。后面的线程进来发现已经有缓存了，就直接走缓存。（可以利用guava中的机制，第一个请求阻塞等待，其他请求先获取旧的数据，等新的数据获取到后更新）

#### 缓存雪崩
当某一时刻发生大规模的缓存失效的情况，比如你的缓存服务宕机了，会有大量的请求进来直接打到DB上面。结果就是DB 扛不住，挂掉。

##### 解决

1. 使用集群缓存，保证缓存服务的高可用
2. Hystrix限流&降级
3. 开启Redis持久化机制，尽快恢复缓存集群



## 分库分表的数据库如何进行数据迁移

#### 停服扩容

1. 停止服务
2. 新建2*n个新库，并做好高可用
3. 进行数据迁移，把数据从n个库里select出来，insert到2*n个库里；（耗时最长）
4. 修改微服务的数据库路由配置，模n变为模2*n；
5. 微服务重启，连接新库重新对外提供服务；


#### 平滑扩容

##### 前提
每台db都有一个salve，作为高可用也好，扩容也好



db| ip映射 | 取模
---|---|---
db1 | ip0 |  0
db1 salve | 无  | 
db2 | ip1 |  1
db2 salve | 无 | 

##### 步骤一：修改配置。

* 数据库实例所在的机器做双虚ip：   
* * 原%2=0的库是虚ip0，现增加一个虚ip00
* * 原%2=1的库是虚ip1，现增加一个虚ip11
* 修改服务的配置，将2个库的数据库配置，改为4个库的数据库配置，修改的时候要注意旧库与新库的映射关系：
* *  %2=0的库，会变为%4=0与%4=2
* *  %2=1的部分，会变为%4=1与%4=3


db| ip映射 | 取模
---|---|---
db1 | ip0，ip00 |  0
db1 salve | ip0，ip00  |
db2 | ip1，ip11 |  1
db2 salve | ip1，ip11 | 


##### 步骤二：reload配置，实例扩容


reload可能是这么几种方式：
* 比较原始的，重启服务，读新的配置文件；
* 高级一点的，配置中心给服务发信号，重读配置文件，重新初始化数据库连接池；
 
db| ip映射 | 取模
---|---|---
db1 | ip0，ip00 |  0
db1 salve | ip0，ip00  | 2
db2 | ip1，ip11 |  1
db2 salve | ip1，ip11 | 3


##### 步骤三：收尾工作，数据收缩


* 把双虚ip修改回单虚ip；
* 解除旧的双主同步，让成对库的数据不再同步增加；
* 增加新的双主同步，保证高可用；
* 删除掉冗余数据，例如：ip0里%4=2的数据全部删除，只为%4=0的数据提供服务

db| ip映射 | 取模
---|---|---
db1 | ip0 |  0
new db1 salve |  | 
db2 | ip00  | 2
new db2 salve |  | 
db3 | ip1 |  1
new db3 salve |  | 
db4 | ip11 | 3
new db4 salve |  | 

##### Reference
https://mp.weixin.qq.com/s?__biz=MjM5ODYxMDA5OQ==&mid=2651962231&idx=1&sn=1b51d042c243f0b3ce0b748ddbcff865&chksm=bd2d0eab8a5a87bdcbe7dd08fb4c969ad76fa0ea00b2c78645db8561fd2a78d813d7b8bef2ac&mpshare=1&scene=1&srcid=&key=d9a46d47128ca0584390daedbbb1c39077582436967d7e2189f09b10441423d60e73732ec92855b3f85f6cb547616c14be51b004a0da4c46163e1cf8d0ead0630120007ed885a7a4d0cc383294ea8e15&ascene=1&uin=MjU3NDYyMjA0Mw%3D%3D&devicetype=Windows+10&version=62060739&lang=zh_CN&pass_ticket=8%2BAcIGVZ5r%2B3LMkUMz0mfS12OgN7SW%2B%2B1eeeqRcOcIFUut%2FZk5Lj0iFnfAWSN4HV
