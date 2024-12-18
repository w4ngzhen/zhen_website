---
layout: post
title: Java中的Integer
date: 2017-08-22
tags: 
- Java
- Integer
categories: 
- 技术
---

<!-- more -->

众所周知，在Java中，存在着值比较与应用比较两种情况。例如，如下的比较，可以根据值比较与引用比较来跟容易的判断出结果来：

```java
int a = 123;
int b = 123;
String s1 = new String("123");
String s2 = new String("123");
System.out.println(a == b); //true
System.out.println(s1 == s2); //false
```
这里，a与b由于是基本类型，所以Java在比较的时候直接就是按值来比较，而下面的s1与s2则是由于分别指向内容为“123”的字符串对象引用（关于string的细节，见本人的另一篇文章），而这两个字符串的地址并不一样，所以结果是false。

那么，今天要讨论的是，对于Java自动拆装箱的问题的深入探讨。如下所示，请问结果是什么呢？
```java
Integer a = 666;
Integer b = 666;
System.out.println(a == b);
```

结果是false，您可能会说，这有什么好问的，Integer对象的比较，引用的比较，而这两个只是值相同，而对象不同的Integer对象罢了，所以当然为false。好，那么我再问你，下面的结果是什么？
```java
Integer a = 100;
Integer b = 100;
System.out.println(a == b);
```
您可能说，哇，当我傻吗，当然还是false了。可是，结果是true。

为什么同样的情况下，当值变小了，结果就变为true了呢。

其实，Java中，对于可装箱的对象类型，都存在一个1字节的范围：-128到127。在这个范围类的数字，Java认为是常用的数字，所以自动进行了值比较，而不是进行引用的比较。所以，无论是Long还是Integer，只有你的值在-128到127，这两个对象的比较直接按照其所存储的值来进行。就像如下的情况：
```java
Integer a = 128;
Integer b = 128;
Integer c = 127;
Integer d = 127;
Long e = -129L;
Long f = -129L;
Long g = -128L;
Long h = -128L;
System.out.println(a == b); //false
System.out.println(c == d); //true
System.out.println(e == f); //false
System.out.println(g == h); //true
```
