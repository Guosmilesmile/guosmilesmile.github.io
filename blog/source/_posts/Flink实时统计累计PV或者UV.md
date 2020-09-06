---
title: Flink实时统计累计PV或者UV
date: 2020-09-06 16:23:03
tags:
categories:
	- Flink
---


### 背景
经常会遇到这样的需求，统计1小时的PV或者UV，但是想要每分钟都可以看到当前的数据，对接到实时大屏上，可以动态看到数据变化过程。


### Flink DataStream

需要用到ContinuousProcessTimeTrigger或者ContinuousEventTimeTrigger。


![image](https://note.youdao.com/yws/api/personal/file/A28B1475267545F18C441E11F4EF4B6A?method=download&shareKey=8465642b1068d54b7ede98c423354343)

使用示例：

![image](https://note.youdao.com/yws/api/personal/file/CD86D9C320DD4A95AFE80929713FE1DA?method=download&shareKey=238c4bd9753d812e012b67b58da382ac)


假如我们定义一个5分钟的基于 EventTime 的滚动窗口，定义一个每2分触发计算的 Trigger，有4条数据事件时间分别是20:01、20:02、20:03、20:04，对应的值分别是1、2、3、2，我们要对值做 Sum 操作。

初始时，State 和 Result 中的值都为0。

![image](https://note.youdao.com/yws/api/personal/file/BD2418989D0249C4B937DAA035E9E602?method=download&shareKey=91ff3a6a761529161d044211d146a47d)

当第一条数据在20:01进入窗口时，State 的值为1，此时还没有到达 Trigger 的触发时间。



![image](https://note.youdao.com/yws/api/personal/file/21475CAFD97B434CBEB375378FD8A01E?method=download&shareKey=231972a2cf924dcbc8e58291264f6543)

第二条数据在20:02进入窗口，State 中的值为1+2=3，此时达到2分钟满足 Trigger 的触发条件，所以 Result 输出结果为3。

![image](https://note.youdao.com/yws/api/personal/file/314C55B4476B441999E6C7F29A3D7D46?method=download&shareKey=3b6f12b19778a38d2a2e5feba4e5a4b5)


第三条数据在20:03进入窗口，State 中的值为3+3 = 6，此时未达到 Trigger 触发条件，没有结果输出。

![image](https://note.youdao.com/yws/api/personal/file/BD67B242B1B0448DB18102AF5F21F0B1?method=download&shareKey=a1464b05e32df59d24edb0b966a63276)

第四条数据在20:04进入窗口，State中的值更新为6+2=8，此时又到了2分钟达到了 Trigger 触发时间，所以输出结果为8。如果我们把结果输出到支持 update 的存储，比如 MySQL，那么结果值就由之前的3更新成了8。

#### 问题：如果 Result 只能 append？

![image](https://note.youdao.com/yws/api/personal/file/A49F0BF06FA74F3795740A38A1885BBF?method=download&shareKey=6a4c39f1581f186ed043a64075728194)

如果 Result 不支持 update 操作，只能 append 的话，则会输出2条记录，在此基础上再做计算处理就会引起错误。

这样就需要 PurgingTrigger 来处理上面的问题。

#### PurgingTrigger 的应用


![image](https://note.youdao.com/yws/api/personal/file/A2170B8E30DC43A69E76BADA12A3AE7C?method=download&shareKey=c721727a31db28a24029a0350c00ff3e)
和上面的示例一样，唯一的不同是在 ContinuousEventTimeTrigger 外面包装了一个 PurgingTrigger，其作用是在 ContinuousEventTimeTrigger 触发窗口计算之后将窗口的 State 中的数据清除。


再看下流程：
![image](https://note.youdao.com/yws/api/personal/file/813F8EA86A49411BA84D735693ED0C10?method=download&shareKey=5b764c265f7381ef13652003b83a0502)

前两条数据先后于20:01和20:02进入窗口，此时 State 中的值更新为3，同时到了Trigger的触发时间，输出结果为3。
![image](https://note.youdao.com/yws/api/personal/file/FB9455EF298B41AE8FAB5679E82E08E6?method=download&shareKey=879b12065b6867d6ce6fd8fe51732ba1)

由于 PurgingTrigger 的作用，State 中的数据会被清除。



### Flink SQL

每分钟统计当日0时到当前的累计UV数.


可以有两种方式，
1. 可以直接用group by + mini batch
2. window聚合 + fast emit

#### 方法一

group by的字段里面可以有一个日期的字段，例如你上面提到的DATE_FORMAT(rowtm, 'yyyy-MM-dd')。
这种情况下的状态清理，需要配置state retention时间，配置方法可以参考[1] 。同时，mini batch的开启也需要
用参数[2] 来打开。



在 Flink 1.11 中，你可以尝试这样：
```
CREATE TABLE mysql (
   time_str STRING,
   uv BIGINT,
   PRIMARY KEY (ts) NOT ENFORCED
) WITH (
   'connector' = 'jdbc',
   'url' = 'jdbc:mysql://localhost:3306/mydatabase',
   'table-name' = 'myuv'
);

INSERT INTO mysql
SELECT MAX(DATE_FORMAT(ts, 'yyyy-MM-dd HH:mm:00')), COUNT(DISTINCT  user_id)
FROM user_behavior;

```

#### 方法二

这种直接开一个天级别的tumble窗口就行。然后状态清理不用特殊配置，默认就可以清理。
fast emit这个配置现在还是一个experimental的feature

```
table.exec.emit.early-fire.enabled = true
table.exec.emit.early-fire.delay = 60 s
```


在 Flink 1.11 中，你可以尝试这样：
```
CREATE TABLE mysql (
   time_str STRING,
   uv BIGINT,
   PRIMARY KEY (ts) NOT ENFORCED
) WITH (
   'connector' = 'jdbc',
   'url' = 'jdbc:mysql://localhost:3306/mydatabase',
   'table-name' = 'myuv'
);

INSERT INTO mysql
SELECT DATE_FORMAT(TUMBLE_START(t, INTERVAL '1' MINUTE) , 'yyyy-MM-dd HH:mm:00'), COUNT(DISTINCT  user_id)
FROM user_behavior group by  TUMBLE(ts, INTERVAL '1' MINUTE);

```


### Reference

[数仓系列 | Flink窗口的应用与实现](https://mp.weixin.qq.com/s/xZTGeFaaVW4VDDVgp3jwqg)


[FLINKSQL1.10实时统计累计UV](http://apache-flink.147419.n8.nabble.com/FLINKSQL1-10-UV-td4003.html)