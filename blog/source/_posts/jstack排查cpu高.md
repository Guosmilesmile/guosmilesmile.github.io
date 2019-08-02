---
title: 线上排查cpu高
date: 2019-05-03 13:08:07
tags:
categories: Java
---
### 查消耗cpu最高的进程PID
1. top -c
2. 按P，进程按照cpu使用率排序

### 根据PID查出消耗cpu最高的线程号
1. top -Hp 进程号
2. 按P，进程按照cpu使用率排序
3. 将最高的十进制线程号转为十六进制

### 根据线程号查出对应的java线程，进行处理

1. jstack -l pid > pid.test
2. cat pid.test | grep '十六进制线程号' -C 8 或者直接less进去看

![TIM截图20190503131026](https://note.youdao.com/yws/api/personal/file/3F1C24F735294029BD27EDFB5A692B1E?method=download&shareKey=106151c7135270ee93724431076b5061)

可以看出，如果想要提高性能，可以从SimpleDateFormat性能上入手，换成DateTimeFormatter。


### 使用arthas和jstat与jmap

https://alibaba.github.io/arthas/quick-start.html


#### 案例1
![image](https://note.youdao.com/yws/api/personal/file/53FDF91F1F944F75A406928233222AD8?method=download&shareKey=722f3469863901e461d392a8080a4871)


![image](https://note.youdao.com/yws/api/personal/file/5522D2F9D50C447391A5D2E10931246A?method=download&shareKey=2c0f21f74d9c4e533486674a633f7a82)


从上面可以看到大部分都在内存几乎都满了，通过jstat或者arthas都可以看到。通过jmap看到大部分的内存集中在字符，应该是序列化和逆序列化堵了，而且线程结果可以看到，多次采样，在SimpleDateFormat上性能消耗很多。

将SimpleDateFormat换成FastDateFormat.本地测试，后者性能比前者高一倍。

其次，调整jstorm内部的资源分配，不是一味的增加资源就能解决问题，资源分配对内存的分配至关重要。


### 案例2

![image](https://note.youdao.com/yws/api/personal/file/00F5CD3D90CB4AE8A029C85BA91844F6?method=download&shareKey=1915d75cdbd197db14dfa943ab8423d4)

很明显可以看到，这个Channel类，在内存中不下百万个实例，频繁的gc会导致程序性能不行。




### 内存dump

导出整个JVM 中内存信息
```
jmap -dump:format=b,file=文件名 [pid]
```

通过mat分析堆栈信息。