---
layout: post
title: "Java内存模型和同步"
description: ""
category: Java
tags: [JMM]
---
{% include JB/setup %}

本文译自[Synchronization and the Java Memory Model](http://gee.cs.oswego.edu/dl/cpj/jmm.html)，作者*Doug Lea*。

*这组从2.2节摘录的内容包括关于Java内存模型如何影响并发编程的主要讨论。*

*有关内存模型正在进行的工作信息，参见 [Bill Pugh's Java Memory Model pages](http://www.cs.umd.edu/~pugh/java/memoryModel/).*

考虑这个微小的类，没有任何同步定义：
<?prettify linenums=1?>
    final class SetCheck {
      private int  a = 0;
      private long b = 0;

      void set() {
        a =  1;
        b = -1;
      }

      boolean check() {
        return ((b ==  0) || (b == -1 && a == 1)); 
      }
    }

在一个纯粹有序的语言，`check`方法可能永远不会返回`false`。这是成立的，即使编译器，运行时系统和硬件可能通过某种你无法直观地想到的途径处理这个代码。例如，以下任何一种可能适用于方法的执行集合：

- 编译器可能重新排列语句的顺序，所以`b`可以在`a`之前被分配。如果该方法被内联，编译器可能会进一步相对于另一些语句重新排列。

- 处理器也可以重新排列对应于语句的机器指令的执行顺序，或者甚至同时执行它们。

- The memory system (as governed by cache control units) may rearrange the order in which writes are committed to memory cells corresponding to the variables. These writes may overlap with other computations and memory actions.

- The compiler, processor, and/or memory system may interleave the machine-level effects of the two statements. For example on a 32-bit machine, the high-order word of b may be written first, followed by the write to a, followed by the write to the low-order word of b.

- The compiler, processor, and/or memory system may cause the memory cells representing the variables not to be updated until sometime after (if ever) a subsequent check is called, but instead to maintain the corresponding values (for example in CPU registers) in such a way that the code still has the intended effect.

In a sequential language, none of this can matter so long as program execution obeys as-if-serial semantics. Sequential programs cannot depend on the internal processing details of statements within simple code blocks, so they are free to be manipulated in all these ways. This provides essential flexibility for compilers and machines. Exploitation of such opportunities (via pipelined superscalar CPUs, multilevel caches, load/store balancing, interprocedural register allocation, and so on) is responsible for a significant amount of the massive improvements in execution speed seen in computing over the past decade. The as-if-serial property of these manipulations shields sequential programmers from needing to know if or how they take place. Programmers who never create their own threads are almost never impacted by these issues.

Things are different in concurrent programming. Here, it is entirely possible for check to be called in one thread while set is being executed in another, in which case the check might be "spying" on the optimized execution of set. And if any of the above manipulations occur, it is possible for check to return false. For example, as detailed below, check could read a value for the long b that is neither 0 nor -1, but instead a half-written in-between value. Also, out-of-order execution of the statements in set may cause check to read b as -1 but then read a as still 0.

In other words, not only may concurrent executions be interleaved, but they may also be reordered and otherwise manipulated in an optimized form that bears little resemblance to their source code. As compiler and run-time technology matures and multiprocessors become more prevalent, such phenomena become more common. They can lead to surprising results for programmers with backgrounds in sequential programming (in other words, just about all programmers) who have never been exposed to the underlying execution properties of allegedly sequential code. This can be the source of subtle concurrent programming errors.

In almost all cases, there is an obvious, simple way to avoid contemplation of all the complexities arising in concurrent programs due to optimized execution mechanics: Use synchronization. For example, if both methods in class SetCheck are declared as synchronized, then you can be sure that no internal processing details can affect the intended outcome of this code.

But sometimes you cannot or do not want to use synchronization. Or perhaps you must reason about someone else's code that does not use it. In these cases you must rely on the minimal guarantees about resulting semantics spelled out by the Java Memory Model. This model allows the kinds of manipulations listed above, but bounds their potential effects on execution semantics and additionally points to some techniques programmers can use to control some aspects of these semantics (most of which are discussed in �2.4).

The Java Memory Model is part of The JavaTM Language Specification, described primarily in JLS chapter 17. Here, we discuss only the basic motivation, properties, and programming consequences of the model. The treatment here reflects a few clarifications and updates that are missing from the first edition of JLS.

The assumptions underlying the model can be viewed as an idealization of a standard SMP machine of the sort described in �1.2.4:

