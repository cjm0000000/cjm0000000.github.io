---
layout: post
title: "Java 8并发更新"
description: ""
category: Java
tags: [concurrent]
---
{% include JB/setup %}

[原文地址](http://openjdk.java.net/jeps/155)

### 概要

可扩展可更新的变量，为`ConcurrentHashMap`增强了面向缓存的API，改进了`ForkJoinPool`，和额外的`Lock`和`Future`类。

### 动机

并发和并行应用程序中的用途不断演变，需要在类库的支持上不断演变。这里描述的所有工作都是由使用`java.util.concurrent`包的用户的经验和建议促使。

### 描述

1. 可扩展的可更新的变量。维护一个单计数，求和等，可能被多个线程更新，这是一个常见的可扩展性问题。一小套新类（`DoubleAccumulator`，`DoubleAdder`，`LongAccumulator`，`LongAdder`）内部采用竞争减排（contention-reduction）技术，相比原子变量它提供了更加巨大的吞吐量。

2. 新增功能（并且可能添加API）使`ConcurrentHashMap`和从他们构造的类在高速缓存中更有用。这些方法包括计算键的值当他们不存在的时候，以及改进支持扫描和可能的驱逐项，以及更好地支持具有大量元素的map。

3. 为`ForkJoinPools`添加功能和提高性能，使他们被用户期望的那样，在越来越广泛的应用中更有效地使用。新功能包括支持基于完成的（completion-based）设计，往往是最适合IO-绑定（IO-bound）等用法。

可能会进一步增加额外的`Lock`和`Future`类（**笔者注释：截止翻译日期，已经增加了新的锁`StampedLock`，关于Java中几把锁的性能和使用场景可以参考[这篇文章](http://mechanical-sympathy.blogspot.com/2013/08/lock-based-vs-lock-free-concurrent.html)**），以及复议更好地构建STM（Software Transactional Memory）框架的相关支持。STM支持本身不是JDK 8的目标。

-全文完