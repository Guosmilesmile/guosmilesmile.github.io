---
title: clickhouse global用法
date: 2019-03-31 23:55:57
tags:
categories: clickhouse
---

### global介绍
global 有两种用法，GLOBAL in /GLOBAL join。

### 分布式查询
先介绍一下分布式查询
```sql
SELECT uniq(UserID) FROM distributed_table
```
将会被发送到所有远程服务器
```sql
SELECT uniq(UserID) FROM local_table
```
然后并行运行，直到达到中间结果可以结合的阶段。然后，中间结果将被返回给请求者服务器并在其上合并，最终的结果将被发送到客户端。

### in/join 的问题

当使用in的时候，查询被发送到远程服务器，并且每个服务器都在IN或JOIN子句中运行子查询
```sql
SELECT uniq(UserID) FROM distributed_table WHERE CounterID = 101500 AND UserID IN (SELECT UserID FROM distributed_table WHERE CounterID = 34)
```
这个查询将发送到所有服务器,子查询的分布式表名会替换成本地表名
```sql
SELECT uniq(UserID) FROM local_table WHERE CounterID = 101500 AND UserID IN (SELECT UserID FROM distributed_table WHERE CounterID = 34)
```
子查询将开始在每个远程服务器上运行。由于子查询使用分布式表，所以每个远程服务器上的子查询将会对每个远程服务器都感到不满，如果您有一个100个服务器集群，执行整个查询将需要10000个基本请求，这通常被认为是不可接受的。

### 使用GLOBAL in /GLOBAL join 
```
SELECT uniq(UserID) FROM distributed_table WHERE CounterID = 101500 AND UserID GLOBAL IN (SELECT UserID FROM distributed_table WHERE CounterID = 34)
```

服务器将运行子查询
```sql
SELECT UserID FROM distributed_table WHERE CounterID = 34
```
结果将被放在RAM中的临时表中。然后请求将被发送到每个远程服务器
```sql
SELECT uniq(UserID) FROM local_table WHERE CounterID = 101500 AND UserID GLOBAL IN _data1
```
临时表“data1”将连同查询一起被发送到每个远程服务器（临时表的名称是实现定义的）。


### 使用注意


* 创建临时表时，数据不是唯一的，为了减少通过网络传输的数据量，请在子查询中使用DISTINCT（你不需要在普通的IN中这么做）
* 临时表将发送到所有远程服务器。其中传输不考虑网络的拓扑结构。例如，如果你有10个远程服务器存在与请求服务器非常远的数据中心中，则数据将通过通道发送数据到远程数据中心10次。使用GLOBAL IN时应避免大数据集。
* 当使用global…JOIN，首先会在请求者服务器运行一个子查询来计右表(right table)。将此临时表传递给每个远程服务器，并使用传输的临时数据在其上运行查询。会出现网络传输，因此尽量将小表放在右表。