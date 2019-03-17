---
layout: post
title: JDK中的设计模式
categories: [Java]
description: 分析JDK中的设计模式，更好的理解设计模式
keywords: Java, Design Pattern
---

tips: 分析JDK中的设计模式，更好的理解设计模式

## JDK中的设计模式——长期更新

## 引言

很多人学习设计模式都会看《大话设计模式》类似的书，看这种书有一个好处是便于理解各种设计模式指的是什么。但是，这种书很难让你在实际工程中想到可以用哪个设计模式，因为书中很多以现实世界的例子来讲解设计模式，然而，做起工程来跟现实世界还是有很大的差距的，我们总不太可能把工程问题映射为一个现实世界的问题进而再联想到设计模式。

本人在阅读部分JDK源码的时候，会发现源码中使用到一些设计模式，这个时候，往往有种醍醐灌顶的感觉，哦，原来这种情况下使用这个模式。并且，从JDK中学到的设计模式总会跟代码的业务逻辑相关，这样当我们在工程中遇到类似的业务逻辑的时候，可以很自然的联想到相应的设计模式，所以，通过JDK学习大神是怎样使用设计模式才是一个学习设计模式比较有实际意义的方法。本文总结了一些本人在之前看JDK源码中遇到的一些设计模式，以后随着本人对JDK源码的继续阅读，本文将会长期更新。

## 装饰器模式

* **目的**：主要是为了增加已有类的功能而使用的设计模式
* **实现方式**：装饰器类一般会提供和被装饰器类同名的函数，在函数中通过调用被装饰器的同名函数达到增加类功能的目的。
* **tips**:其实刚开始我是很疑惑的，增加已有类的功能，直接继承已有类，覆盖相应的方法就好了。后来《Effective Java》第16条：“复合优先于继承”这一节解开了我的困惑。主要原因在于：并不是所有类都适合继承。《Effective Java》第17条：“要么为继承而设计，并提供文档说明，要么就禁止继承”。JDK中很多类不是为继承而设计的，虽然他们没有显式的禁止继承，但是也没有提供详细的文档说明，因此，针对这种类，是不适合通过继承来增加功能的。具体的例子请参考《Effective Java》对应章节，例子非常经典。所以，当我们在需要增加已有类的功能，并且这个类不是为继承而设计的时候，我们要采用装饰器模式，也就是复合优先于继承的原则。
* **JDK实例**：JDK中IO部分很经典的使用了装饰器模式。以InputStream为例介绍, InputStream类图如下所示

![](/images/post/java_io_uml.png)

FilterInputStream是所有使用装饰器类的父类，InflaterInputStream增加了数据压缩的功能。BufferedInputStream增加了缓冲区的功能，DataInputStream增加了按照数据类型读取数据的功能。以BufferedInputStream为例，BufferedInputStream的构造函数如下：

	public BufferedInputStream(InputStream in) {
        this(in, DEFAULT_BUFFER_SIZE);
    }
需要为BufferedInputStream提供一个InputStream，也就是被包装的类。BufferedInputStream的read函数代码如下：
 	
	public synchronized int read() throws IOException {
        if (pos >= count) {
            fill();
            if (pos >= count)
                return -1;
        }
        return getBufIfOpen()[pos++] & 0xff;
    }
函数首先判断缓冲区内数据是否已经读完，如果读完调用fill()函数，fill函数实际是通过调用in(被包装类)的read函数实现每次读取一块数据达到数据缓冲的目的。

## 代理模式

* **目的**：限制对被代理类函数访问的权限。
* **实现方式**：代理类对被代理类的函数分两种情况处理：
	* 正常代理的函数，只需要提供和被代理函数同名的函数，函数内部一般直接调用被代理函数对应同名函数即可。
	* 限制访问的函数，可以不提供相应的函数接口或在函数中直接抛出异常实现。
