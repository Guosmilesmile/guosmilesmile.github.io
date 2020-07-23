---
title: 如何使用jemalloc分析flink使用的native memory
date: 2020-07-07 21:47:53
tags:categories:
	- Flink
	- Kubernetes
---




## 前言

最近笔者因为flink集群运行在kubernetes上，由于不可抗力导致pod重生，job需要restart，在没有开启checkpoint的情况下，作业只要重启就会频繁被os kill，这明显是堆外内存超用的现象。

heap memory和direct memory被jvm控制了，显然不会被os kill，而是OOM，可以被flink 捕捉而爆出异常的，被os kill只有托管给rocksdb的native memory了。


如何分析native memory的leak呢，就需要引入jemalloc。


## 什么是jemalloc

系统的物理内存是有限的，而对内存的需求是变化的, 程序的动态性越强，内存管理就越重要，选择合适的内存管理算法会带来明显的性能提升。
比如nginx， 它在每个连接accept后会malloc一块内存，作为整个连接生命周期内的内存池。 当HTTP请求到达的时候，又会malloc一块当前请求阶段的内存池, 因此对malloc的分配速度有一定的依赖关系。


内存管理可以分为三个层次，自底向上分别是：

* 操作系统内核的内存管理
* glibc层使用系统调用维护的内存管理算法
* 应用程序从glibc动态分配内存后，根据应用程序本身的程序特性进行优化， 比如使用引用计数std::shared_ptr，apache的内存池方式等等。
当然应用程序也可以直接使用系统调用从内核分配内存，自己根据程序特性来维护内存，但是会大大增加开发成本


glibc malloc的实现是ptmalloc2，其替代品tcmalloc 和 jemalloc。

#### tcmalloc
tcmalloc是Google开源的一个内存管理库， 作为glibc malloc的替代品。目前已经在chrome、safari等知名软件中运用。
根据官方测试报告，ptmalloc在一台2.8GHz的P4机器上（对于小对象）执行一次malloc及free大约需要300纳秒。而TCMalloc的版本同样的操作大约只需要50纳秒。

#### jemalloc
jemalloc是facebook推出的， 最早的时候是freebsd的libc malloc实现。 目前在firefox、facebook服务器各种组件中大量使用。

对应的git地址如下

https://github.com/jemalloc/jemalloc/blob/dev/INSTALL.md


jemalloc有一项功能，对应长时间运行的程序可以trace内存,见文档[6]。



如果想更加详细的了解这三者的性能和对比，可以参考文档[5]


### 环境

笔者的flink集群版本是1.10.1，运行在1.17的kubernetes上。


### 编译jemalloc

下载
```shell
wget https://github.com/jemalloc/jemalloc/releases/download/5.2.1/jemalloc-5.2.1.tar.bz2
```

解压

```shell

tar -zxvf  jemalloc-5.2.1.tar.bz2

```

开始编译

```
./configure --enable-prof --enable-stats --enable-debug --enable-fill
```
一定要加上--enable-prof 才可以使用heap-prof的功能


```
make 
make install
```

对我们来说，需要的是

```
bin
lib
```

### 将文件打入flink镜像中
```
bin
lib
```

打入flink镜像
```
ADD jemalloc /opt/jemalloc/
```

不知道怎么自己构建镜像的可以参考

https://guosmilesmile.github.io/2020/05/27/Flink-on-native-kubernetes-%E4%BD%BF%E7%94%A8%E5%92%8C%E4%BF%AE%E6%94%B9/



### 配置jemalloc


如果flink的运行方式的native kubernetes，可以在构建集群的脚本添加

```
-Dcontainerized.taskmanager.env.LD_PRELOAD: /opt/jemalloc/lib/libjemalloc.so.2
-Dcontainerized.taskmanager.env.MALLOC_CONF: prof:true,lg_prof_interval:25,lg_prof_sample:17,prof_prefix:/opt/state/jeprof.out
```


如果是on kubernetes的standalone，那么需要修改deployment

```
env:
    - name: LD_PRELOAD
        value: /opt/jemalloc/lib/libjemalloc.so.2
    - name: MALLOC_CONF
        value: prof:true,lg_prof_interval:30,lg_prof_sample:17,prof_prefix:/opt/state/tmp/jeprof.out

```


配置解释：

LD_PRELOAD： 将内存分配从ptmalloc2改为libjemalloc.so.2

MALLOC_CONF： jemalloc的配置，prof_prefix是将生成的内存文件dump到指定文件。lg_prof_interval:30 是 2^30 byte（1G）生成一个文件，具体参数可以参考

https://github.com/jemalloc/jemalloc/blob/dev/INSTALL.md


### 进入容器补充工具

可以到/opt/state/tmp/下看到很多jeprof.out开头的heap文件

