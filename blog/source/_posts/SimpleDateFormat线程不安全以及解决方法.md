---
title: SimpleDateFormat线程不安全以及解决方法
date: 2019-03-25 23:50:29
tags:
categories: Java
---

### 前言

SimpleDateFormat主要是用它进行时间的格式化输出和解析，挺方便快捷的，但是SimpleDateFormat**并不是一个线程安全的类**。

##### 先看看《阿里巴巴开发手册》对于SimpleDateFormat是怎么看待的：
【强制】SimpleDateFormat 是线程不安全的类，一般不要定义为 static变量，如果定义为
static，必须加锁，或者使用 DateUtils 工具类。
正例：注意线程安全，使用 DateUtils。亦推荐如下处理：
```java
private static final ThreadLocal<DateFormat> df = new ThreadLocal<DateFormat>() {
    @Override
    protected DateFormat initialValue() {
    return new SimpleDateFormat("yyyy-MM-dd");
    }
};
```
说明：如果是 JDK8 的应用，可以使用 Instant 代替 Date，LocalDateTime 代替 Calendar，DateTimeFormatter代替Simpledateformatter，官方给出的解释：simple beautiful strong immutable thread-safe。

### 原因分析

一般我们使用SimpleDateFormat的时候会把它定义为一个静态变量，避免频繁创建它的对象实例，如下代码：
```java

public class simpleDateTest {

    private static final SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");

    public static String formatDate(Date date) {
        return sdf.format(date);
    }


    public static Date formatDate(String date) throws ParseException {
        return sdf.parse(date);
    }

}

```

在单线程是没有问题的，在多线程中就会出问题。因为我们把SimpleDateFormat定义为静态变量，那么多线程下SimpleDateFormat的实例就会被多个线程共享，B线程会读取到A线程的时间，就会出现时间差异和其它各种问题。SimpleDateFormat和它继承的DateFormat类也不是线程安全的。


我们来看下format方法

```java
// Called from Format after creating a FieldDelegate
    private StringBuffer format(Date date, StringBuffer toAppendTo,
                                FieldDelegate delegate) {
        // Convert input date to time field list
        calendar.setTime(date);

        boolean useDateFormatSymbols = useDateFormatSymbols();

        for (int i = 0; i < compiledPattern.length; ) {
            int tag = compiledPattern[i] >>> 8;
            int count = compiledPattern[i++] & 0xff;
            if (count == 255) {
                count = compiledPattern[i++] << 16;
                count |= compiledPattern[i++];
            }

            switch (tag) {
            case TAG_QUOTE_ASCII_CHAR:
                toAppendTo.append((char)count);
                break;

            case TAG_QUOTE_CHARS:
                toAppendTo.append(compiledPattern, i, count);
                i += count;
                break;

            default:
                subFormat(tag, count, delegate, toAppendTo, useDateFormatSymbols);
                break;
            }
        }
        return toAppendTo;
    }
```
注意， calendar.setTime(date)，SimpleDateFormat的format方法实际操作的就是**Calendar**。

因为我们声明SimpleDateFormat为static变量，那么它的Calendar变量也就是一个共享变量，可以被多个线程访问。

假设线程A执行完calendar.setTime(date)，把时间设置成2019-01-02，这时候被挂起，线程B获得CPU执行权。线程B也执行到了calendar.setTime(date)，把时间设置为2019-01-03。线程挂起，线程A继续走，calendar还会被继续使用(subFormat方法)，而这时calendar用的是线程B设置的值了，而这就是引发问题的根源，出现时间不对，线程挂死等等。

**其实SimpleDateFormat源码上作者也给过我们提示：**
```java
 * Date formats are not synchronized.
 * It is recommended to create separate format instances for each thread.
 * If multiple threads access a format concurrently, it must be synchronized
 * externally.
日期格式不同步。

建议为每个线程创建单独的格式实例。

如果多个线程同时访问一种格式，则必须在外部同步该格式。
```

### 解决方法

1. 只在需要的时候创建新实例，不用static修饰。

```
 public static String formatDate(Date date) {
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        return sdf.format(date);
    }


    public static Date formatDate(String date) throws ParseException {
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        return sdf.parse(date);
    }
```

**不过也加重了创建对象的负担，会频繁地创建和销毁对象，效率较低。**

2. 无脑synchronized

```java

    public synchronized static String formatDate(Date date) {

        return sdf.format(date);
    }


    public synchronized static Date formatDate(String date) throws ParseException {
        return sdf.parse(date);
    }
```
** 简单粗暴，synchronized往上一套也可以解决线程安全问题，缺点自然就是并发量大的时候会对性能有影响，线程阻塞。**

3. ThreadLocal

``` java
public class simpleDateTest {

    private static ThreadLocal<DateFormat> threadLocal = new ThreadLocal<DateFormat>() {
        @Override
        protected DateFormat initialValue() {
            return new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        }
    };

    public  static String formatDate(Date date) {

        return threadLocal.get().format(date);
    }


    public  static Date formatDate(String date) throws ParseException {
        return threadLocal.get().parse(date);
    }

}

```

ThreadLocal可以确保每个线程都可以得到单独的一个SimpleDateFormat的对象，那么自然也就不存在竞争问题了。

4、基于JDK1.8的DateTimeFormatter

```
public class simpleDateTest {

   private static final DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");


    public  static String formatDate(LocalDateTime date) {

        return formatter.format(date);
    }


    public  static LocalDateTime formatDate(String date) throws ParseException {
        return LocalDateTime.parse(date,formatter);
    }
}
```