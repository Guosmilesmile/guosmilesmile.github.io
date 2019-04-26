---
title: java8 Map merge、compute、computeIfAbsent、computeIfPresent
date: 2019-03-30 10:46:12
tags:
categories: Java
---

### 单词计数

```java
var words = List.of("Foo", "Bar", "Foo", "Buzz", "Foo", "Buzz", "Fizz", "Fizz");

```

##### 普通写法

```java
var map = new HashMap<String, Integer>();
words.forEach(word -> {
    var prev = map.get(word);
    if (prev == null) {
        map.put(word, 1);
    } else {
        map.put(word, prev + 1);
    }
});


```


##### putIfAbsent写法

```
words.forEach(word -> {
    map.putIfAbsent(word, 0);
    map.put(word, map.get(word) + 1);
});

```
putIfAbsent 如果不存在该key才执行，则put对应的value到对应的key。

##### computeIfPresent写法

```java
words.forEach(word -> {
    map.putIfAbsent(word, 0);
    map.computeIfPresent(word, (w, prev) -> prev + 1);
});
```
computeIfPresent是仅当 word中的的key存在的时候才调用给定的转换。否则它什么都不处理。我们通过将key初始化为零来确保key存在，因此增量始终有效。




##### computer写法
```java
words.forEach(word ->
        map.compute(word, (w, prev) -> prev != null ? prev + 1 : 1)
);
```

compute ()就像是computeIfPresent()，但无论给定key的存在与否如何都会调用它。如果key的值不存在，则prev参数为null


##### merge写法

merge() 适用于两种情况。如果给定的key不存在，它就变成了put(key, value)。但是，如果key已经存在一些值，我们  remappingFunction 可以选择合并的方式。这个功能是完美契机上面的场景：

* 只需返回新值即可覆盖旧值： (old, new) -> new
* 只需返回旧值即可保留旧值：(old, new) -> old
* 以某种方式合并两者，例如：(old, new) -> old + new
* 甚至删除旧值：(old, new) -> null

```java
words.forEach(word ->
        map.merge(word, 1, (prev, one) -> prev + one)
);
```

#### put 和 compute 和 computeIfAbsent 和 putIfAbsent 差别？

* put返回旧值，如果没有则返回null，并且替换该key的value
* compute（相当于put,只不过返回的是新值），并且替换该key的value
* putIfAbsent返回旧值，如果没有则返回null 如果不存在新增，如果存在，不操作
* computeIfAbsent:存在时返回存在的值，不存在时返回新值。如果存在不替换