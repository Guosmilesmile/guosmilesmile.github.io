---
title: Apache Kylin性能优化
date: 2020-08-16 22:52:59
tags:
categories: Kylin
---




## 高级设置

Apache Kylin 的主要工作就是为源数据构建 N 个维度的 Cube，实现聚合的预计算。理论上而言，构建 N 个维度的 Cube 会生成 2的N次方  个 Cuboid， 如图 1 所示，构建一个 4 个维度（A，B，C, D）的 Cube，需要生成 16 个Cuboid


![image](https://note.youdao.com/yws/api/personal/file/EC32CD207C734D5EB17DBE58DA39B992?method=download&shareKey=51fd59d977cb7bf931e1d9c3237b6e17)


随着维度数目的增加 Cuboid 的数量会爆炸式地增长，不仅占用大量的存储空间还会延长 Cube 的构建时间。为了缓解 Cube 的构建压力，减少生成的 Cuboid 数目，Apache Kylin 引入了一系列的高级设置，帮助用户筛选出真正需要的 Cuboid。这些高级设置包括聚合组（Aggregation Group）、联合维度（Joint Dimension）、层级维度（Hierachy Dimension）和必要维度（Mandatory Dimension）等




#### 问题 
随着维度数目的增加 Cuboid 的数量会爆炸式地增长，不仅占用大量的存储空间还会延长 Cube 的构建时间。

#### 解决问题
缓解 Cube 的构建压力，减少生成的 Cuboid 数目




### 聚合组（Aggregation Group）

用户根据自己关注的维度组合，可以划分出自己关注的组合大类，这些大类在 Apache Kylin 里面被称为聚合组。例如图 1 中展示的 Cube，如果用户仅仅关注维度 AB 组合和维度 CD 组合，那么该 Cube 则可以被分化成两个聚合组，分别是聚合组 AB 和聚合组 CD。如图 2 所示，生成的 Cuboid 数目从 16 个缩减成了 8 个。
![image](https://note.youdao.com/yws/api/personal/file/FA8A2EC0305E41AEAF29892F227D7996?method=download&shareKey=c32b90477fc3b524dc50070d19c3fa5c)

用户关心的聚合组之间可能包含相同的维度，例如聚合组 ABC 和聚合组 BCD 都包含维度 B 和维度 C。这些聚合组之间会衍生出相同的 Cuboid，例如聚合组 ABC 会产生 Cuboid BC，聚合组 BCD 也会产生 Cuboid BC。这些 Cuboid不会被重复生成，一份 Cuboid 为这些聚合组所共有

![image](https://note.youdao.com/yws/api/personal/file/5C7115FA940D4286B82A78016EA79595?method=download&shareKey=ad2561627a7a67e277ade58209bd8aa8)

有了聚合组用户就可以粗粒度地对 Cuboid 进行筛选，获取自己想要的维度组合。



#### 应用实例


假设创建一个交易数据的 Cube，它包含了以下一些维度：顾客 ID buyer_id 交易日期 cal_dt、付款的方式 pay_type 和买家所在的城市 city。有时候，分析师需要通过分组聚合 city 、cal_dt 和 pay_type 来获知不同消费方式在不同城市的应用情况；有时候，分析师需要通过聚合 city 、cal_dt 和 buyer_id，来查看顾客在不同城市的消费行为。在上述的实例中，推荐建立两个聚合组

![image](https://note.youdao.com/yws/api/personal/file/79BDB99846F64301A5EDCFFD22B7AA9A?method=download&shareKey=205b021e366ab172c7944eae60f3b4e8)




聚合组 1： [cal_dt, city, pay_type]

聚合组 2： [cal_dt, city, buyer_id]



在不考虑其他干扰因素的情况下，这样的聚合组将节省不必要的 3 个 Cuboid: [pay_type, buyer_id]、[city, pay_type, buyer_id] 和 [cal_dt, pay_type, buyer_id] 等，节省了存储资源和构建的执行时间。

Case 1:

SELECT cal_dt, city, pay_type, count(*) FROM table GROUP BY cal_dt, city, pay_type 则将从 Cuboid [cal_dt, city, pay_type] 中获取数据。

Case2:

SELECT cal_dt, city, buy_id, count(*) FROM table GROUP BY cal_dt, city, buyer_id 则将从 Cuboid [cal_dt, city, buyer_id] 中获取数据。

Case3 如果有一条不常用的查询:

SELECT pay_type, buyer_id, count(*) FROM table GROUP BY pay_type, buyer_id 则没有现成的完全匹配的 Cuboid。

此时，Apache Kylin 会通过在线计算的方式，从现有的 Cuboid 中计算出最终结果。(或者出现下压到hive查询的情况)




### 层级维度（Hierarchy Dimension）


用户选择的维度中常常会出现具有层级关系的维度。例如对于国家（country）、省份（province）和城市（city）这三个维度，从上而下来说国家／省份／城市之间分别是一对多的关系。也就是说，用户对于这三个维度的查询可以归类为以下三类:

1. group by country
2. group by country, province（等同于group by province）
3. group by country, province, city（等同于 group by country, city 或者group by city）

假设维度 A 代表国家，维度 B 代表省份，维度 C 代表城市，那么ABC 三个维度可以被设置为层级维度，生成的Cube 如图 


![image](https://note.youdao.com/yws/api/personal/file/8FA88E222608410988C7DB2E5005AAAB?method=download&shareKey=ea1208f55f8a5bb165705af8d5633933)

例如，Cuboid [A,C,D]=Cuboid[A, B, C, D]，Cuboid[B, D]=Cuboid[A, B, D]，因而 Cuboid[A, C, D] 和 Cuboid[B, D] 就不必重复存储。

下图展示了 Kylin 按照前文的方法将冗余的Cuboid 剪枝从而形成图 2 的 Cube 结构，Cuboid 数目从 16 减小到 8。

![image](https://note.youdao.com/yws/api/personal/file/FCA741F9F29A484EB8A935758848B8F1?method=download&shareKey=f0ee97b48879f798501fc72ee94084df)

#### 应用实例



假设一个交易数据的 Cube，它具有很多普通的维度，像是交易的城市 city，交易的省 province，交易的国家 country， 和支付类型 pay_type等。分析师可以通过按照交易城市、交易省份、交易国家和支付类型来聚合，获取不同层级的地理位置消费者的支付偏好。在上述的实例中，建议在已有的聚合组中建立一组层级维度（国家country／省province／城市city），包含的维度和组合方式如图


![image](https://note.youdao.com/yws/api/personal/file/4F9A1EAB915B4115AEC1AB747E445C8F?method=download&shareKey=53ccacf579ecbaec3827c62de921e39a)

聚合组：[country, province, city，pay_type]

层级维度： [country, province, city]

Case 1 当分析师想从城市维度获取消费偏好时：

SELECT city, pay_type, count(*) FROM table GROUP BY city, pay_type 则它将从 Cuboid [country, province, city, pay_type] 中获取数据。

Case 2 当分析师想从省级维度获取消费偏好时：

SELECT province, pay_type, count(*) FROM table GROUP BY province, pay_type则它将从Cuboid [country, province, pay_type] 中获取数据。

Case 3 当分析师想从国家维度获取消费偏好时：

SELECT country, pay_type, count(*) FROM table GROUP BY country, pay_type则它将从Cuboid [country, pay_type] 中获取数据。

Case 4 如果分析师想获取不同粒度地理维度的聚合结果时：

无一例外都可以由图 3 中的 cuboid 提供数据 。

例如，SELECT country, city, count(*) FROM table GROUP BY country, city 则它将从 Cuboid [country, province, city] 中获取数据。


### 联合维度（Joint Dimension）

用户有时并不关心维度之间各种细节的组合方式，例如用户的查询语句中仅仅会出现 group by A, B, C，而不会出现 group by A, B 或者 group by C 等等这些细化的维度组合。这一类问题就是联合维度所解决的问题。例如将维度 A、B 和 C 定义为联合维度，Apache Kylin 就仅仅会构建 Cuboid ABC，而 Cuboid AB、BC、A 等等Cuboid 都不会被生成。最终的 Cube 结果如图 2 所示，Cuboid 数目从 16 减少到 4。

![image](https://note.youdao.com/yws/api/personal/file/11848D1619A94FA6A51D07AAA25FE6C0?method=download&shareKey=cdab2f2ac35f7cb12b71a7a5b6b679b4)

#### 应用实例

假设创建一个交易数据的Cube，它具有很多普通的维度，像是交易日期 cal_dt，交易的城市 city，顾客性别 sex_id 和支付类型 pay_type 等。分析师常用的分析方法为通过按照交易时间、交易地点和顾客性别来聚合，获取不同城市男女顾客间不同的消费偏好，例如同时聚合交易日期 cal_dt、交易的城市 city 和顾客性别 sex_id来分组。在上述的实例中，推荐在已有的聚合组中建立一组联合维度，包含的维度和组合方式如图

![image](https://note.youdao.com/yws/api/personal/file/D5E2052B974B4CF9B6AC0A6B7185C4FF?method=download&shareKey=baa6ccc766dc73a6b7ebe070ebb8b27b)

聚合组：[cal_dt, city, sex_id，pay_type]

联合维度： [cal_dt, city, sex_id]

Case 1：

SELECT cal_dt, city, sex_id, count(*) FROM table GROUP BY cal_dt, city, sex_id则它将从Cuboid [cal_dt, city, sex_id]中获取数据

Case2如果有一条不常用的查询：

SELECT cal_dt, city, count(*) FROM table GROUP BY cal_dt, city 则没有现成的完全匹配的 Cuboid，Apache Kylin 会通过在线计算的方式，从现有的 Cuboid 中计算出最终结果。




### 设计规范

#### 模型设计规范

1. 按照日期增量构建，推荐与Hive表分区一致
2. 维表主键必须确保唯一
3. 高基维表禁用Snapshot存储方式(高基维:count distinct后数据很大，例如身份证号)
4. Snapshot维表，不需要历史，缓慢变化维设置为TYPE1(https://zhuanlan.zhihu.com/p/55597977)
5. Snapshot维表请勿存储历史切片数据
6. 模型支持雪花模型和星型模型，但仅有一张事实表
7. 模型表支持当前表、切片表，暂不支持拉链表
8. 对于度量分段的场景，建议通过模型可计算列支持
9. 字段设置：'D'表示维度，'M'表示度量，'—'表示禁用

##### 小Tips


1. 推荐日期维表与事实表的日期分区字段关联
2. 字段被禁用后，前端将不可查询
3. 拉链表可以借助视图转换成切片表
4. 模型启用分区后子查询不能跨分区查询  
```
date in ('2020-08-15','2020-08-14') (x)
xxxx date in ('2020-08-15')  union xxxxx date in ('2020-08-14') (√)
```
#### Cube设计规范

1. 优先设置日期为必选维度，且为hive分区字段
2. 普通维度不能超过62个
3. 同一维表不建议普通维度和衍生维度混用
4. Cuboid的数量推荐在100以内， 一般不操作300
5. RowKey顺序按照基数降序，参与过滤、使用频率高的靠前
6. 可计算列的RowKey需要根据实际情况自行调整
7. 可以根据具体指标的使用场景设置不同的聚合组
8. 基数较大使用频率较高的维度，不建议设置成衍生维
9. Cube设计尽量避免超高基维作为维度
10. 联合维度的基数乘积不宜超过1000

##### 小Tips


### SQL查询规范


1. 日期过滤字段是构建cube的日期增量字段
2. 分组、过滤字段在cube中需要设置成维度
3. 度量的聚合方式需要与cube中设置一致
4. 统一系数折算，建议加在聚合后
5. 非统一系数折算，续在模型中设置可计算列
6. 查询表名前面带database名称
7. 表关联方式需要与模型设计中的一致
8. 涉及度量汇总前的过滤，需要对度量维度化
9. 基础查询必须包含group关键词
10. 非业务场景必须的排序，请勿查询排序
11. 同一事实表的多次查询，能合并的推荐合并


暂不支持同一cube对同一超高基维同时配置count distinct和Top N计算，建议通过CC列派生新列







## 小tip

### 事实表需要带上group

```
select sum(value) , date from (
(select * from 事实表  where id=1 )left  join ( select * from 维表 ) on 事实表.date = 维表.date )
group by date
```
上面的sql出来的 sum(value) 会出现null的情况，无法命中cube，因为事实表没有带group，kylin认为是查询的明细表, 在kylin中，事实表要带group查询.

维表不受该限制.

正确sql
```
select sum(value) , date from (
(select date,sum(value) as value  from 事实表  where id=1  group by date )left  join ( select * from 维表 ) on 事实表.date = 维表.date )
group by date

```
### 过滤条件不要放在子查询

Kylin遵循的是“Scatter and gather”模式，而有的时候在【第二阶段SQL查询】时无法实现Filter Pushdown和Limit Pushdown等优化手段，需要等待数据集中返回Kylin后再筛选数据，这样数据吞吐量会很大，影响查询性能。优化方法是重写SQL语句。

例如，该SQL查询的筛选条件（斜体加粗部分）放在子查询中，因此无法实现Filter Pushdown。

```
select KYLIN_SALES.PART_DT, sum(KYLIN_SALES.PRICE) from KYLIN_SALESinner join (select ACCOUNT_ID, ACCOUNT_BUYER_LEVEL from KYLIN_ACCOUNT where ACCOUNT_COUNTRY = 'US' ) as TTon KYLIN_SALES.BUYER_ID = TT.ACCOUNT_IDgroup by KYLIN_SALES.PART_DT
```
正确的写法应该是：
```
select KYLIN_SALES.PART_DT, sum(KYLIN_SALES.PRICE)from KYLIN_SALESinner join KYLIN_ACCOUNT as TT on KYLIN_SALES.BUYER_ID = TT.ACCOUNT_ID where TT.ACCOUNT_COUNTRY = 'US'group by KYLIN_SALES.PART_DT
```




### Referenece

[Apache Kylin查询性能优化](https://www.jianshu.com/p/4f11eb995caa)

[【技术帖】Apache Kylin 高级设置：聚合组（Aggregation Group）原理解析](https://blog.51cto.com/xiaolanlan/2068981)


[【技术帖】Apache Kylin 高级设置：层级维度（Hierarchy Dimension）原理](https://blog.51cto.com/xiaolanlan/2068975)

[【技术帖】Apache Kylin 高级设置：联合维度（Joint Dimension）原理解析](https://blog.51cto.com/xiaolanlan/2068969)


[cube 优化](https://blog.51cto.com/xiaolanlan/2068952)

[缓慢变化维相关](https://zhuanlan.zhihu.com/p/55597977)

https://blog.csdn.net/yu616568/article/details/51974526


