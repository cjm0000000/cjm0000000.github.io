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

在一个纯粹有序的语言，`check`方法可能永远不会返回`false`。这是成立的，即使编译器，运行时系统和硬件可能通过某种你无法直观地想到的途径处理这个代码。例如，以下任何一种可能适用于`set`方法的执行：

- 编译器可能重新排列语句的顺序，所以`b`可以在`a`之前被分配。如果该方法被内联，编译器可能会进一步相对于另一些语句重新排列。

- 处理器也可以重新排列对应于语句的机器指令的执行顺序，或者甚至同时执行它们。

- 内存系统可以重新排列提交到变量对应的内存单元的写入顺序。这些写入操作可能会和其他的计算或内存操作重叠。

- 编译器，处理器和/或内存系统可能交织地在机器级别影响两个语句。例如一个32位机器上，`b`的高位字可以先写，然后写入`a`，然后写`b`的低位字。

- 编译器，处理器和/或内存系统可能导致变量对应的内存单元不被更新，直到一段时间后（如果有的话）调用后续的检查，而是保持相应的值（例如在`CPU`的寄存器）这样一种方式，该代码任然有预期的效果。

在顺序语言，这一切都没问题，只要程序的执行服从`as-if-serial`语义。顺序执行的程序不能依赖于简单代码块语句的内部处理细节，所以他们可以自由地在所有这些方式进行操作。这为编译器和机器提供了必不可少的灵活性。利用这样的机会（通过流水线超标量处理器，多级高速缓存，加载/存储平衡，过程间寄存器分配等等）是可靠的，对于在过去十年的计算中看到的执行速度的大量的巨大改进。这些操作的`as-if-serial`属性屏蔽顺序程序员不需要知道它们如何发生。没有创建自己的线程的程序员几乎从未被这些问题影响。

在并发编程里情况就不同了。在这里，完全可能一个线程调用`check`方法的同时另一个线程正在执行`set`方法，在这种情况下，`check`方法可能在优化的执行集合被“侦察”到。如果发生上述任何操作，`check`方法可能返回`false`。例如，详情如下，`check`方法可以读取一个长整形`b`的值，它既不是0也不是-1，而是一个写了一半的*中间值*。此外，`set`方法中语句的乱序执行可能导致`check`方法读取`b`的值是-1，但是读到的`a`的值仍旧是0。

In other words, not only may concurrent executions be interleaved, but they may also be reordered and otherwise manipulated in an optimized form that bears little resemblance to their source code. As compiler and run-time technology matures and multiprocessors become more prevalent, such phenomena become more common. They can lead to surprising results for programmers with backgrounds in sequential programming (in other words, just about all programmers) who have never been exposed to the underlying execution properties of allegedly sequential code. This can be the source of subtle concurrent programming errors.

换句话说，不仅可以被交错地并发执行，但他们也可能被重排序，并且以优化的形式操作，不像他们的源代码。由于编译器和运行时技术的成熟和多处理器变得越来越普遍，这种现象变得越来越普遍。对于在顺序编程背景下的程序员（换句话说，几乎所有的程序员），谁从来没有接触过据称顺序代码的底层执行属性，他们可能会导致意想不到的结果。这可能是细微的并发编程错误来源。

In almost all cases, there is an obvious, simple way to avoid contemplation of all the complexities arising in concurrent programs due to optimized execution mechanics: Use synchronization. For example, if both methods in class SetCheck are declared as synchronized, then you can be sure that no internal processing details can affect the intended outcome of this code.

But sometimes you cannot or do not want to use synchronization. Or perhaps you must reason about someone else's code that does not use it. In these cases you must rely on the minimal guarantees about resulting semantics spelled out by the Java Memory Model. This model allows the kinds of manipulations listed above, but bounds their potential effects on execution semantics and additionally points to some techniques programmers can use to control some aspects of these semantics (most of which are discussed in §2.4).

The Java Memory Model is part of The JavaTM Language Specification, described primarily in JLS chapter 17. Here, we discuss only the basic motivation, properties, and programming consequences of the model. The treatment here reflects a few clarifications and updates that are missing from the first edition of JLS.

The assumptions underlying the model can be viewed as an idealization of a standard SMP machine of the sort described in §1.2.4:

