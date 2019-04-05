---
title: clickhouse 数组
date: 2019-03-17 15:47:28
categories: clickhouse
---



#### 数据的集合
可以是各种类型，但是数组内的数据必须是同一种类型


#### 创建数组

可以通过array(T)或者[]创建数组

```
array(1, 2)
 [1, 2]

array('1','2')
 ['1','2']
```

#### 建表语句
```
create table test.testTable
(id Int64, app Array(string), all_count Int64)
ENGINE = MergeTree(stat_day,id,8192)
```
```
insert into test.xxx  values (1,['1','2'],30).
```

##### clickhouse的数组在使用上是一般字段一致，select出来看不到对应的详细内容，需要通过toString 方法将其转为字符串。

##### 如果需要判断某个值是否在数组内，可以通过has(array,element)的方法判断，后者是通过hasAny(array,array)的方法来判断,等价于in


##### 如果需要进行group by，需要将这个数组进行arrayJoin

```
select sum(count),arrayJoin(isp) from xx where arrayJoin(isp) in (?,?) group by arrayJoin(isp)

```

### Attention

使用ArrayJoin的时候，如果直接进行数据求和，会出现数据重复计算的场景


 A | count
---|---
['1','2'] | 10
['2','3'] | 10
['3','4'] | 10
['5','6'] | 10
['5','6'] | 10

过滤出A中有5、6的 count总和

正确答案是20

如果使用arrayJoin
```
select sum(count) from table where arrayJoin(A) in ('5','6')

```
结果查出来却是40...

因为arrayJoin将数据拆成了4条，每条都是10，求和就是40.


应该要用

```
select sum(count) from table where hasAny(A,['5','6'])

```
#### 总结
* arrayJoin是将数据拆分成多条，会导致数据重复
* 如果需要拆分使用arrayJoin，请加上group by 这个字段，才可以进行数值操作
* 使用hasAny可以进行数值计算，因为数组还没拆分


其他数组操作函数详见

https://clickhouse.yandex/docs/en/query_language/functions/array_functions/

