---
title: java 逃逸分析
date: 2019-10-16 21:13:53
tags:
categories:
	- Java
---


## Java中的对象都是在堆中分配吗？

### 什么是逃逸分析？

关于 Java 逃逸分析的定义：

逃逸分析（Escape Analysis）简单来讲就是，Java Hotspot 虚拟机可以分析新创建对象的使用范围，并决定是否在 Java 堆上分配内存的一项技术。

逃逸分析的 JVM 参数如下：

* 启逃逸分析：-XX:+DoEscapeAnalysis
* 关闭逃逸分析：-XX:-DoEscapeAnalysis
* 显示分析结果：-XX:+PrintEscapeAnalysis
* 逃逸分析技术在 Java SE 6u23+ 

开始支持，并默认设置为启用状态，可以不用额外加这个参数。


### 对象逃逸状态

#### 1、全局逃逸（GlobalEscape）

即一个对象的作用范围逃出了当前方法或者当前线程，有以下几种场景：

* 对象是一个静态变量
* 对象是一个已经发生逃逸的对象
* 对象作为当前方法的返回值
```java
public class EscapeAnalysis {

     public static Object object;
     
     public void globalVariableEscape(){//全局变量赋值逃逸  
         object =new Object();  
      }  
     
     public Object methodEscape(){  //方法返回值逃逸
         return new Object();
     }
     
     public void instancePassEscape(){ //实例引用发生逃逸
        this.speak(this);
     }
     
     public void speak(EscapeAnalysis escapeAnalysis){
         System.out.println("Escape Hello");
     }
}
```

#### 2、参数逃逸（ArgEscape）

即一个对象被作为方法参数传递或者被参数引用，但在调用过程中不会发生全局逃逸，这个状态是通过被调方法的字节码确定的。

#### 3、没有逃逸

即方法中的对象没有发生逃逸。

### 逃逸分析优化

针对上面第三点，当一个对象没有逃逸时，可以得到以下几个虚拟机的优化。

#### 1) 锁消除

我们知道线程同步锁是非常牺牲性能的，当编译器确定当前对象只有当前线程使用，那么就会移除该对象的同步锁。

例如，StringBuffer 和 Vector 都是用 synchronized 修饰线程安全的，但大部分情况下，它们都只是在当前线程中用到，这样编译器就会优化移除掉这些锁操作。

锁消除的 JVM 参数如下：
* 开启锁消除：-XX:+EliminateLocks
* 关闭锁消除：-XX:-EliminateLocks

锁消除在 JDK8 中都是默认开启的，并且锁消除都要建立在逃逸分析的基础上。

#### 2) 标量替换

首先要明白标量和聚合量，基础类型和对象的引用可以理解为标量，它们不能被进一步分解。而能被进一步分解的量就是聚合量，比如：对象。

对象是聚合量，它又可以被进一步分解成标量，将其成员变量分解为分散的变量，这就叫做标量替换。

这样，如果一个对象没有发生逃逸，那压根就不用创建它，只会在栈或者寄存器上创建它用到的成员标量，节省了内存空间，也提升了应用程序性能。

标量替换的 JVM 参数如下：

* 开启标量替换：-XX:+EliminateAllocations
* 关闭标量替换：-XX:-EliminateAllocations
* 显示标量替换详情：-XX:+PrintEliminateAllocations
标量替换同样在 JDK8 中都是默认开启的，并且都要建立在逃逸分析的基础上。

通过逃逸分析确定该对象不会被外部访问，并且对象可以被进一步分解时，JVM不会创建该对象，而会将该对象成员变量分解若干个被这个方法使用的成员变量所代替。这些代替的成员变量在栈帧或寄存器上分配空间。

####  3) 栈上分配

当对象没有发生逃逸时，该对象就可以通过标量替换分解成成员标量分配在栈内存中，和方法的生命周期一致，随着栈帧出栈时销毁，减少了 GC 压力，提高了应用程序性能。

## 总结

逃逸分析讲完了，总结了不少时间，我们也应该大概知道逃逸分析是为了优化 JVM 内存和提升程序性能的。

我们知道这点后，在平时开发过程中就要可尽可能的控制变量的作用范围了，变量范围越小越好，让虚拟机尽可能有优化的空间。

简单举一个例子吧，如：

```
return stringBuilder ;
```

可以改为：
```
return stringBuilder .toString();
```

### Reference
https://www.cnblogs.com/javastack/p/11023044.html
https://www.jianshu.com/p/580f17760f6e