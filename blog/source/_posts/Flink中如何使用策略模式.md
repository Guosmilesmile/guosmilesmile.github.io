---
title: Flink中如何使用策略模式
date: 2020-07-05 11:18:11
tags:
categories:
	- Flink
---


## 前言

转自https://mp.weixin.qq.com/s?__biz=MzUzMDYwOTAzOA==&mid=2247484204&idx=1&sn=fe7c79fd59ae3ca30f4b28fe7a1fba8d&chksm=fa4e65cdcd39ecdbbdc27cb0aeda51b4f151de18c8725725c1a7cef1938d7bfced772d9d34a4&mpshare=1&scene=24&srcid=0704F1iuTBgVFwCn24ZxMc2d&sharer_sharetime=1593853582031&sharer_shareid=797dbcdd3a4e624875c639b16a4ef5d9#rd


## 正文

策略模式是一个非常简单且常用的设计模式，策略模式最常见的作用就是解决代码中冗长的 if-else 或 switch 分支判断语句。本文的标题是《如此简单的策略模式也能秀操作？》，问题来了：策略模式如此简单，笔者今天到底能秀出什么样的操作呢？对策略模式比较熟悉的同学建议快速过一下前半部分，本文后半部分应该会让熟悉策略模式的同学也会有一些收获。本文重点在于笔者阅读 Flink 源码过程中发现了一个设计比较巧妙的点，可以对策略模式进行优化，所以特意写篇文章总结输出一下。



本文主要讲述：

* 什么场景需要使用策略模式，即：策略模式的作用，策略模式解决了什么问题
* 策略模式的定义和使用
* 秀操作：策略工厂类中，如果每次都要返回新的策略对象，怎么优化 if-else 分支判断逻辑？
* 

### 1、 什么场景需要使用策略模式？


如下代码所示，代码中 process 方法根据 type 的不同，执行不同分支的代码逻辑。


```java
// 根据 type 的不同，执行不同分支的代码逻辑，
private void process(String type){
  if(type == "A"){
    // 执行一大堆 A 类型的代码逻辑

  } else if (type == "B") {
    // 执行一大堆 B 类型的代码逻辑

  } else if (type == "C") {
    // 执行一大堆 C 类型的代码逻辑

  } else {
    // 上述都没有匹配成功，执行默认的代码逻辑，
    // 或者抛出异常打印错误日志等
  }
}
```

process 方法中会具体执行 A、B、C 三种不同类型的代码逻辑，假设三种类型的代码逻辑都比较繁琐，那么 process 方法将会非常长。设计模式还需要解决代码易扩展的问题，随着业务的发展，如果出现了 D、E、F 类型，是不是 process 方法中还需要增加了一堆 if-else 判断逻辑，process 方法会变得非常冗长且不可维护。

解决冗长 if-else 的方法很多，今天主要描述策略模式如果解决该问题。



### 2、 策略模式的定义和使用

#### 2.1 策略模式的定义


引用极客时间-王争老师的《设计模式之美》课程中对策略模式的定义：


```
策略模式，英文全称是 Strategy Design Pattern。在 GoF 的《设计模式》一书中，它是这样定义的：

Define a family of algorithms, encapsulate each one, and make them interchangeable. Strategy lets the algorithm vary independently from clients that use it.

翻译成中文就是：定义一组算法类，将每个算法分别封装起来，让它们可以互相替换。策略模式可以使算法的变化独立于使用它们的客户端（这里的客户端代指使用算法的代码）。
```

经上述分析，笔者认为策略模式可以达到这样的效果：假设 A 类调用 B 类，那么可以认为 A 类是 B 类的客户端，当 B 类增加了一些策略时，客户端 A 类不用进行任何的代码改动即可使用新策略。

策略模式的使用包含三部分：策略的定义、创建和使用。

### 2.2 策略的定义

策略类的定义比较简单，包含一个策略接口和一组实现这个接口的策略类。所有的策略类都实现相同的接口，所以，客户端代码基于接口而非实现编程，可以灵活地替换不同的策略。


```java
public interface Strategy {
  void algorithmInterface();
}

public class ConcreteStrategyA implements Strategy {
  @Override
  public void  algorithmInterface() {
    //具体的算法...
  }
}

public class ConcreteStrategyB implements Strategy {
  @Override
  public void  algorithmInterface() {
    //具体的算法...
  }
}
```


如上述代码所示，定义了一个策略接口 Strategy，具体的策略实现类都实现 Strategy 接口，并重写 algorithmInterface 方法，最后客户端根据不同的策略即可调用不同实现类的 algorithmInterface 方法。


### 2.3 策略的创建


可以将创建策略的代码逻辑抽象到工厂类中，提前在工厂类创建好所有策略类，缓存在 Map 中。Map 的 key 为策略类型，value 为具体的策略实现类。当需要使用策略时根据 type 去 Map 中 get 即可获取到相应的策略实现类。


