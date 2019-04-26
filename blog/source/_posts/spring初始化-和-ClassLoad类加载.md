---
title: spring初始化 和 ClassLoad类加载
date: 2019-04-20 21:25:58
tags:
categories: Java
---

```
public class Test2 implements InitializingBean {

    @Autowired
    private Test1 test1;
    private int i;

    public Test2() {
        System.out.println("test2");
    }

    public void setI(int i) {
        System.out.println(1);
        this.i = i;
    }

    @PostConstruct
    public void test() {
        System.out.println("postconstruct");
    }

    public void init() {
        System.out.println("init");
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("after");
    }
}



@Component
@Lazy
public class Test1 {

    public Test1() {
        System.out.println("test1");
    }
}


<bean class="com.cnc.qoss.singlelock.Test2" init-method="init">
    <property name="i" value="1"/>
</bean>

main函数

Test2 test2 = spirngContext.getBean(Test2.class);
```

### 初始化结果

```
test2
test1
1
postconstruct
after
init
```

初始化顺序
1. 构造函数
2. 成员变量构造
3. xml中的入参摄入
4. 构造函数后PostConstruct
5. afterPropertiesSet
6. xml中的初始化调用函数


### 普通类加载

``` java
public class ClassLoadTest {     
     private static ClassLoadTest  test = new ClassLoadTest(); 
     static int x;
     static int y = 0;
     public ClassLoadTest(){ x++; y++;}
     public static void main(String[]args){
         System.out.println(test.x);
         System.out.println(test.y);
     }
    
}
 ```
 
![image](https://note.youdao.com/yws/api/personal/file/9C9E728C0A37436F8B281A888FA7DFBE?method=download&shareKey=cb21ecdc9b016191bcff479d2e580c7e)


#### 准备过程
这个过程相当于给类变量分配内存并设置变量初始值的阶段，这些变量所使用的内存都将在方法区中进行分配。

针对上述例子：
```
test=null;
x=0;
y=0;
```
**注意**：这里有个特殊情况，如果该字段被final修饰，那么在准备阶段改字段就会被设置成咱们自定义的值。public static final int value = 11,在准备阶段就会直接赋值11，并不是该变量的初始值。

#### 解析过程
将符号引用转换成直接引用的过程。这里有两个名词符号引用和直接引用。
```
* 符号引用：符号引用与虚拟机的布局无关，甚至引用的目标不一定加载到了内存中。符号可以是任何形式的字面量，只要使用时能够准确的定位到目标即可。
* 直接引用：直接引用可以直接指向目标的指针、相对偏移量或是一个能间接定位到目标的句柄。直接引用与虚拟机布局有关，如果有了直接引用，那么引用的目标必定已经在内存中存在。

```

#### 初始化
在准备阶段，变量已经赋值过系统要求的默认值，在初始化阶段，则会根据程序制定的主观计划去初始化类变量和其他资源。这句话听起来有些绕口，根据上述例子，实际上就是：
```
test = new ClassLoadTest();// x = 1;y =1
y = 0;
```

这个过程，由于x咱们自己并没有去设定一个值，所以初始化阶段它不会发生任何改变,但是y咱们有设定一个值0，所以最后造成最终结果为x = 1;y = 0。
ps：在同一个类加载器下，一个类只会初始化一次。多个线程同时初始化一个类，只有一个线程能正常初始化，其他线程都会进行阻塞等待，直到活动线程执行初始化方法完毕。

如果将代码改为

``` java
public class ClassLoadTest {     
     
     static int x;
     static int y = 0;
     private static ClassLoadTest  test = new ClassLoadTest(); 
     public ClassLoadTest(){ x++; y++;}
     public static void main(String[]args){
         System.out.println(test.x);
         System.out.println(test.y);
     }
    
}
 ```
就会得到不一样的结果

```
y = 0;
test = new ClassLoadTest();// x = 1;y =1
```

#### summary
* 先是准备阶段，将成员变量初始化，基础类型为0，对象用null。
* 然后是解析阶段，将对象一一赋值，从上到下，先从x，y开始，然后test。