![image](https://note.youdao.com/yws/api/personal/file/CA2C26BB853C41108317C971100565C9?method=download&shareKey=3b645d4477190f7c98d6731da06ff6c2)

由于flink的容器是最简化模式，会缺少很多工具，想要直接使用jeprof是会缺少很多的，需要补充下载

先将源改为国内的源

在容器内运行

```
mv /etc/apt/sources.list /etc/apt/sources.list.bak
echo "deb http://mirrors.163.com/debian/ jessie main non-free contrib" >> /etc/apt/sources.list
echo "deb http://mirrors.163.com/debian/ jessie-proposed-updates main non-free contrib" >>/etc/apt/sources.list
echo "deb-src http://mirrors.163.com/debian/ jessie main non-free contrib" >>/etc/apt/sources.list
echo "deb-src http://mirrors.163.com/debian/ jessie-proposed-updates main non-free contrib" >>/etc/apt/sources.list

apt-get update 


```

安装对应的工具
```
apt-get install -y binutils graphviz ghostscript 
```


### 分析内存

```
/opt/jemalloc/bin/jeprof --show_bytes `which java` /opt/state/tmp/jeprof.out.301.808.i808.heap
```

```

root@flink-taskmanager-69df85b5b9-dq42m:/opt/state/tmp# /opt/jemalloc/bin/jeprof --show_bytes `which java` /opt/state/tmp/jeprof.out.301.808.i808.heap 
Using local file /usr/local/openjdk-8/bin/java.
Argument "MSWin32" isn't numeric in numeric eq (==) at /opt/jemalloc/bin/jeprof line 5124.
Argument "linux" isn't numeric in numeric eq (==) at /opt/jemalloc/bin/jeprof line 5124.
Using local file /opt/state/tmp/jeprof.out.301.808.i808.heap.
Welcome to jeprof!  For help, type 'help'.
(jeprof) top
Total: 5580982945 B
2350833429  42.1%  42.1% 2350833429  42.1% os::malloc@8b2970
2002406207  35.9%  78.0% 2002406207  35.9% rocksdb::UncompressBlockContentsForCompressionType
1182793728  21.2%  99.2% 1183056000  21.2% rocksdb::Arena::AllocateNewBlock
11014112   0.2%  99.4% 13300172   0.2% rocksdb::LRUCacheShard::Insert
 9440064   0.2%  99.6% 2011846271  36.0% rocksdb::BlockBasedTable::PartitionedIndexIteratorState::NewSecondaryIterator
 6151347   0.1%  99.7%  6151347   0.1% std::string::_Rep::_S_create
 3335701   0.1%  99.7%  3335701   0.1% readCEN
 2621559   0.0%  99.8%  2621559   0.0% rocksdb::WritableFileWriter::Append
 2381515   0.0%  99.8%  3581933   0.1% rocksdb::VersionSet::ProcessManifestWrites
 2286059   0.0%  99.9%  2286059   0.0% rocksdb::LRUHandleTable::Resize
(jeprof) 

```

可以导出成pdf或者svg

```
Output type:
   --text              Generate text report
   --callgrind         Generate callgrind format to stdout
   --gv                Generate Postscript and display
   --evince            Generate PDF and display
   --web               Generate SVG and display
   --list=<regexp>     Generate source listing of matching routines
   --disasm=<regexp>   Generate disassembly of matching routines
   --symbols           Print demangled symbol names found at given addresses
   --dot               Generate DOT file to stdout
   --ps                Generate Postcript to stdout
   --pdf               Generate PDF to stdout
   --svg               Generate SVG to stdout
   --gif               Generate GIF to stdout
   --raw               Generate symbolized jeprof data (useful with remote fetch)

```

```
 /opt/jemalloc/bin/jeprof --show_bytes -svg `which java` /opt/state/tmp/jeprof.out.301.1009.i1009.heap  > 105.svg

```

![image](https://note.youdao.com/yws/api/personal/file/113E796812C746BBBF41A9A43D7C3760?method=download&shareKey=dea4a20686cc3f44d7fda64aaf876b38)




### Refernce

[1][Flink任务物理内存溢出问题定位](www.jianshu.com/p/f18f0494e8ab)

[2][jemalloc初体验](https://note.abeffect.com/articles/2019/07/26/1564106051567.html)

[3][Using jemalloc to get to the bottom of a memory leak](https://technology.blog.gov.uk/2015/12/11/using-jemalloc-to-get-to-the-bottom-of-a-memory-leak/)

[4][Debugging Java Native Memory Leaks](https://www.evanjones.ca/java-native-leak-bug.html)

[5][内存优化总结:ptmalloc、tcmalloc和jemalloc](http://www.cnhalo.net/2016/06/13/memory-optimize/ )

[6][Use Case: Heap Profiling](https://github.com/jemalloc/jemalloc/wiki/Use-Case%3A-Heap-Profiling)
