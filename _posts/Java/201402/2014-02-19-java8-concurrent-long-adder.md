---
layout: post
title: "Java 8 LongAdder 学习笔记"
description: ""
category: Java8
tags: [concurrent]
---
{% include JB/setup %}
<?prettify?>
    public class LongAdder extends Striped64 implements Serializable
    
[Java Doc](http://download.java.net/lambda/b78/docs/api/java/util/concurrent/atomic/LongAdder.html)

### 文档翻译

一个或多个变量，它们共同维持一个初始为零类型为`long`的总和。当跨线程竞争更新（`add(long)`方法）时，该组变量可以动态地扩展，以减少争用。`sum()`方法（或者等价的`longValue()`方法）返回当前整组变量保持的总和。

This class is usually preferable to AtomicLong when multiple threads update a common sum that is used for purposes such as collecting statistics, not for fine-grained synchronization control. Under low update contention, the two classes have similar characteristics. But under high contention, expected throughput of this class is significantly higher, at the expense of higher space consumption.

LongAdders can be used with a ConcurrentHashMap to maintain a scalable frequency map (a form of histogram or multiset). For example, to add a count to a ConcurrentHashMap<String,LongAdder> freqs, initializing if not already present, you can use freqs.computeIfAbsent(k -> new LongAdder()).increment();

This class extends Number, but does not define methods such as equals, hashCode and compareTo because instances are expected to be mutated, and so are not useful as collection keys.
