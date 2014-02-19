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

当多线程更新一个用于通用的求和，例如收集统计信息，而不是细粒度的同步控制，这个类通常比`AtomicLong`类表现更好。在低争用下更新，这两个类具有相似的特性。但是在高并发下，这个类的预期吞吐量会显著提高，它以更高的空间消耗为代价。

`LongAdders`可以与`ConcurrentHashMap`一起使用，维护一个可频繁扩展的`map`（`histogram`或` multiset`的形式）。例如，为`ConcurrentHashMap<String,LongAdder> freqs`增加一个计数器，如果不存在则初始化，你可以使用代码`freqs.computeIfAbsent(k -> new LongAdder()).increment();`。

这个类继承于`Number`类，但是并没有定义例如`equals`，`hashCode`和`compareTo`之类的方法，因为实例预期将会突变，所以他们作为集合的键是没有用的。
