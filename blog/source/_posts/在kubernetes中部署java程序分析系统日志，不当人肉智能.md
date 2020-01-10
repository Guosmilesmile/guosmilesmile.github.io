---
title: 在kubernetes中部署java程序分析系统日志，不当人肉智能
date: 2020-01-10 20:45:21
tags:
categories:
	- Kubernetes
---


### 背景


是否遇到过服务器cpu温度高导致降频，磁盘只读等服务器问题。如果你遇到的问题不多或者你只用云服务，那恭喜你，可以跳过这篇文章。如果你被各种辣鸡服务器困扰，还被半死不活的服务器拖后腿导致集群性能短板，那么摆脱人肉智能，需要针对系统日志进行分析。


### 怎么判断服务器是否有问题

简单点的通过观察系统日志来判断是否有问题
```
/var/log/messages
```

该日志可以反馈很多问题。


例如

* Temperature above threshold, cpu clock throttled（cpu温度高导致的降频）
* I/O error，dev sdh (磁盘有问题)
* memory 有问题等等


### 采集文件

针对系统日志的读取和采集，需要针对一个不断生成数据的文件不停的采集，市面上现在有很多开源的组件可以完成，例如filebeat、logstash、flume等等。如果要自己手动开发呢，需要怎么办呢。

java版本的可以采用apace的common io来完成。

他采用的是线程方式来监控文件内容的变化

1、Tailer类（采用线程的方式进行文件的内容变法）

2、TailerListener类

3、TailerListenerAdapter类，该类是集成了TailerListener的实现空的接口方式


实例代码如下

```java
import org.apache.commons.io.FileUtils;  
import org.apache.commons.io.IOUtils;  
import org.apache.commons.io.input.Tailer;  
import org.apache.commons.io.input.TailerListenerAdapter;  
  
import java.io.File;  
  
public class TailerTest {  
  
    public static void main(String []args) throws Exception{  
        TailerTest tailerTest = new TailerTest();  
        tailerTest.test();  
        boolean flag = true;  
        File file = new File("C:/Users/hadoop/Desktop/test/1.txt");  
  
        while(flag){  
            Thread.sleep(1000);  
            FileUtils.write(file,""+System.currentTimeMillis()+ IOUtils.LINE_SEPARATOR,true);  
        }  
  
    }  
  
    public void test() throws Exception{  
        File file = new File("C:/Users/hadoop/Desktop/test/1.txt");  
        FileUtils.touch(file);  
  
        Tailer tailer = new Tailer(file,new TailerListenerAdapter(){  
  
            @Override  
            public void fileNotFound() {  //文件没有找到  
                System.out.println("文件没有找到");  
                super.fileNotFound();  
            }  
  
            @Override  
            public void fileRotated() {  //文件被外部的输入流改变  
                System.out.println("文件rotated");  
                super.fileRotated();  
            }  
  
            @Override  
            public void handle(String line) { //增加的文件的内容  
                System.out.println("文件line:"+line);  
                super.handle(line);  
            }  
  
            @Override  
            public void handle(Exception ex) {  
                ex.printStackTrace();  
                super.handle(ex);  
            }  
  
        },4000,true);  
        new Thread(tailer).start();  
    }  
}  
```


文件的路径最好是通过参数传递进去。这样可以做到可变。


### 如何部署以及该程序的可用性


1. 需要的是每个服务器都部署一个而且只部署一个。
2. 一个监控服务器的程序，势必需要另一个程序去监控这个程序，为了形成闭环，那么最近需要通过环境去监控上一个程序。


因此采用的是在kubernetes上部署，通过DaemonSet的形式，做到第一点。然后基于kubernetes可以做到程序挂了自动拉起的高可用形式，解决第二点。


### 容器化引来的新问题

1. 如何获取容器外，宿主机的文件
2. 如果获取宿主机的ip


针对第一点，通过将宿主机的文件挂载到容器内部即可,如下配置可以将宿主机的文件映射到在容器内的/log/messages

```
volumeMounts:
           - mountPath: /log/messages
             subPath: messages
             name: logmessage
       volumes:
       - hostPath:
           path: /var/log/
         name: logmessage


```

如何将文件的路径传递进去呢，采用command指令

```
 command: ["java"]
 args: ["-cp","fileCollect-1.0-SNAPSHOT.jar","study.TailLog","/log/messages"]

```

第二点，目前没有办法拿到宿主机的ip（有谁知道的求告知），可以通过获取容器的名称然后去找对应服务器。如何在java程序中获取呢。这里要用到kubernetes的环境变量。


每个容器都自带了这个一个系统变量HOSTNAME。java中获取方式如下

```java
 Map<String, String> map = System.getenv();
        String hostName = map.get("HOSTNAME");
        System.out.println(hostName);  
```




### Reference


https://blog.csdn.net/nickDaDa/article/details/89357667

https://kubernetes.io/zh/docs/tasks/inject-data-application/define-environment-variable-container/
         

https://www.cnblogs.com/hd-zg/p/5930636.html