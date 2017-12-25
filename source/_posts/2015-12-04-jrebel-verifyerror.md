---
layout: post
title: "Jrebel VerifyError"
date: 2015-12-04 15:40:14 +0800
comments: true
categories: [java,jrebel]
---


Jrebel 动态部署极大提高了开发效率，是开发必备神器之一. 个人也使用了2年多了，一直都非常顺手.  但是今天启动应用的时候，报了如下的错误:


    Caused by: java.lang.VerifyError: Bad <init> method call from inside of a branch
    Exception Details:
    Location:
      com/taobao/healthcenter/web/servlet/StatusServlet.<init>(Lcom/zeroturnaround/javarebel/Ge;I[Ljava/lang/Object;)V @11: invokespecial
    Reason:
      Error exists in the bytecode
    Bytecode:
      0000000: 04b8 008e c600 0b2a 2b1c 2db7 0113 b112
      0000010: 771c b800 7dbf                         
    Stackmap Table:
      same_frame(@15)

      java version : 1.7.0_67
      jrebel version : 6.0.0


Google 发现Jrebel官网已经有人提出了这个问题: [VerifyError](http://zeroturnaround.com/forums/topic/verifyerror-bad-method-call-from-inside-of-a-branch/)

<!-- more -->

根据帖子中提到的[JDK Bug](https://bugs.openjdk.java.net/browse/JDK-8051012) 1.7版本最早修复该bug 的版本是 7u72.

根据帖子中提供的方案在JVM参数中添加 `-noverify` 解决问题。

##### Tip

我其实用 jdk7 和 jrebel 6 已经好久了，为什么今天就碰到这个问题了呢。 通过对比发现是: 之前我都是将 Java 的编译版本设置为了 1.6, 而今天出问题的时候我将编译版本设置成了 1.7, 而jdk7 相比 jdk6 生成的字节码引入了一些功能, 导致 jrebel 修改字节码后会出现一些验证的问题。

原本打算直接升级到 jdk8 的， 但是受限于公司规定的框架只能先升级到 jdk7. 看来还要等会才能在线上使用 lambda 了

#### 参考资料

1. [Java 7 Bytecode Verifier: Huge backward step for the JVM](http://chrononsystems.com/blog/java-7-design-flaw-leads-to-huge-backward-step-for-the-jvm)
2. [Use of -noverify when launching java apps](http://stackoverflow.com/questions/300639/use-of-noverify-when-launching-java-apps)
