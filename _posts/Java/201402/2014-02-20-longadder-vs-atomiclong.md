---
layout: post
title: "【译】Java 8性能改进：LongAdder和AtomicLong对比"
description: ""
category: Java8
tags: [Performance]
---
{% include JB/setup %}

本文译自[Java 8 Performance Improvements: LongAdder vs AtomicLong](http://blog.palominolabs.com/?p=255)，以下是译文：

[Java 8](https://jdk8.java.net/)正在开发中，它为[JVM](http://en.wikipedia.org/wiki/Java_virtual_machine)上最广泛使用的语言带来了许多新特性。可能最常被提到的功能是`lambdas`，这让`Scala`和`JRuby`的爱好者将发布“最后”一声叹息。不那么华丽，但是对某些类别的多线程应用非常重要，是增加了[LongAdder](http://download.java.net/jdk8/docs/api/java/util/concurrent/atomic/LongAdder.html)和[DoubleAdder](http://download.java.net/jdk8/docs/api/java/util/concurrent/atomic/DoubleAdder.html)，原子序列的实现，它在多线程并发环境下相比[AtomicInteger](http://download.java.net/jdk8/docs/api/java/util/concurrent/atomic/AtomicInteger.html)和[AtomicLong](http://download.java.net/jdk8/docs/api/java/util/concurrent/atomic/AtomicLong.html)提供了更加卓越的性能。

一些简单的基准测试显示了两者之间的性能差异，以下基准测试我们使用了一个[m3.2xlarge EC2](http://aws.amazon.com/ec2/instance-types/instance-details/)实例，它提供了访问英特尔至强`E5-2670`的所有8个内核。
 
<img class="imgaligncenter" src="/images/long-adder-vs-atomic-long-1.png" />

用单线程的方式，新的`LongAdder`类会慢`1/3`，但是当多线程竞争着去增加字段的值的时候，`LongAdder`显示了它的价值。注意，每个线程做的唯一的事情就是试图增加计数器——这是一个最极端形式的合成基准。这里的竞争比你可能看到的大多数实际应用要高，但是有时你需要这种共享计数器，并且`LongAdder`类将是一个很大的帮助。

你可以在我们的[java-8-benchmarks](https://github.com/palominolabs/java-8-benchmarks)仓库找到这些基准测试的代码。它采用[JMH](http://openjdk.java.net/projects/code-tools/jmh/)去做所有的实际工作，用[马歇尔](http://blog.palominolabs.com/author/marshall/)的[gradle-jmh-demo](https://bitbucket.org/marshallpierce/gradle-jmh-demo)探究。`JMH`通过为你做所有繁琐的细节使得基准测试变得容易，确保所产生的数字代表目前工艺水平在基于JVM的基准测试的精度。然而`JMH`不适合运行性能测试，所以我们也有一些简单的[独立的](https://github.com/palominolabs/java-8-benchmarks/blob/master/standalone/src/main/java/com/palominolabs/benchmark/IncrementingBenchmark.java)基准给它（*性能测试*）。

#### 用perf-stat的更多细节

我们写了独立的基准测试程序，这样我们可以有更多的控制权，并且让他们运行在[perf-stat](https://perf.wiki.kernel.org/index.php/Tutorial)去获得一些更多的关于这到底是怎么回事的细节。最基本的是每个基准测试运行的挂钟时间。这些基准测试都是运行在一个英特尔酷睿`i7-2600 K`上（真正的硬件，不是虚拟化的）。

<img class="imgaligncenter" src="/images/long-adder-vs-atomic-long-2.png" />

虽然`AtomicLong`在单线程情况下更快，它迅速失去和`LongAdder`的优势，在两个线程的时候比`LongAdder`慢了将近`4`倍，线程数达到机器的核心数时慢了将近`5`倍。更令人印象深刻的是，`LongAdder`的表现是恒定的，直到线程的数量超过CPU的物理核心数量（在例4）。

#### 每个周期的指令数

<img class="imgaligncenter" src="/images/long-adder-vs-atomic-long-3.png" />

每个周期的指令数测量CPU有多少工作需要做。当它等待内存加载或者缓存一致性协议来解决。在这种情况下，我们看到`AtomicLong`在多线程环境下糟糕透顶的`IPC`。核心数从4到8的性能下降可能是因为这款CPU有4个核心，每个核心有两个硬件线程，并且硬件线程实际上没有帮助。

#### 空闲时间

处理器的执行管道分为两大组：前端部分，负责提取和解码操作，后端执行指令。在提取操作时，没有太多有趣的事情发生，所以让我们跳过前端处理。

<img class="imgaligncenter" src="/images/long-adder-vs-atomic-long-4.png" />

后端的活动提供了更多的关于是怎么回事的了解，上图显示了`AtomicLong`留下超过两倍多的空闲周期。`AtomicLong`的高空闲时间类似于每个CPU周期其拙劣的指令：`CPU`的核心花费了大量的时间去决定哪个核心控制包含`AtomicLong`的高速缓存行。


#### 参考资料

- [A more detailed look at LongAdder](http://minddotout.wordpress.com/2013/05/11/java-8-concurrency-longadder/)
- [Using JMH to Benchmark Multi-Threaded Code](http://psy-lob-saw.blogspot.com/2013/05/using-jmh-to-benchmark-multi-threaded.html)