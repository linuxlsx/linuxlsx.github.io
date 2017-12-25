---
layout: post
title: "synchronized 作用于局部变量"
date: 2015-12-10 20:36:22 +0800
comments: true
categories: [java, synchronized, local variable]
---

`synchronized` 是java 中元老级的锁。他的使用非常的简单, 可以作用在普通方法，静态方法以及方法块上。对象是 `synchronized` 实现同步的基础，java 中每个对象都可以作为锁。具体表现为以下三种方式:

    1. 对于普通方法, 锁是当前对象的实例
    2. 对于静态同步方法, 锁是当前类的 Class 对象
    3. 对于同步方法块, 锁是 synchronized 括号里面配置的对象

<!-- more -->

今天代码review 的时候, 发现一个新奇的写法，简化后的代码如下:

    public static Object get(String key){

      String syncKey = key + "something else";

      synchronized (syncKey){
          // 简化的逻辑
          // obj = map.get(syncKey)
          // if obj == null
          //      obj = Provider.get()
          //      map.put(obj)
          // return obj
      }
    }

咋一看这跟我们常规的写法不一样，常规的写法 一般是将 `syncKey` 定义为类的一个属性, 这样能够保证多线程确实是基于同一个对象实例来做同步的。对于这个写法作者的解释是 java 对于String 是做过特殊处理的，值一样的String 是会做缓存的。所以多线程访问的情况下仍然是作用在同一个对象上, 可以达到同步的效果。

对于这个结论是一半对一半不对, 具体能不能达到想要的同步效果，就要看是不是作用在同一个对象实例上。那String 到底是不是同一个对象实例呢？ 那就要从 String 的内存模型说起了。具体详解可以参考[深入理解Java：String](http://www.cnblogs.com/ITtangtang/p/3976820.html)

所以针对这个示例，其实是不对的。 `String syncKey = key + "something else"` 每次都返回一个 `new String()` 对于 `synchronized` 来说肯定不是同一个对象实例, 所以就达不到同步的效果。但是如果修改成如下的就是可以的:


    public static Object get(String key){

      String syncKey = "something else";

      synchronized (syncKey){
          // 简化的逻辑
          // obj = map.get(syncKey)
          // if obj == null
          //      obj = Provider.get()
          //      map.put(obj)
          // return obj
      }
    }

`String syncKey = "something else";` 这样的方式是将 `something else` 这个字符串保存到了JVM 的字符串常量池中, 并且可以共享。所以多线程访问的时候拿到的都是同一个对象, 就能达到同步的效果。

Java 还有没有能够达到这种效果的对象吗？答案是肯定的, 以下的几种情况得到的都是同一个对象实例:

    1. byte(Byte), short(Short), int(Integer), long(Long) 值在[-128,127] 之间
    2. boolean (true or false)
    3. char 值在 ['\u0000' - '\u007f']

这个是Java 语言规范中定义的。这些值会在JVM中缓存下来, 每次使用都会返回同一个实例。但是仅限`Integer i = 1` 这样的方式. 如果是 `Integer i = new Integer(1)` 的话， 那就是不同的对象了。

#### 结论

`synchronized` 将锁标记保存在对象头中, 所以只要对象是同一个就能够达到同步的效果，也不一定要是实例变量。而Java 语言中某些对象实例虽然是局部变量, 但是由于一些性能优化上的考虑, 会对String, Byte,Short,Boolean,Integer,Long 类型的数据做一些优化。

#### 参考资料

1. [深入理解Java: String](http://www.cnblogs.com/ITtangtang/p/3976820.html)
2. [String.intern in Java 6, 7 and 8 – string pooling](http://java-performance.info/string-intern-in-java-6-7-8/)
3. [深入理解Java String#intern() 内存模型](http://codelog.me/2015/01/20/2015-01-20-string-intern/)