* **tips**：仅仅从代码结构来看，装饰器模式和代理模式还是很像的，都是内部有一个被代理或被装饰类的对象，提供被代理或被装饰类的同名函数。两者主要区别在于目的不同，主要有以下区别，
	* 装饰器为了增加被装饰类的功能，所以一般会提供被装饰类的所有函数；代理类则是为了限制被代理类部分函数的访问，一般提供被代理类部分的函数，禁止被代理类另一部分的函数。
	* 装饰器在同名函数中一般会有额外的逻辑达到增加功能的目的；代理类在同名函数中直接调用被代理类的函数，一般没有额外的逻辑。
* **JDK实例**：JDK中AQS类就是为了代理模式设计的，作者也建议开发人员以代理的形式使用AQS类。具体关于AQS类的使用可以看[这里](https://niceaz.com/2017/04/01/reentrantlock/)。本文以Collections.UnmodifiableMap为例讲解，Collections.UnmodifiableMap是一个非常典型的代理场景，Collections.UnmodifiableMap返回Map的一个不可更改的视图，所以它需要禁止Map中“增删改”的函数，直接代理“查”的函数。Collections.UnmodifiableMap部分源码：

**构造函数**：

```
UnmodifiableMap(Map<? extends K, ? extends V> m) {
        if (m==null)
            throw new NullPointerException();
        this.m = m;
}
传入一个被代理的类。
```
**部分函数实现**：

```
public int size()                        {return m.size();}
public boolean isEmpty()                 {return m.isEmpty();}
public boolean containsKey(Object key)   {return m.containsKey(key);}
public boolean containsValue(Object val) {return m.containsValue(val);}
public V get(Object key)                 {return m.get(key);}

public V put(K key, V value) {
    throw new UnsupportedOperationException();
}
public V remove(Object key) {
    throw new UnsupportedOperationException();
}
public void putAll(Map<? extends K, ? extends V> m) {
    throw new UnsupportedOperationException();
}
public void clear() {
    throw new UnsupportedOperationException();
}
```

* size、isEmpty、containsKey、containsValue、get都属于"查"类的函数，所以直接代理即可。
* put、remove、putAll、clear都属于"增删改"类的函数，通过直接抛出异常而禁止访问。


## 享元模式

* **目的**：复用内存中已经存在的对象，降低创建对象的开销。
* **实现方式**：使用缓存机制。
* **tips**：说到享元模式就不得不说***Immutable Class***，Immutable Class指对象一旦创建后就不会发生改变的类，这种类由于天然是线程安全的而越来越受到欢迎，Scala中默认的类是Immutable Class。JDK中很多类是Immutable Class，例如，String，Integer等。由于这样的类不可变，所以当它的值需要改变时，一般会创建新的对象，比如String的+操作。为了减少系统重复创建大量重复对象，就用到了享元模式。
* **JDK实例**：以Integer为例介绍。Integer有一个内部类IntegerCache，默认缓存了[-128,127]所有的Integer。当开发人员调用Integer.valueOf就会从查看IntegerCache，返回一个已经存在的，或者新创建一个对象。
Integer.valueOf源码：

```
public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
```

## 观察者(发布-订阅)模式*(回调机制)*

* **目的**：当被观察者状态发生变化时，主动调用观察者定义的行为
* **实现方式**：被观察者维护一个观察者列表。观察者一般有一个统一的父类，父类中定义类观察者的接口，观察者继承该父类并实现该接口，将自己加入被观察者的观察者列表。
* **tips**：观察者模式在很多地方被用到，典型的比如在UI系统中，给Button添加一个Listener，点击Button，回调Listener。
* **JDK实例**：

## 适配器模式

* **目的**：
* **实现方式**：
* **tips**：
* **JDK实例**：

## 责任链模式

* **目的**：
* **实现方式**：
* **tips**：
* **JDK实例**：

## 单例模式

* **目的**：
* **实现方式**：
* **tips**：
* **JDK实例**：

## 工厂模式

* **目的**：
* **实现方式**：
* **tips**：
* **JDK实例**：

## 引用
IO部分类图来自：[https://www.ibm.com/developerworks/cn/java/j-lo-javaio/](https://www.ibm.com/developerworks/cn/java/j-lo-javaio/)