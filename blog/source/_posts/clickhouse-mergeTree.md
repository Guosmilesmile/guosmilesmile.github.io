---
title: clickhouse MergeTree族原理和使用场景
date: 2019-03-18 22:26:19
tags:
categories: clickhouse
---

### MergeTree
一个mergetree类型的表必须有一个包含date类型的列（类型：Date），这个表是由很多个part构成。每一个part按照主键进行了排序，除此之外，每一个part含有一个最小日期和最大日期。当插入数据的时候，会创建一个新的sort part，同时会在后台周期性的进行merge的过程，当merge的时候，很多个part会被选中，通常是最小的一些part，然后merge成为一个大的排好序的part。

换句话说，整个这个合并排序的过程是在数据插入表的时候进行的。这个merge会导致这个表总是由少量的排序好的part构成，而且这个merge本身并没有做特别多的工作。
 在插入数据的过程中，属于不同的month的数据会被分割成不同的part，这些归属于不同的month的part是永远不会merge到一起的。新版本支持按天分表，partition by 一个指定的Date字段。
 向 MergeTree 表中插入数据时，引擎会首先对新数据执行递增排序而保存索引块；其后，数据索引块之间又会进一步合并，以减少总体索引块数量。 因此，合并过程本身并无过多排序工作。
 索引块合并时设有体积上限，以避免索引块合并产生庞大的新索引块

对于每一个part，会生成一个索引文件。这个索引文件存储了表里面每一个索引块里数据的主键的value值，换句话说，这是个对有序数据的小型索引。

  对列来说，在每一个索引块里的数据也写入了标记，从而让数据可以在明确的数值范围内被查找到。
  
  
一个mergetree类型的表必须有一个包含date类型的列（类型：Date），这个表是由很多个part构成。每一个part按照主键进行了排序，除此之外，每一个part含有一个最小日期和最大日期。当插入数据的时候，会创建一个新的sort part，同时会在后台周期性的进行merge的过程，当merge的时候，很多个part会被选中，通常是最小的一些part，然后merge成为一个大的排好序的part。换句话说，整个这个合并排序的过程是在数据插入表的时候进行的。这个merge会导致这个表总是由少量的排序好的part构成，而且这个merge本身并没有做特别多的工作。 索引块合并时设有体积上限，以避免索引块合并产生庞大的新索引块

在插入数据的过程中，属于不同的month的数据会被分割成不同的part，这些归属于不同的month的part是永远不会merge到一起的。新版本支持按天分表，partition by 一个指定的Date字段。 向 MergeTree 表中插入数据时，引擎会首先对新数据执行递增排序而保存索引块；其后，数据索引块之间又会进一步合并，以减少总体索引块数量。对于每一个part，会生成一个索引文件。这个索引文件存储了表里面每一个索引块里数据的主键的value值，换句话说，这是个对有序数据的小型索引。因此，合并过程本身并无过多排序工作。


### 建表语句
新版本支持按天分partition，建群的搭建方式一般是使用local表和分布式表组成，分布式表不存储数据，本地表存储数据。
```sql
CREATE TABLE mydb.mytable_cluster (  host String,  count Int64 statistic_time UInt64,  create_time UInt64,  stat_day Date) ENGINE = Distributed(clickhouse_cluster, 'mydb', 'mytable_cluster_local', rand())


```

```sql
CREATE TABLE mydb.mytable_cluster_local ( host String,  count Int64 statistic_time UInt64,  create_time UInt64,  stat_day Date) ENGINE = MergeTree PARTITION BY stat_day ORDER BY (statistic_time, host) SETTINGS index_granularity = 8192
```
### 底层文件存储

column_name.mrk：每个列都有一个mrk文件
column_name.bin：每个列都有一个bin文件，里边存储了压缩后的真实数据
primary.idx：主键文件，存储了主键值

主键自身是"稀疏的"。它不定位到每个行 ，但是仅是一些数据范围。 对于每个N-th行， 一个单独的primary.idx 文件有主键的值, N 被称为 index_granularity(通常情况下, N = 8192). 每8192⾏行行，抽取一行数据形成稀疏索引.每一部分按照主键顺序存储数据 (数据通过主键 tuple 来排序). 所有的表的列都在各自的column.bin文件中保存.mrk是针对每个列的，对每个列来说，都有一个mrk，记录的这个列的偏移量。

