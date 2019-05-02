---
title: Assemble（系列二）java基础
date: 2019-05-01 11:39:11
tags:
categories: Assemble
---


## java 向上转型和向下转型

* 向上转型：子类对象转为父类，父类可以是接口。
公式：Father f = new Son();
Father是父类或接口，son是子类。
* 向下转型：父类对象转为子类。公式：Son s = (Son)f;

向上转型没有什么好说的，就是多态，看下向下转型。
```java
public class Human {
    public void sleep() {
        System.out.println("Human sleep..");
    }

    public static void main(String[] args) {
        Human h = new Male();// 向上转型
        Human h1 = new Human();
        //h.speak();此时需要向下转型，否则不能调用speak方法。
         Male m = (Male) h;
         m.speak();
         /**Male m1 = (Male)h1;
         m1.speak();    此时会出现运行时错误，所以可以用instanceOF判断*/
         if (h1 instanceof Male){
             Male m1 = (Male)h1;
             m1.speak();
             
         }
    }
}

class Male extends Human {
    @Override
    public void sleep() {
        System.out.println("Male sleep..");
    }

    public void speak() {
        System.out.println("I am Male");
    }
}
```
##### 注意点
向下转型需要考虑安全性，如果父类引用的对象是父类本身，那么在向下转型的过程中是不安全的，编译不会出错，但是运行时会出现java.lang.ClassCastException错误。它可以使用instanceof来避免出错此类错误即能否向下转型，只有先经过向上转型的对象才能继续向下转型。

## String,StringBuffer, StringBuilder 的区别是什么？String为什么是不可变的？
1. String是字符串常亮，其余两个是字符串变量，后者内容可变，前者创建后内容不可变。
2. String不可变是因为jdk中String类被声明成final
3. StringBuffer是线程安全的，StringBulider是非线程安全的
