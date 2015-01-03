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

换句话说，不仅可以被交错地并发执行，但他们也可能被重排序，并且以优化的形式操作，不像他们的源代码。由于编译器和运行时技术的成熟和多处理器变得越来越普遍，这种现象变得越来越普遍。对于在顺序编程背景下的程序员（换句话说，几乎所有的程序员），谁从来没有接触过据称顺序代码的底层执行属性，他们可能会导致意想不到的结果。这可能是细微的并发编程错误来源。

在几乎所有的情况下，有一个明显的，简单的方式避免在并发程序中由于优化执行机制带来的复杂性沉思：使用同步。例如，如果SetCheck类中的两个方法都声明为同步，那么你可以肯定没有内部处理细节可以影响这个代码的预期结果。

但有时你不能或不想使用同步。或者你必须推出别人的代码不使用它。在这种情况下，你必须依赖Java内存模型阐明的最小语义保证。这种模式允许多种上述操作，但限定他们对执行语义的潜在影响，并且还指出了程序员可以用于控制这些语义的某些方面的技术（其中大部分在§2.4中讨论）。

Java内存模型是Java&trade;语言规范的一部分，主要是在第17章JLS中描述。在这里，我们只讨论基本的动机，性质和模型的编程后果。这里的处理方式反映了一些澄清和从第一版的JLS缺少的更新。

该模型的基本假设可以被看作是在§1.2.4中描述的那种标准的SMP机器的理想化：

<img class="imgaligncenter" src="/images/synchronization-and-java-memory-model-1.gif" />

对模型的目的，每个线程可以被看作运行在与其它任何线程不同的CPU上。即使是在多处理器上，这实际上是罕见的，但事实证明这种每个线程映射到CPU是合法的方式去为一些模型的最初的令人惊奇的性质实现线程账户。例如，因为CPU持有的寄存器不能被其他CPU直接访问，该模型必须允许一个线程不知道被另一个线程操纵的值的情况。然而，该模型的影响绝非限于多处理器。即使在单CPU系统，编译器和处理器的操作可能导致相同的顾虑。

该模型并没有特别提及是否上面讨论的执行策略的种类是由编译器，处理器，缓存控制器或任何其它机制进行。它甚至没有讨论程序员熟悉的类，对象和方法。相反，该模型定义了线程和主内存之间的抽象关系。每个线程都被定义为具有一个工作内存（高速缓存和寄存器的抽象），在其中可以存储值。该模型保证了一些属性围绕对应于方法和内存单元对应的指令序列的相互作用。大多数规则的措辞中，当值必须与主内存和每个线程的工作内存之间传输的方面。该规则解决三个相互交织的问题：

- *原子性*  
指令必须有不可分割的效果。出于模型的目的，这些规则需要说明仅适用于简单的读取和表示字段的存储单元的写入 - 实例和静态变量，也包括数组元素，但不包括方法内部的局部变量。
    
- *可见性*  
在什么条件下一个线程的影响对另一个线程可见的。这里感兴趣的影响是写入字段，可以通过读取这些字段看到。
    
- *有序性*  
在什么条件下操作的影响对任何给定的线程可能出现乱序。主要排序问题围绕读写和赋值语句的顺序有关。

When synchronization is used consistently, each of these properties has a simple characterization: All changes made in one synchronized method or block are atomic and visible with respect to other synchronized methods and blocks employing the same lock, and processing of synchronized methods or blocks within any given thread is in program-specified order. Even though processing of statements within blocks may be out of order, this cannot matter to other threads employing synchronization.

When synchronization is not used or is used inconsistently, answers become more complex. The guarantees made by the memory model are weaker than most programmers intuitively expect, and are also weaker than those typically provided on any given JVM implementation. This imposes additional obligations on programmers attempting to ensure the object consistency relations that lie at the heart of exclusion practices: Objects must maintain invariants as seen by all threads that rely on them, not just by the thread performing any given state modification.

The most important rules and properties specified by the model are discussed below.