例如有如下数据，主键为x和y

x | y | z
---|---|---
A | a  | 1
A | a  | 2 
A | c  | 1
B | c  | 1
B | c  | 2
C | a  | 3 
C | a  | 1 

假设index_granularity为2，先将数据分为多个block

x | y | block-id
---|---|---
A | a | 1
A | a | 1
A | c | 2
B | c | 2
B | b | 3
C | a | 3
C | a | 4  

primary.idx内容展示的是主键和block的关系

主键 | block
---|---
(A,a) | 1
(A,c) | 2
(B,b) | 3
(C,a) | 4


x.bin 和 y.bin存储对应的各个列的数据，x.mrk存储如下

block-id | offset
---|---
1 | 1-3
2 | 4-9
3 | 10-30

### 查询过程

1、查询条件
2、通过查询条件的主键，可以根据primary.idx得出数据落在哪些block
3、根据block id 到各自的 mrk上寻找对应的offset
4、根据offset去bin中获取对应的数据，加载到内存中向量化操作，过滤



#### Attention
* 务必加上partition的条件去查询
* 稀疏索引会读取很多不必要的数据：读取主键的每一个部分，会多读取index_granularity * 2的数据。这对于稀疏索引来说很正常，也没有必要减少index_granularity的值.ClickHouse的设计，致力于高效的处理海量数据，这就是为什么一些多余的读取并不会有损性能。


  
### SummingMergeTree引擎测试
* 该引擎会把索引以为的所有number型字段（包含int和double）自动进行聚合
* 该引擎在分布式情况下并不是完全聚合，而是每台机器有一条同纬度的数据。SummingMergeTree是按part纬度来聚合，数据刚导入clickhouse可能会产生多个part，但是clickhouse会定期把part merge，从而实现一台机器只有一条同纬度的数据。
* 还未测试该引擎对导入性能有多大影响
* 如果将字段设为索引，则不会继续聚合，对于非设为索引的字段，如果是int类型会进行聚合，非int类型，会随机选取一个字段进行覆盖。（不正规测试，第一次进行merge的时候，会取第一条的字段，后续的merge都以第一次的为主）

##### 使用场景
在数据分析的时候，无法将数据完全聚合的时候可以考虑使用，例如使用jstorm分析数据，由于jstorm是无状态分析，暂留内存压力过大，或者数据延迟太大。可以考虑在不完全聚合的条件下使用该引擎，数据库进行聚合，但是查询还是得用group by！

### 建表语句
新版本支持按天分partition，建群的搭建方式一般是使用local表和分布式表组成，分布式表不存储数据，本地表存储数据。
```sql
CREATE TABLE mydb.mytable_cluster (  host String,  count Int64 statistic_time UInt64,  create_time UInt64,  stat_day Date) ENGINE = Distributed(clickhouse_cluster, 'mydb', 'mytable_cluster_local', rand())


```

```sql
CREATE TABLE mydb.mytable_cluster_local ( host String,  count Int64 statistic_time UInt64,  create_time UInt64,  stat_day Date) ENGINE = SummingMergeTree PARTITION BY stat_day ORDER BY (statistic_time, host) SETTINGS index_granularity = 8192
```
如上可以实现相同statistic_time，相同host，count累加的场景，由于create_time不是索引，也会被累加，如果出现别的String不在索引内，会被随机覆盖。

### ReplacingMergeTree
* 该引擎会把相同索引的数据进行替换，但仅限单台机器。如果使用分布式表，就要确保相同索引的数据入到同一台机器，否则每台机器可能会有一条相同索引的数据。
* 该索引只有在merge的时候才会执行替换，因为merge是不定时的，如果没有merge的情况下，会出现多条数据的情况。因此必要的话，可以进行手动进行merge。手动merge命令：optimize table db.table;
* 该索引的建表语句如果没有用某个字段标定版本，该字段可以是int、double、date类型，数据库就一定会把后入库的覆盖新入库(如果有区分版本的字段，则会留下数值大的那条记录)。

