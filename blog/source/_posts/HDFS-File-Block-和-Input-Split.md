---
title: HDFS File Block 和 Input Split
date: 2019-06-11 23:55:29
tags:
categories: 
	- hdfs


---


Blocks 是物理分区而 input splits 是逻辑分区，一个input splits可以对应多个物理block。当提交一个hadoop任务的时候，会将输入数据逻辑切分交给每个Mapper task. Mapper的数量与切分的数据是一致的. 很重要的一点，InputSplit 不储存真实的数据，而是数据的引用（存储的位置）。一个split包含下面两个基础信息：字节长度和存储位置。

Block size 和 split size 是可以自定义的，默认的block size是64M，默认的split size等价于block size。
```
1 data set = 1….n files = 1….n blocks for each file

1 mapper = 1 input split = 1….n blocks
```
客户端的**InputFormat.getSplits()** 负责生成input splits，每一个split会作为每个mapper的输入。默认情况，这个类为每一个block生成一个split。

假设存在一份300M大小的文件，会分布在3个block上（(block size 为128Mb）。假设我能够为每个block得到一个InputSplit。

**文件由6行字符串组成，每行50Mb**

![image](https://note.youdao.com/yws/api/personal/file/044A9AE2BFAD48EF9E97FA63A11B98C2?method=download&shareKey=ea2843f03e7fb581291d1b6fba5c7359)


### 读取流程
* 第一个读取器将从B1块(位置为0)开始读取字节，前两个EOL（换行符）将分别在50Mb和100Mb处满足，2行字符串(L1和L2)将被读取并以键/值对的形式发给发送给Mapper 1实例。从字节100Mb开始，在找到第三个EOL之前，我们将到达block的末尾(128Mb)，这个未完整的行将通过读取块B2中的字节来完成，读取直到位置为150Mb的地方。所以L3的读取方式为一部分将从块B1本地读取，第二部分将从块B2远程读取，再以键值对的形式发送给mapper1实例
* 第二个读取器从B2块开始，位置为128Mb。因为128M不是文件的开始，因此我们的指针很有可能位于一个已经被上一个读取器处理过的现有记录中的某个位置。因此需要通过跳到下一个可得的EOL来跳过这条被处理过的数据，发现下一个EOL在150M。实际上RecordReader 2 从150M的位置开始读取，而不是128M。

其余流程遵循相同的逻辑，所有内容总结如下表所示

![image](https://note.youdao.com/yws/api/personal/file/209CC747218B44ADAB0F80F3FE23D8B3?method=download&shareKey=124321a8d43d06d1ef41df295e348055)


### 如果EOL在block的第一行？
如果假设一个block是以EOL开始，通过寻找下一个EOL跳过被处理的记录，可能会丢失一条记录。所以在寻找下一个EOL前，需要将初始化的start值，改为start-1，从start-1开始寻找下一个EOL，确保没有记录被跳过。




### Reference
https://hadoopabcd.wordpress.com/2015/03/10/hdfs-file-block-and-input-split/
https://data-flair.training/blogs/mapreduce-inputsplit-vs-block-hadoop/