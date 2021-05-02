---
title: Flink 实战采坑记之 Kryo 序列化
date: 2021-05-02 15:14:32
tags:
categories: Flink
---




本文也是通过上述流程一步步解决了线上任务的问题，整体流程：

1. 看到现象猜想可能是序列化有问题
2. 修改 StateBackend 后结果正确，验证了猜想的正确性
3. 解释了为什么 LinkedHashMap 会序列化出错
4. 列出了具体的解决方案，解决方案可以举一反三（以后类似情况都可以使用本文的序列化方案）

### 异常现象
任务在测试环境运行符合预期，在线上环境运行有个数据每次都是最新的值，不会计算累计值了。

怀疑 LinkedHashMap 序列化有问题，为什么会这么怀疑呢？

因为之前这里使用的 guava 的 Cache，但 guava 的 Cache 不支持序列化，所以换成了 LinkedHashMap，但好像 LinkedHashMap 也没那么容易序列化。

#### Flink 哪些场景需要对数据进行序列化和反序列化？

## reduce导致yarn容器挂掉

* 上下游数据传输
* 读写 RocksDB 中的 State 数据
* 从 Savepoint 或 Checkpoint 中恢复状态数据

memory 或 filesystem 模式下，State 数据存在内存中，所以每次读写并不需要序列化和反序列化。

第一部分异常现象是任务在测试环境运行符合预期主要是因为测试环境 StateBackend 使用的 filesystem，所以没走序列化相关的逻辑，线上使用的是 RocksDB。

但如果状态中的数据类型序列化存在问题，是不是从 RocksDB 切到 memory 或 filesystem 模式就可以了呢？

不行，如果从 Checkpoint 和 Savepoint 恢复还需要走序列化逻辑，还是不能正常恢复。

#### 验证是否是序列化导致结果统计出错

线上环境将 RocksDB 改为 filesystem，结果就正确了，于是断定是因为序列化导致的结果错误。

具体到代码就是 LinkedHashMap 不能被 Kryo 正确地序列化和反序列化。

#### LinkedHashMap 如何使用？


LinkedHashMap 是 HashMap 的子类，相比 HashMap 增加了基于 LRU 的淘汰策略。

所以一般使用 LinkedHashMap 都是要使用其剔除策略功能，如果不需要该功能，HashMap 即可满足业务需求。

使用 LinkedHashMap 的代码一般会这么搞，搞一个 LinkedHashMap 的匿名内部类，并重写 LinkedHashMap 的 removeEldestEntry 方法，当 LinkedHashMap 中元素个数超过 maxSize 时，就会根据 LRU 策略，将最久没被使用过的数据从 LinkedHashMap 中剔除。

```java
new LinkedHashMap<K, V>() {
  @Override
  protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
    return size() > maxSize;
  }
};
```
如果上述 Flink 的 State 中存储了上述的 LinkedHashMap 对象，将会出问题。为什么呢？

答：Kryo 「不支持匿名类」，反序列化时往往会产生错误的数据（这比报错更加危险），请尽量不要使用匿名类传递数据。

Kryo 反序列化时，默认根据对象的无参构造器通过反射机制创建对象，匿名内部类哪来的无参构造器。

注：上述操作确实危险，Flink 如果直接报错反序列异常还好，用户可以直接定位到问题。现在的现象是 Flink 序列化不报错，只是跑出来的结果是错的，很蛋疼。

#### 解决方案


匿名内部类改写成普通类

#### 现象

```java

public class LinkedHashMapEnhance<K, V> extends LinkedHashMap<K, V> {
 
  private int maxSize;
 
  public LinkedHashMapEnhance() {
    super();
    maxSize = Integer.MAX_VALUE;
  }
 
  public LinkedHashMapEnhance(int maxSize) {
    super();
    this.maxSize = maxSize;
  }
 
  public int getMaxSize() {
    return maxSize;
  }
 
  @Override
  protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
    return size() > maxSize;
  }
 
  @Override
  public String toString() {
    return "LinkedHashMapEnhance : " + super.toString() + " maxSize:" + maxSize;
  }
}
```

LinkedHashMapEnhance 类继承 LinkedHashMap，且增加了 maxSize 的功能。

之后直接使用 LinkedHashMapEnhance 类即可，但是还存在问题：「maxSize 变量不能被正常的反序列化」。

因为 LinkedHashMapEnhance 实现了 Map 接口，都会默认走 kryo 对 Map 序列化的通用逻辑。


#### kryo 中的 MapSerializer 实现原理


kryo 代码中的 MapSerializer 类封装了通用的 Map 类型序列化和反序列化逻辑。

其中 write 方法表示序列化，read 方法表示反序列化。

序列化逻辑：
write 方法精简后的截图如下所示，大概逻辑：调用 Map 的迭代器，将 map 中所有 Key Value 数据遍历出来，依次序列化。

