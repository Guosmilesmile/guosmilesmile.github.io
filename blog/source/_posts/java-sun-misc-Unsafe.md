---
title: java sun.misc.Unsafe
date: 2019-09-17 23:36:22
tags:
categories:
	- Java
---



### 初始化

初始化的代码主要包括调用JVM本地方法registerNatives()和sun.reflect.Reflection#registerMethodsToFilter。然后新建一个Unsafe实例命名为theUnsafe，通过静态方法getUnsafe()获取，获取的时候需要做权限判断。由此可见，Unsafe使用了单例设计(可见构造私有化了)。Unsafe类做了限制，如果是普通的调用的话，它会抛出一个SecurityException异常；只有由主类加载器(BootStrap classLoader)加载的类才能调用这个类中的方法。

```java
private static native void registerNatives();
    static {
        registerNatives();
        sun.reflect.Reflection.registerMethodsToFilter(Unsafe.class, "getUnsafe");
    }

    private Unsafe() {}

    private static final Unsafe theUnsafe = new Unsafe();

    @CallerSensitive
    public static Unsafe getUnsafe() {
        Class<?> caller = Reflection.getCallerClass();
        if (!VM.isSystemDomainLoader(caller.getClassLoader()))
            throw new SecurityException("Unsafe");
        return theUnsafe;
    }
```

最简单的使用方式是基于反射获取Unsafe实例。

```java
Field f = Unsafe.class.getDeclaredField("theUnsafe");
f.setAccessible(true);
Unsafe unsafe = (Unsafe) f.get(null);
```
### 类、对象和变量相关方法

#### getObject

```java
public native Object getObject(Object o, long offset);
```

通过给定的Java变量获取引用值。这里实际上是获取一个Java对象o中，获取偏移地址为offset的属性的值，此方法可以突破修饰符的抑制，也就是无视private、protected和default修饰符。类似的方法有getInt、getDouble等等。

可以使用如下的方法获取私有变量对应的offset。
```java
long staticFieldOffset = unsafe.objectFieldOffset(Person.class.getDeclaredField("name"));
```

获取static变量
```java
 long staticFieldOffset = unsafe.staticFieldOffset(Person.class.getDeclaredField("numberStatic"));
```

通过下面的方法获取对应的数值
```java
unsafe.getObject(person, staticFieldOffset)
```

#### putObject

```java
public native void putObject(Object o, long offset, Object x);
```

将引用值存储到给定的Java变量中。这里实际上是设置一个Java对象o中偏移地址为offset的属性的值为x，此方法可以突破修饰符的抑制，也就是无视private、protected和default修饰符。类似的方法有putInt、putDouble等等。


#### getObjectVolatile

```java
public native Object getObjectVolatile(Object o, long offset);
```
方法和上面的getObject功能类似，不过附加了'volatile'加载语义，也就是强制从主存中获取属性值。类似的方法有getIntVolatile、getDoubleVolatile等等。这个方法要求被使用的属性被volatile修饰，否则功能和getObject方法相同。


#### staticFieldOffset

```java
public native long staticFieldOffset(Field f);
```

返回给定的静态属性在它的类的存储分配中的位置(偏移地址)。不要在这个偏移量上执行任何类型的算术运算，它只是一个被传递给不安全的堆内存访问器的cookie。注意：这个方法仅仅针对静态属性，使用在非静态属性上会抛异常。

### objectFieldOffset

```java
public native long objectFieldOffset(Field f);
```

返回给定的非静态属性在它的类的存储分配中的位置(偏移地址)。不要在这个偏移量上执行任何类型的算术运算，它只是一个被传递给不安全的堆内存访问器的cookie。注意：这个方法仅仅针对非静态属性，使用在静态属性上会抛异常。


### staticFieldBase

```java
public native Object staticFieldBase(Field f);
```
返回给定的静态属性的位置，配合staticFieldOffset方法使用。实际上，这个方法返回值就是静态属性所在的Class对象的一个内存快照。注释中说到，此方法返回的Object有可能为null，它只是一个'cookie'而不是真实的对象，不要直接使用的它的实例中的获取属性和设置属性的方法，它的作用只是方便调用上面提到的像getInt(Object,long)等等的任意方法。

### arrayBaseOffset

```java
public native int arrayBaseOffset(Class<?> arrayClass);
```

返回数组类型的第一个元素的偏移地址(基础偏移地址)。如果arrayIndexScale方法返回的比例因子不为0，你可以通过结合基础偏移地址和比例因子访问数组的所有元素。Unsafe中已经初始化了很多类似的常量如ARRAY_BOOLEAN_BASE_OFFSET等。

### arrayIndexScale

```java
public native int arrayIndexScale(Class<?> arrayClass);
```

返回数组类型的比例因子(其实就是数据中元素偏移地址的增量，因为数组中的元素的地址是连续的)。此方法不适用于数组类型为"narrow"类型的数组，"narrow"类型的数组类型使用此方法会返回0(这里narrow应该是狭义的意思，但是具体指哪些类型暂时不明确，笔者查了很多资料也没找到结果)。Unsafe中已经初始化了很多类似的常量如ARRAY_BOOLEAN_INDEX_SCALE等。

```java
// 数组第一个元素的偏移地址,即数组头占用的字节数
int[] intarr = new int[0];
System.out.println(unsafe.arrayBaseOffset(intarr.getClass()));

// 数组中每一个元素占用的大小
System.out.println(unsafe.arrayIndexScale(intarr.getClass()));
```

### 内存管理



#### addressSize
```
public native int addressSize();
```

获取本地指针的大小(单位是byte)，通常值为4或者8。常量ADDRESS_SIZE就是调用此方法。


#### pageSize
```java
public native int pageSize();
```
获取本地内存的页数，此值为2的幂次方。

#### allocateMemory
```java
public native long allocateMemory(long bytes);
```
分配一块新的本地内存，通过bytes指定内存块的大小(单位是byte)，返回新开辟的内存的地址。如果内存块的内容不被初始化，那么它们一般会变成内存垃圾。生成的本机指针永远不会为零，并将对所有值类型进行对齐。可以通过freeMemory方法释放内存块，或者通过reallocateMemory方法调整内存块大小。bytes值为负数或者过大会抛出IllegalArgumentException异常，如果系统拒绝分配内存会抛出OutOfMemoryError异常。

#### reallocateMemory
```java
public native long reallocateMemory(long address, long bytes);
```
通过指定的内存地址address重新调整本地内存块的大小，调整后的内存块大小通过bytes指定(单位为byte)。可以通过freeMemory方法释放内存块，或者通过reallocateMemory方法调整内存块大小。bytes值为负数或者过大会抛出IllegalArgumentException异常，如果系统拒绝分配内存会抛出OutOfMemoryError异常。

#### setMemory
```java
public native void setMemory(Object o, long offset, long bytes, byte value);
```
将给定内存块中的所有字节设置为固定值(通常是0)。内存块的地址由对象引用o和偏移地址共同决定，如果对象引用o为null，offset就是绝对地址。第三个参数就是内存块的大小，如果使用allocateMemory进行内存开辟的话，这里的值应该和allocateMemory的参数一致。value就是设置的固定值，一般为0(这里可以参考netty的DirectByteBuffer)。一般而言，o为null，所有有个重载方法是public native void setMemory(long offset, long bytes, byte value);，等效于setMemory(null, long offset, long bytes, byte value);。


### Reference 

https://www.cnblogs.com/throwable/p/9139947.html

https://blog.csdn.net/liao0801_123/article/details/85066921

https://blog.csdn.net/ahilll/article/details/81628215