```java
public class StrategyFactory {
  // Map 的 key 为策略类型，value 为具体的策略实现类
  private static final Map<String, Strategy> strategies = new HashMap<>();

  // 提前创建好所有策略类，缓存到 Map 中
  static {
    strategies.put("A", new ConcreteStrategyA());
    strategies.put("B", new ConcreteStrategyB());
  }

  // 需要使用策略时根据 type 去 Map 中 get 即可获取到相应的策略实现类
  public static Strategy getStrategy(String type) {
    if (type == null || type.isEmpty()) {
      throw new IllegalArgumentException("type should not be empty.");
    }
    return strategies.get(type);
  }
}
```

这里重点在于使用"查表法"代替了大量的分支判断，即每次根据 type 去 Map 中获取，省略了大量的 if-else。

#### 2.4 策略的使用


具体使用 A、B、C 何种策略，在具体的场景，可以会根据系统的配置来选择。可以从配置文件中读取出配置，然后传递给策略工厂类 StrategyFactory 的 getStrategy 方法即可获取到相应的策略类。最后调用策略类的 algorithmInterface 方法去执行代码逻辑。

代码如下所示，省略了从配置文件中读取配置的流程。

```java

// 根据 type 的不同，执行不同分支的代码逻辑，
private void process(String type){
 Strategy strategy = StrategyFactory.getStrategy(type);
  // 调用策略类的 algorithmInterface 方法去执行代码逻辑
  strategy.algorithmInterface();
}

```


#### 2.5 当前设计是否容易扩展性


最原始的 process 方法中，首先会判断各种 type，然后执行不同类型的代码逻辑。如果需要扩展新的 D、E、F 类型，需要大量修改 process 方法。

优化后，如果需要扩展新的 D、E、F 类型，流程如下：

* 定义相应的 D、E、F 类型的策略实现类
* 提前在策略工厂类 StrategyFactory 中创建相应的策略实现类，并添加到 Map 中
* 客户端代码不用进行任何改动，即：process 方法不需要进行改动

优化后的代码相对来说职责更加单一，且对调用方非常友好。调用方代码不需要任何改动即可使用新的策略。要做的可能就是在配置文件中配置新的策略即可。

到这里，策略模式的基本知识就讲完了。


### 3、 有状态的策略类不能提前创建


在策略类的创建部分，在类初始化时，将所有的策略类提前创建好，存放在 Map 中。当需要使用策略时根据 type 去 Map 中 get 即可获取到相应的策略实现类。

假设策略类是有状态的，每次获取策略对象时，都要求创建新的策略类。此时，就不能使用 Map 缓存的方式来优化代码结构了。可以使用如下方式实现策略工厂类：


```java
public class StrategyFactory {
  public static Strategy getStrategy(String type) {
    if (type == null || type.isEmpty()) {
      throw new IllegalArgumentException("type should not be empty.");
    }

    if (type.equals("A")) {
      return new ConcreteStrategyA();
    } else if (type.equals("B")) {
      return new ConcreteStrategyB();
    }

    return null;
  }
}
```

上述代码又退化回了 if-else 嵌套，当然也可以优化为 switch-case 的设计。

极客时间-王争老师的《设计模式之美》课程第 60 节最后的课堂讨论中留下了一道题目：在策略工厂类中，如果每次都要返回新的策略对象，我们还是需要在工厂类中编写 if-else 分支判断逻辑，那这个问题该如何解决呢？

笔者看到文末大家的回答，点赞数最高的评论是：仍然可以用查表法，只不过存储的不再是实例，而是class，使用时获取对应的class，再通过反射创建实例。

反射在这里应该是可以实现，但是笔者感觉不是非常灵活，假设策略实现类需要在这里调用一些有参构造器，且不同的策略类的有参构造器需要传入的参数不同，那么反射实现起来不是非常灵活。

例如 ConcreteStrategyA 的构造器需要传入 age，ConcreteStrategyB 的构造器需要传入 date。对于这样的 case，反射不太好实现，如果实现出来，也是一对 if-else 分支判断。

文末也没有其他令笔者眼前一亮的回答，反倒是笔者在阅读 Flink 源码的过程中，发现了一个笔者感觉比较优秀的解决方案，下面就到了秀操作环节。



### 秀操作


首先分析一个问题来源：


* 对于无状态的策略类，将所有的策略类提前创建好，存放在 Map 中。当需要使用策略时根据 type 去 Map 中 get 即可获取到相应的策略实现类。
* 对于有状态的策略类，不能提前创建所有的策略类，所以没办法提前创建好将其存放在 Map 中


换种思路：

* 给每个具体的策略类创建相应的策略类工厂。例如 ConcreteStrategyA 的工厂为 StrategyFactoryA， ConcreteStrategyB 的工厂为 StrategyFactoryB。
* 虽然没办法提前创建好策略类放到 Map 中，但是可以将策略类的工厂类提前创建好放到 Map 中。根据传入的 type 就可以从 Map 中获取相应策略类的工厂类，然后执行工厂类的 create 方法即可创建出相应的策略类。