![image](https://mmbiz.qpic.cn/mmbiz_png/7iahLicCzg1mcLQE7RVmxw8JqbYVuSsC7AywMwsmbGnPAD4BquX2dvXxxvbpPzf6hF6XamRrAl3LdGtibiaWpnibbng/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)


反序列化逻辑：
read 方法精简后的截图如下所示，大概逻辑：反序列化出所有 Key Value 的数据，依次 put 到 Map 中。

![image](https://mmbiz.qpic.cn/mmbiz_png/7iahLicCzg1mcLQE7RVmxw8JqbYVuSsC7AIktQrTDQlrhfP67MWZ1VLrAZH0aWTGV1n4QyBicORp8HWHNsQbfycBw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

由上述原理分析可知：

Map 默认的序列化器只会序列化 Key Value 的数据，并没有序列化我们自定义的 maxSize 变量。

所以反序列化出来的 LinkedHashMapEnhance 类功能并不完善，maxSize 默认值为 Integer.MAX_VALUE，相当于没有容量上限，缺失了数据淘汰的功能。

庆幸的是 Kryo 支持用户自定义序列化器，我们可以为 LinkedHashMapEnhance 类定义特定的序列化器。

#### 自定义序列化器



自定义序列化器只需要实现 Kryo 的 Serializer 接口，并重写 read 和 write 方法即可。

序列化时，模仿 MapSerializer 将 map 的容量、maxSize、所有的 kv 数据依次序列化即可。反序列化也是类似。

具体代码如下所示：

```java
public class LinkedHashSetSerializer<K, V> extends Serializer<LinkedHashMapEnhance<K, V>> implements Serializable {
 
  private static final long serialVersionUID = -3335512745506751743L;
 
  @Override
  public void write(Kryo kryo, Output output, LinkedHashMapEnhance<K, V> linkedHashMapEnhance) {
    // 序列化 map 的容量和 maxSize
    kryo.writeObject(output, linkedHashMapEnhance.size());
    kryo.writeObject(output, linkedHashMapEnhance.getMaxSize());
 
    // 迭代器遍历一条条数据，将其序列化写出到 output
    for (Map.Entry<K, V> entry : linkedHashMapEnhance.entrySet()) {
      kryo.writeClassAndObject(output, entry.getKey());
      kryo.writeClassAndObject(output, entry.getValue());
    }
  }
 
  @Override
  public LinkedHashMapEnhance<K, V> read(Kryo kryo, Input input, Class<LinkedHashMapEnhance<K, V>> aClass) {
    // 先读取 map 的容量和 maxSize
    int size = kryo.readObject(input, Integer.class);
    int maxSize = kryo.readObject(input, Integer.class);
    // 构造 LinkedHashMapEnhance
    LinkedHashMapEnhance<K, V> map = new LinkedHashMapEnhance<>(maxSize);
 
    // for 循环遍历 size 次，每次读取出 key 和 value，并将其插入到 map 中
    for (int i = 0; i < size; i++) {
      K key = (K) kryo.readClassAndObject(input);
      V value = (V) kryo.readClassAndObject(input);
      map.put(key, value);
    }
    return map;
  }
}
```
最后只需要将序列化器注册给 Kyro 即可。原生的 Kryo 和 Flink 注册方式稍有不同，不过非常类似。

代码如下所示：


```java
// 原生 kryo 的注册方式
Kryo kryo = new Kryo();
kryo.register(LinkedHashSet.class, new LinkedHashSetSerializer());
 
// Flink 的注册方式
env.getConfig().registerTypeWithKryoSerializer(LinkedHashSet.class, LinkedHashSetSerializer.class);
```


当然如果你是平台方，想让用户通过参数传递来注册 Kryo 序列化，可以通过反射的方式实现。

使用反射注册的代码如下所示：

```java

// 要序列化的类名及序列化器的类名使用参数传递
String className = "com.dream.flink.kryo.LinkedHashSet";
String serializerClassName = "com.dream.flink.kryo.LinkedHashSetSerializer";
 
// 原生 kryo 的注册方式
Kryo kryo = new Kryo();
kryo.register(Class.forName(className),
        (Serializer) Class.forName(serializerClassName).newInstance());
 
// Flink 的注册方式
env.getConfig().registerTypeWithKryoSerializer(Class.forName(className),
                                               Class.forName(serializerClassName));
```


1. 服务器load值500+，无法界定是因为load值高导致容器被yarn认为异常剔除还是其他原因。
2. cpu普通很低
3. iotop查看到磁盘写入很高直接将load跑高，abrt-hook-ccpp 有多个进程 
 
```
abrt-hook-ccpp 是linux的程序，在进程崩溃的时候会将内存快照等信息dump到磁盘
```

#### 分析

1. 首先排除因为计算导致的cpu异常
2. 内存可能是一个导致爆炸的原因     
通过修改读取数据的大小，将数据压到200M，依旧出现这种情况，当时提供的服务器是5台，每台98G。开始出现灵异事件。
3. 将reduce内的操作全部剔除直接返回，程序正常运行。
4. 重点分析reduce内的操作.



## 个人案例

```java

.reduce(new ReduceFunction<DuplicateEntity>() {
    @Override
    public DuplicateEntity reduce(DuplicateEntity value1, DuplicateEntity value2) throws Exception {
        RangeLestEntity rangeEntities = value1.getRangeEntities();
        RangeLestEntity rangeEntities2 = value2.getRangeEntities();
        for (RangeEntity rangeEntity : rangeEntities2.getTreeSet()) {
            rangeEntities.addRange(rangeEntity);
        }
        value1.setRangeEntities(rangeEntities);
        return value1;
    }
})
```

只很对RangeLestEntity这个实体进行操作。


```java
//异常类
public class RangeLestEntity extends TreeSet<RangeEntity> {

    // 自定义addRange方法
}
```


将该类修改为如下操作，运行正常。

```java

//正常运行类
public class RangeLestEntity  {

    private TreeSet<RangeEntity> treeSet = new TreeSet<>();


    // 自定义addRange方法
}

```

如果是继承treeSet在序列化传输的时候，通过set序列化器的时候，没办法把自定义方法传递过去。所有会有问题。

如果是作为一个实体，那么就不会使用set的序列化器去序列化

### Reference
https://mp.weixin.qq.com/s/GJjZxpq4FIl4eiM_PQrdyA