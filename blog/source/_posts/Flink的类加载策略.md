---
title: Flink的类加载策略
date: 2020-08-19 22:21:43
tags:
categories: Flink
---





在JVM中，一个类加载的过程大致分为加载、链接（验证、准备、解析）、初始化.
而我们通常提到类的加载，就是指利用类加载器（ClassLoader）通过类的全限定名来获取定义此类的二进制字节码流，进而构造出类的定义。Flink作为基于JVM的框架，在flink-conf.yaml中提供了控制类加载策略的参数classloader.resolve-order，可选项有child-first（默认）和parent-first。本文来简单分析一下这个参数背后的含义。

![image](https://note.youdao.com/yws/api/personal/file/4B005588A08E43E89B3BCFDB6FF4353A?method=download&shareKey=75bdeef77cc407661a482bec334bfe73)

### parent-first类加载策略

ParentFirstClassLoader和ChildFirstClassLoader类的父类均为URLClassLoader,这两个类都可以通过FlinkUserCodeClassLoaders.create创建。


```java
public static URLClassLoader create(
		ResolveOrder resolveOrder, URL[] urls, ClassLoader parent, String[] alwaysParentFirstPatterns) {

		switch (resolveOrder) {
			case CHILD_FIRST:
				return childFirst(urls, parent, alwaysParentFirstPatterns);
			case PARENT_FIRST:
				return parentFirst(urls, parent);
			default:
				throw new IllegalArgumentException("Unknown class resolution order: " + resolveOrder);
		}
	}
```


```java
/**
	 * Regular URLClassLoader that first loads from the parent and only after that from the URLs.
	 */
	static class ParentFirstClassLoader extends URLClassLoader {

		ParentFirstClassLoader(URL[] urls) {
			this(urls, FlinkUserCodeClassLoaders.class.getClassLoader());
		}

		ParentFirstClassLoader(URL[] urls, ClassLoader parent) {
			super(urls, parent);
		}
	}
```

因为Flink App的用户代码在运行期才能确定，所以通过URL在JAR包内寻找全限定名对应的类是比较合适的。而ParentFirstClassLoader仅仅是一个继承FlinkUserCodeClassLoader的空类而已。

这样就相当于ParentFirstClassLoader直接调用了父加载器的loadClass()方法。之前已经讲过，JVM中类加载器的层次关系和默认loadClass()方法的逻辑由双亲委派模型（parents delegation model）来体现

```
如果一个类加载器要加载一个类，它首先不会自己尝试加载这个类，而是把加载的请求委托给父加载器完成，所有的类加载请求最终都应该传递给最顶层的启动类加载器。只有当父加载器无法加载到这个类时，子加载器才会尝试自己加载。

```

Flink的parent-first类加载策略就是照搬双亲委派模型的。也就是说，用户代码的类加载器是Custom ClassLoader，Flink框架本身的类加载器是Application ClassLoader。用户代码中的类先由Flink框架的类加载器加载，再由用户代码的类加载器加载。但是，Flink默认并不采用parent-first策略，而是采用下面的child-first策略

### child-first类加载策略


我们已经了解到，双亲委派模型的好处就是随着类加载器的层次关系保证了被加载类的层次关系，从而保证了Java运行环境的安全性。但是在Flink App这种依赖纷繁复杂的环境中，双亲委派模型可能并不适用。例如，程序中引入的Flink-Cassandra Connector总是依赖于固定的Cassandra版本，用户代码中为了兼容实际使用的Cassandra版本，会引入一个更低或更高的依赖。而同一个组件不同版本的类定义有可能会不同（即使类的全限定名是相同的），如果仍然用双亲委派模型，就会因为Flink框架指定版本的类先加载，而出现莫名其妙的兼容性问题，如NoSuchMethodError、IllegalAccessError等。

鉴于此，Flink实现了ChildFirstClassLoader类加载器并作为默认策略。它打破了双亲委派模型，使得用户代码的类先加载，官方文档中将这个操作称为"Inverted Class Loading"。代码仍然不长，录如下。

```java
public final class ChildFirstClassLoader extends URLClassLoader {

	/**
	 * The classes that should always go through the parent ClassLoader. This is relevant
	 * for Flink classes, for example, to avoid loading Flink classes that cross the
	 * user-code/system-code barrier in the user-code ClassLoader.
	 */
	private final String[] alwaysParentFirstPatterns;

	public ChildFirstClassLoader(URL[] urls, ClassLoader parent, String[] alwaysParentFirstPatterns) {
		super(urls, parent);
		this.alwaysParentFirstPatterns = alwaysParentFirstPatterns;
	}

	@Override
	protected synchronized Class<?> loadClass(
		String name, boolean resolve) throws ClassNotFoundException {

		// First, check if the class has already been loaded
		Class<?> c = findLoadedClass(name);

		if (c == null) {
			// check whether the class should go parent-first
			for (String alwaysParentFirstPattern : alwaysParentFirstPatterns) {
				if (name.startsWith(alwaysParentFirstPattern)) {
					return super.loadClass(name, resolve);
				}
			}

			try {
				// check the URLs
				c = findClass(name);
			} catch (ClassNotFoundException e) {
				// let URLClassLoader do it, which will eventually call the parent
				c = super.loadClass(name, resolve);
			}
		}

		if (resolve) {
			resolveClass(c);
		}

		return c;
	}

	@Override
	public URL getResource(String name) {
		// first, try and find it via the URLClassloader
		URL urlClassLoaderResource = findResource(name);

		if (urlClassLoaderResource != null) {
			return urlClassLoaderResource;
		}

		// delegate to super
		return super.getResource(name);
	}

	@Override
	public Enumeration<URL> getResources(String name) throws IOException {
		// first get resources from URLClassloader
		Enumeration<URL> urlClassLoaderResources = findResources(name);

		final List<URL> result = new ArrayList<>();

		while (urlClassLoaderResources.hasMoreElements()) {
			result.add(urlClassLoaderResources.nextElement());
		}

		// get parent urls
		Enumeration<URL> parentResources = getParent().getResources(name);

		while (parentResources.hasMoreElements()) {
			result.add(parentResources.nextElement());
		}

		return new Enumeration<URL>() {
			Iterator<URL> iter = result.iterator();

			public boolean hasMoreElements() {
				return iter.hasNext();
			}

			public URL nextElement() {
				return iter.next();
			}
		};
	}
}

```
核心逻辑位于loadClassWithoutExceptionHandling()方法中，简述如下：

1. 调用findLoadedClass()方法检查全限定名name对应的类是否已经加载过，若没有加载过，再继续往下执行。
2. 检查要加载的类是否以alwaysParentFirstPatterns集合中的前缀开头。如果是，则调用父类的对应方法，以parent-first的方式来加载它。
3. 如果类不符合alwaysParentFirstPatterns集合的条件，就调用findClass()方法在用户代码中查找并获取该类的定义（该方法在URLClassLoader中有默认实现）。如果找不到，再fallback到父加载器来加载。
4. 最后，若resolve参数为true，就调用resolveClass()方法链接该类，最后返回对应的Class对象

可见，child-first策略避开了“先把加载的请求委托给父加载器完成”这一步骤，只有特定的某些类一定要“遵循旧制”。alwaysParentFirstPatterns集合中的这些类都是Java、Flink等组件的基础，不能被用户代码冲掉。它由以下两个参数来指定：

classloader.parent-first-patterns.default

```
java.;
scala.;
org.apache.flink.;
com.esotericsoftware.kryo;
org.apache.hadoop.;
javax.annotation.;
org.slf4j;
org.apache.log4j;
org.apache.logging;
org.apache.commons.logging;
ch.qos.logback;
org.xml;
javax.xml;
org.apache.xerces;
org.w3c
```

classloader.parent-first-patterns.additional：除了上一个参数指定的类之外，用户如果有其他类以child-first模式会发生冲突，而希望以双亲委派模型来加载的话，可以额外指定（分号分隔）。



### Reference

https://www.jianshu.com/p/bc7309b03407





