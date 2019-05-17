---
title: jstack排查cpu高
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