根据上述思路，实现相应代码。

首先定义策略工厂接口，并分别实现策略 A 和策略 B 的工厂类：


```java
// 策略工厂接口
public interface StrategyFactory {
  Strategy create();
}

// 策略 A 的工厂类，用于创建策略 A
public class StrategyFactoryA implements StrategyFactory{
    @Override
    public Strategy create() {
        return new ConcreteStrategyA();
    }
}

// 策略 B 的工厂类，用于创建策略 B
public class StrategyFactoryB implements StrategyFactory{
    @Override
    public Strategy create() {
        return new ConcreteStrategyB();
    }
}
```


对外开放的工厂实现如下：


```java
public class Factory {

    // Map 的 key 为策略类型，value 为 策略的工厂类
    private static final Map<String, StrategyFactory> 
      STRATEGY_FACTORIES = new HashMap<>();

    static {
       // 将各种实现类的工厂提前创建好放到 Map 中
        STRATEGY_FACTORIES.put("A", new StrategyFactoryA());
        STRATEGY_FACTORIES.put("B", new StrategyFactoryB());
    }

    public static Strategy getStrategy(String type) {
        if (type == null || type.isEmpty()) {
            throw new IllegalArgumentException("type should not be empty.");
        }
        // 根据 type 获取对应的策略工厂
        StrategyFactory strategyFactory = STRATEGY_FACTORIES.get(type);
       // 调用具体工厂类的 create 方法即可创建出相应的策略类
        return strategyFactory.create();
    }
}
```


Factory 类中定义了 Map，Map 的 key 为策略类型，value 为 策略的工厂类。Factory 类初始化时，将各种实现类的工厂提前创建好放到 Map 中。

Factory 类的静态方法 getStrategy 用于根据 type 创建相应的策略类，getStrategy 方法根据 type 从 Map 中获取 type 对应的策略类的工厂，调用具体工厂类的 create 方法即可创建出相应的策略类。

当策略类的构造方法比较复杂也没关系，封装在策略类相应的工厂中即可。

旧方案对于每次要创建新策略类的场景，要搞一堆 if-else 分支判断，上述流程使用 Map 优化了 if-else 分支判断逻辑。但带来了一个新的问题，即：创建出了很多类，相比之前的实现来讲，多了 StrategyFactoryA 和 StrategyFactoryB 类。

为了代码的简洁，可以利用 Java8 的 lambda 表达式将  StrategyFactoryA 和 StrategyFactoryB 类优化掉。截取上述部分代码实现：

```java
// 策略工厂接口
public interface StrategyFactory {
  Strategy create();
}

// 策略 A 的工厂类，用于创建策略 A
public class StrategyFactoryA implements StrategyFactory{
    @Override
    public Strategy create() {
        return new ConcreteStrategyA();
    }
}

Map<String, StrategyFactory> STRATEGY_FACTORIES = new HashMap<>();
// 将各种实现类的工厂提前创建好放到 Map 中
STRATEGY_FACTORIES.put("A", new StrategyFactoryA());
```

上述代码使用 lambda 优化后：

```java

// 策略工厂接口
public interface StrategyFactory {
  Strategy create();
}

Map<String, StrategyFactory> STRATEGY_FACTORIES = new HashMap<>();
// 将各种实现类的工厂提前创建好放到 Map 中
STRATEGY_FACTORIES.put("A", () -> new ConcreteStrategyA());
```


关于 lambda 这里就不多解释了。lambda 表达式还能优化为 Java8 的方法引用，代码如下所示：

STRATEGY_FACTORIES.put("A", ConcreteStrategyA::new);


是不是瞬间就觉得很眼熟了。

### 小结

把上述整个代码的最终版贴在这里：

```java

// 策略工厂接口
public interface StrategyFactory {
  Strategy create();
}

public class Factory {

    // Map 的 key 为策略类型，value 为 策略的工厂类
    private static final Map<String, StrategyFactory> 
      STRATEGY_FACTORIES = new HashMap<>();

    static {
       // 将各种实现类的工厂提前创建好放到 Map 中
        STRATEGY_FACTORIES.put("A", ConcreteStrategyA::new);
        STRATEGY_FACTORIES.put("B", ConcreteStrategyB::new);
    }

    public static Strategy getStrategy(String type) {
        if (type == null || type.isEmpty()) {
            throw new IllegalArgumentException("type should not be empty.");
        }
        // 根据 type 获取对应的策略工厂
        StrategyFactory strategyFactory = STRATEGY_FACTORIES.get(type);
       // 调用具体工厂类的 create 方法即可创建出相应的策略类
        return strategyFactory.create();
    }
}
```

代码量相比之前的策略类可以共享的代码设计来讲，只是增加了一个 StrategyFactory 接口的设计，所以整体代码也是非常简洁的。


