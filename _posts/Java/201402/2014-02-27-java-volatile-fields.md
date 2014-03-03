---
layout: post
title: "java并发模型之volatile关键字"
description: ""
category: Java
tags: [concurrent]
---
{% include JB/setup %}

`volatile` 关键字在有些场景能提供比 `synchronized` 更好的性能。

先来看看[Java language specification(Java SE 7)](http://docs.oracle.com/javase/specs/jls/se7/html/index.html)上面的描述：

[原文地址](http://docs.oracle.com/javase/specs/jls/se7/html/jls-8.html#jls-8.3.1.4)


### 8.3.1.4. volatile字段

Java编程语言允许线程访问共享变量([§17.1](http://docs.oracle.com/javase/specs/jls/se7/html/jls-17.html#jls-17.1))。一般来说，为了确保共享变量一致且可靠地更新，一个线程需要确保它通过获得一把锁独占地使用这些变量，一般来说，强制互斥共享变量。

Java编程语言提供了第二种机制，`volatile` 字段，它在某些目的比锁定更方便。

一个字段可以被声明为 `volatile`，在这种情况下，`Java内存模型` 保证了所有的线程看到一致的变量值（[§17.4](http://docs.oracle.com/javase/specs/jls/se7/html/jls-17.html#jls-17.4)）。

*如果一个 `final` 的变量也被声明为 `volatile`，这是一个编译错误。*

#### 例子 8.3.1.4-1. volatile 字段

*如果在下面的例子中，一个线程反复调用 `one` 方法（但是总共调用次数不超过 `Integer.MAX_VALUE` ），而另一个线程反复调用 `two` 方法： *

    <?prettify?>
    class Test {
        static int i = 0, j = 0;
        static void one() { i++; j++; }
        static void two() {
            System.out.println("i=" + i + " j=" + j);
        }
    }

*然后 `two` 方法打印出的 `j` 的值偶尔会比 `i` 的值大，因为例子没有包含同步，而且根据在[§17.4](http://docs.oracle.com/javase/specs/jls/se7/html/jls-17.html#jls-17.4)的规则，共享变量 `i` 和 `j` 的值可能会乱序更新。*

*一种防止这种乱序执行行为的方式是把 `one` 方法和 `two` 方法声明为 `synchronized`（[§8.4.3.6](http://docs.oracle.com/javase/specs/jls/se7/html/jls-8.html#jls-8.4.3.6)）*
    <?prettify?>
    class Test {
        static int i = 0, j = 0;
        static synchronized void one() { i++; j++; }
        static synchronized void two() {
            System.out.println("i=" + i + " j=" + j);
        }
    }

*这可以防止 `one` 方法和 `two` 方法被并发执行，而且保证共享变量 `i` 和 `j` 的值都在 `one` 方法返回前被更新。因此 `two` 方法从来不会观察到 `j` 的值会比 `i` 的值大；事实上，它总是观察到 `i` 和 `j` 具有相同值。*

*另一种方法是把 `i` 和 `j` 声明为 `volatile`*

    <?prettify?>
    class Test {
        static volatile int i = 0, j = 0;
        static void one() { i++; j++; }
        static void two() {
            System.out.println("i=" + i + " j=" + j);
        }
    }

*这允许 `one` 方法和 `two` 方法并发地执行，这只是保证访问共享变量 `i` 和 `j` 的值正好出现相同的次数，并且以完全相同的顺序，因为他们似乎发生在执行程序文本的每个线程中。因此，共享变量 `j` 的值从来都不会比 `i` 的值大，因为在更新 `j` 之前，`i` 的每次更新都必须取得 `i` 的共享区域的值。这是可能的，然而，`two` 方法的任意调用都有可能使观察到`j`的值比观察到的 `i` 的值大，因为当 `two` 方法取 `i` 的值的瞬间和 `two` 方法取 `j` 的值的瞬间，`one` 方法可能被执行了多次。*

*见[§17.4](/Java/2014/03/03/java-memory-model/)的更多讨论和示例。*

