---
layout: post
title: "Java内存模型"
description: ""
category: Java
tags: [JLS]
---
{% include JB/setup %}

本文翻译自Java Language Specification(SE 7)的第17章，[原文地址](http://docs.oracle.com/javase/specs/jls/se7/html/jls-17.html#jls-17.4)。

### 17.4. 内存模型

*内存模型* 描述，给出一个程序和该程序的执行跟踪，不论执行跟踪是否是一个合法的执行程序。Java编程语言的内存模型通过在执行跟踪检查每个读入和按照一定的规则检查读观察到的写是有效的。

内存模型描述了一个程序可能的行为。一个实现是可以自由地生产它喜欢的任何代码，只要所有生成的执行程序产生的结果可以被内存模型预测。

*这为实现者提供了很大的自由去执行大量的代码转换，包括指令重排序和去除不必要的同步。*

#### 例 17.4-1. 错误的同步可能表现出奇怪的行为

*Java编程语言的语义允许编译器和微处理器执行优化，优化错误的同步代码可能产生自相矛盾的行为。以下是错误的同步可能表现出奇怪的行为的一些例子。*

*考虑，例如表17.1所示的示例程序的跟踪。这个程序使用局部变量 `r1` 和 `r2` 和共享变量 `A` 和 `B` 。最初，`A == B == 0`。*

#### 表 17.1 语句重排序造成的令人惊讶的结果 - 原始代码

<table class="table table-striped table-bordered" style="width:50%">
    <tr><th>Thread 1</th><th>Thread 2</th></tr>
    <tr><td>1: r2 = A;</td><td>3: r1 = B;</td></tr>
    <tr><td>2: B = 1;</td><td>4: A = 2;</td></tr>
</table>

*它不可能出现结果： `r2 == 2 并且 r1 == 1` 。直观地说，非指令1先执行即指令3先执行。如果指令1先到达，它不能够看到指令4（对变量A）的写。如果指令3先到达，它不能够看到指令2（对变量B ）的写。*

*如果一些执行表现出这种行为，那么我们将知道指令4先于指令1，指令1先于指令2，指令2先于指令3，指令3先于指令4。事实上，这是荒谬的。*

*然而，当不影响线程隔离执行时，在任何线程中，编译器都被允许对指令重新排序。如果指令1和指令2重排序，如表17.2所示的跟踪，那么很容易看到可能的运行结果 `r2 == 2 并且 r1 == 1` 。*

#### 表 17.2 语句重排序造成的出人意外的结果 － 有效的编译器转换

<table class="table table-striped table-bordered" style="width:50%">
    <tr><th>Thread 1</th><th>Thread 2</th></tr>
    <tr><td>B = 1;</td><td>r1 = B;</td></tr>
    <tr><td>r2 = A;</td><td>A = 2;</td></tr>
</table>

*对于一些程序员，这个行为可能看起来“破碎的”。然而，应该指出的是，这个代码是错误的同步：*

- *一个线程写变量*

- *另一个线程读同一个变量*

- *并且写入和读取没有用同步排序。*

*这种情况是数据争用的一个例子（[§17.4.5](http://docs.oracle.com/javase/specs/jls/se7/html/jls-17.html#jls-17.4.5)）。当代码包含数据争用，往往可能有违反直觉的结果。*

 *在表17.2中有几种机制可以产生重排序。Java虚拟机实现中的即时编译器（`JIT`）可能重新排列代码或处理器。此外，Java虚拟机实现的内存层次结构可能使其看起来好像正在重新排列代码。在这一章，作为编译器，我们应该参考凡是能够重排序的代码。*

*意外结果的另一个例子可以在表17.3看到。最初， `p == q && p.x == 0` 。这个程序也是不正确的同步；其写入共享内存并没有在这些写入操作之间强制排序。*

#### 表17.3 前向替换造成的出人意外的结果

<table class="table table-striped table-bordered" style="width:50%">
    <tr><th>Thread 1</th><th>Thread 2</th></tr>
    <tr><td>r1 = p;</td><td>r6 = p;</td></tr>
    <tr><td>r2 = r1.x;</td><td>r6.x = 3;</td></tr>
    <tr><td>r3 = q;	</td><td></td></tr>
    <tr><td>r4 = r3.x;</td><td></td></tr>
    <tr><td>r5 = r1.x;</td><td></td></tr>
</table>

*一个常见的编译器优化涉及到读取到的 `r2` 的值重新用于 `r5` 。它们都是在没有写干预的情况下读取 `r1.x`。这个情况如表17.4所示。*

#### 表17.4 前向替换造成的出人意外的结果

<table class="table table-striped table-bordered" style="width:50%">
    <tr><th>Thread 1</th><th>Thread 2</th></tr>
    <tr><td>r1 = p;</td><td>r6 = p;</td></tr>
    <tr><td>r2 = r1.x;</td><td>r6.x = 3;</td></tr>
    <tr><td>r3 = q;	</td><td></td></tr>
    <tr><td>r4 = r3.x;</td><td></td></tr>
    <tr><td>r5 = r2;</td><td></td></tr>
</table>

*现在考虑这个情况：在 `Thread 1` 第一次读取 `r1.x` 和读取 `r3.x` 期间，在 `Thread 2` 中给 `r6.x` 赋值。如果编译器决定重用 `r2` 的值给 `r5`，那么 `r2` 和 `r5` 的值将为0，并且 `r4` 的值将为3。从程序员的角度，`p.x` 的值从0变为3，然后又变回0。*

内存模型决定在程序的每个点哪些值可以被读取。每个孤立的线程的活动必须表现为被那个线程的语义管理，除每个线程看到的值是由内存模型决定的之外。我们说程序服从`intra-thread`语义。`Intra-thread`语义是单线程程序的语义，并且允许根据线程中读取动作看到的值完整预测一个线程的行为。为了决定线程t执行的动作是否合法，我们简单地计算线程t以它将在单线程上下文中执行的实现，如在本说明书的其他部分的定义。

每次`t`线程的计算产生一个跨线程动作，它必须与程序指令中紧随其后的`t`线程的跨线程动作`a`相匹配。如果`a`是一个读指令，那么`a`看到的`t`线程进一步计算使用的值由内存模型决定。

本节提供了Java编程语言的内存模型规范，除了处理`final`字段的问题，这在[§17.5](http://docs.oracle.com/javase/specs/jls/se7/html/jls-17.html#jls-17.5)中描述。

*此处指定的内存模型没有从根本上基于Java编程语言的面向对象本质。为了我们的例子简洁和简单，我们通常展示的代码片段不包含类或方法定义，或显式非关联。大多数例子的语句包含由两个或更多线程访问局部变量，共享全局变量或者对象实例字段。我们通常使用 `r1` 或 `r2` 这样的名称来表示方法或者线程的局部变量。这样的变量不能被其他线程访问。*

#### 17.4.1. 共享变量

可以被线程之间共享的内存叫做*共享内存* 或者*堆内存*。

所有实例字段，静态字段和数组元素都存在堆内存。在本章中，我们把字段和数组元素都用术语变量来表示。

本地变量（[§14.4](http://docs.oracle.com/javase/specs/jls/se7/html/jls-14.html#jls-14.4)），方法形参（[§8.4.1](http://docs.oracle.com/javase/specs/jls/se7/html/jls-8.html#jls-8.4.1)）和异常处理参数（[§14.20](http://docs.oracle.com/javase/specs/jls/se7/html/jls-14.html#jls-14.20)）从未在线程间共享，并且不受内存模型影响。

如果至少有一个线程是写入访问，那么两个线程访问（读取或写入）同一个变量据说是冲突的。

#### 17.4.2. 操作

一个线程间操作是一个线程执行的操作可以被检测或者被另一个线程直接影响。一个程序可以执行很多不同的跨线程操作：

- 读（常规的，或非`volatile`）。读取变量。

- 写（常规的，或非`volatile`）。写入变量。

- 同步操作，也就是：
    
    - Volatile读。一个变量的`volatile`读取。
    
    - Volatile写。一个变量的`volatile`写入。
    
    - Lock。锁定监视器。
    
    - Unlock。解锁监视器。
    
    - 一个线程的（`synthetic`）第一个和最后一个动作。
    
    - 启动一个线程或者检测一个线程已终止的动作([§17.4.4](http://docs.oracle.com/javase/specs/jls/se7/html/jls-17.html#jls-17.4.4))。

- 外部操作。一个外部操作是一个可以被执行外部观察到的动作，并且有一个以执行之外的环境为基础的结果。

- 线程分支操作（[§17.4.9](http://docs.oracle.com/javase/specs/jls/se7/html/jls-17.html#jls-17.4.9)）。一个线程分支是只由一个线程无限循环执行，在其中没有内存（*笔者感觉是共享内存*），同步，或者外部操作执行。如果一个线程执行一个线程分支操作，其后将跟随无限数量的线程分支操作。

    *线程分支操作被引入到模型——如何让一个线程可能造成其他所有线程拖延并且不能取得进展*

本规范只涉及跨线程操作。我们不需要关心自己与跨线程操作（比如，两个局部变量求和并且把结果存到第三个局部变量）。正如前面提到的，所有线程需要服从Java程序的正确的线程内语义。我们通常会把跨线程操作简单地称作简单*操作*。

一个操作由一个元组*< t, k, v, u >*描述，包含：

- `t` - 执行操作的线程

- `k` - 操作的种类

- `v` - 操作相关的变量或者监视器

    对于锁定操作，*v*是正在被锁定的监视器；对于解锁操作，*v*是正在被解锁的监视器。
    
    如果是一个读（volatile或非volatile）操作，*v*是一个正在被读的变量。
    
    如果是一个写（volatile或非volatile）操作，*v*是一个正在被写的变量。

- `u` - 对于操作的任意唯一标示符

一个外部操作元组包含附加组件，它包含了线程执行操作时觉察到的外部操作的结果。这可以是关于操作成功或者失败的信息，以及操作读取的其它任何值。

外部操作的参数（比如，写入任何套接字的任何字节）不是外部操作元组的一部分。这些参数都是被线程中的其它操作设置的并且可以通过检查线程内语义确定。他们在内存模型中没有明确探讨过。

在非中断执行中，不是所有的外部操作都能被觉察到。非中断执行和可观察操作在[§17.4.9](http://docs.oracle.com/javase/specs/jls/se7/html/jls-17.html#jls-17.4.9)中讨论。

#### 17.4.3. 程序和程序顺序

在每个线程t执行的所有跨线程操作中，`t`的程序顺序是一个总的顺序，它反映了这些操作将根据`t`的线程内语义执行的顺序。

如果所有操作的发生在一个全序（执行顺序），一组操作的顺序是一致的，这与程序顺序一致，并且更进一步，每个对变量`v`的读`r`都看到了`w`对`v`写入的值：

- `w` 执行顺序在`r`之前，并且

- 没有其他`w`写入，如`w1`在`w2`之前，并且`w2`在`r`之前，这样的执行顺序。

顺序一致性是由程序执行相关的可见性和执行顺序来保证的。在一个一致的顺序执行中，在个别的操作（比如读和写）之上有一个总的顺序，这与程序的执行顺序一致，并且每个操作都是原子的并且对每个线程都立即可见。

如果程序没有数据竞争，那么程序所有执行将显示为顺序一致。

数据竞争的顺序一致性和/或自由度仍然允许一组操作产生错误，这需要（或不需要）以原子的方式感知。

*如果我们把顺序一致性当作内存模型，很多我们讨论的编译器和处理器优化将是非法的。例如，在表17.3的跟踪，一旦把 3写入 `p.x`，随后该位置的读取将要看到该值。*

#### 17.4.4. 同步顺序

每次执行都有一个*同步顺序*。一个同步顺序是一个执行中所有同步操作的总的顺序。对于每个线程`t`，`t`的同步操作([§17.4.2](http://docs.oracle.com/javase/specs/jls/se7/html/jls-17.html#jls-17.4.2))的同步顺序与`t`的程序顺序([§17.4.3](http://docs.oracle.com/javase/specs/jls/se7/html/jls-17.html#jls-17.4.3))是一致的。

同步操作在操作上引起*synchronized-with*关系，定义如下：

- 监视器`m`上的解锁操作和后续的锁定操作同步（这里的“后续”是根据同步顺序定义的）。

- 写入一个`volatile`变量`v`([§8.3.1.4](http://docs.oracle.com/javase/specs/jls/se7/html/jls-8.html#jls-8.3.1.4))与任何线程的所有后续对`v`的读取同步（这里的“后续”是根据同步顺序定义的）。

- 启动一个线程的操作与线程启动的第一个操作同步。

- 每个变量默认值（`0`，`false`或者`null`）的写入与每个线程的第一个操作同步。

    尽管在包含变量的对象被分配之前给变量写入一个默认值可能看起来有点奇怪，概念上讲，每个对象在程序开始的时候用默认初始值被创建。

- `T1`线程的`final`操作与另一个发现`T1`已经终止的`T2`线程的任何操作同步。

    `T2`可以通过调用`T1.isAlive()`或者`T1.join()`做到这一点。

如果`T1`线程中断`T2`线程，`T1`产生的中断与任何点的任何发现T2已经被中断的其它线程（包括T2）同步（通过让其抛出`InterruptedException`异常，或者通过调用`Thread.interrupted`或`Thread.isInterrupted`方法）。

*synchronizes-with*边缘的来源被称作释放，而目的地被称作获取。

#### 17.4.5. Happens-before 顺序

两个操作可以被一个*happens-before*关系排序。如果一个操作发生在另一个操作之前，那么第一个是可见的并且在第二个之前。

如果我们有`x`和`y`两个操作，我们用`hb(x, y)`来表示`x`发生在`y`之前。

- 如果`x`和`y`操作是相同线程里的并且在程序顺序中`x`在`y`之前，那么`hb(x, y)`。

- 从一个对象的构造器的末尾到那个对象垃圾回收（`finalizer`）的开始，有一个*happens-before*边缘。

- 如果一个`x`操作和一个随后的`y`操作同步，那么我们也有`hb(x, y)`。

- 如果`hb(x, y)`并且`hb(y, z)`，那么`hb(x, z)`。

`Object`类([§17.2.1](http://docs.oracle.com/javase/specs/jls/se7/html/jls-17.html#jls-17.2.1))的`wait`系列方法有与之相关的锁定和解锁操作；他们的*happens-before*关系由这些相关联的动作定义。

It should be noted that the presence of a happens-before relationship between two actions does not necessarily imply that they have to take place in that order in an implementation. If the reordering produces results consistent with a legal execution, it is not illegal.

*For example, the write of a default value to every field of an object constructed by a thread need not happen before the beginning of that thread, as long as no read ever observes that fact.*

More specifically, if two actions share a happens-before relationship, they do not necessarily have to appear to have happened in that order to any code with which they do not share a happens-before relationship. Writes in one thread that are in a data race with reads in another thread may, for example, appear to occur out of order to those reads.

The *happens-before* relation defines when data races take place.

A set of synchronization edges, S, is sufficient if it is the minimal set such that the transitive closure of S with the program order determines all of the happens-before edges in the execution. This set is unique.

It follows from the above definitions that:

- An unlock on a monitor happens-before every subsequent lock on that monitor.

- A write to a volatile field ([§8.3.1.4](http://docs.oracle.com/javase/specs/jls/se7/html/jls-8.html#jls-8.3.1.4)) happens-before every subsequent read of that field.

- A call to start() on a thread happens-before any actions in the started thread.

- All actions in a thread happen-before any other thread successfully returns from a join() on that thread.

- The default initialization of any object happens-before any other actions (other than default-writes) of a program.

When a program contains two conflicting accesses ([§17.4.1](http://docs.oracle.com/javase/specs/jls/se7/html/jls-17.html#jls-17.4.1)) that are not ordered by a happens-before relationship, it is said to contain a data race.

The semantics of operations other than inter-thread actions, such as reads of array lengths ([§10.7](http://docs.oracle.com/javase/specs/jls/se7/html/jls-10.html#jls-10.7)), executions of checked casts ([§5.5](http://docs.oracle.com/javase/specs/jls/se7/html/jls-5.html#jls-5.5), [§15.16](http://docs.oracle.com/javase/specs/jls/se7/html/jls-15.html#jls-15.16)), and invocations of virtual methods ([§15.12](http://docs.oracle.com/javase/specs/jls/se7/html/jls-15.html#jls-15.12)), are not directly affected by data races.

*Therefore, a data race cannot cause incorrect behavior such as returning the wrong length for an array.*

A program is correctly synchronized if and only if all sequentially consistent executions are free of data races.

If a program is correctly synchronized, then all executions of the program will appear to be sequentially consistent ([§17.4.3](http://docs.oracle.com/javase/specs/jls/se7/html/jls-17.html#jls-17.4.3)).

*This is an extremely strong guarantee for programmers. Programmers do not need to reason about reorderings to determine that their code contains data races. Therefore they do not need to reason about reorderings when determining whether their code is correctly synchronized. Once the determination that the code is correctly synchronized is made, the programmer does not need to worry that reorderings will affect his or her code.*

*A program must be correctly synchronized to avoid the kinds of counterintuitive behaviors that can be observed when code is reordered. The use of correct synchronization does not ensure that the overall behavior of a program is correct. However, its use does allow a programmer to reason about the possible behaviors of a program in a simple way; the behavior of a correctly synchronized program is much less dependent on possible reorderings. Without correct synchronization, very strange, confusing and counterintuitive behaviors are possible.*

We say that a read r of a variable v is allowed to observe a write w to v if, in the happens-before partial order of the execution trace:

- r is not ordered before w (i.e., it is not the case that hb(r, w)), and

- there is no intervening write w' to v (i.e. no write w' to v such that hb(w, w') and hb(w', r)).

Informally, a read r is allowed to see the result of a write w if there is no happens-before ordering to prevent that read.

A set of actions A is happens-before consistent if for all reads r in A, where W(r) is the write action seen by r, it is not the case that either hb(r, W(r)) or that there exists a write w in A such that w.v = r.v and hb(W(r), w) and hb(w, r).

In a happens-before consistent set of actions, each read sees a write that it is allowed to see by the happens-before ordering.

##### Example 17.4.5-1. Happens-before Consistency

For the trace in Table 17.5, initially A == B == 0. The trace can observe r2 == 0 and r1 == 0 and still be happens-before consistent, since there are execution orders that allow each read to see the appropriate write.

##### Table 17.5. Behavior allowed by happens-before consistency, but not sequential consistency.

<table class="table table-striped table-bordered" style="width:50%">
    <tr><th>Thread 1</th><th>Thread 2</th></tr>
    <tr><td>B = 1;</td><td>A = 2;</td></tr>
    <tr><td>r2 = A;</td><td>r1 = B;</td></tr>
</table>

*Since there is no synchronization, each read can see either the write of the initial value or the write by the other thread. An execution order that displays this behavior is:*
<?prettify linenums=1?>
    B = 1;
    A = 2;
    r2 = A;  // sees initial write of 0
    r1 = B;  // sees initial write of 0
    
*Another execution order that is happens-before consistent is:*
<?prettify linenums=1?>
    r2 = A;  // sees write of A = 2
    r1 = B;  // sees write of B = 1
    B = 1;
    A = 2;
    
*In this execution, the reads see writes that occur later in the execution order. This may seem counterintuitive, but is allowed by happens-before consistency. Allowing reads to see later writes can sometimes produce unacceptable behaviors.*
