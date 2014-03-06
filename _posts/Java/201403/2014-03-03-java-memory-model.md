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
    
- External Actions. An external action is an action that may be observable outside of an execution, and has a result based on an environment external to the execution.

- Thread divergence actions ([§17.4.9](http://docs.oracle.com/javase/specs/jls/se7/html/jls-17.html#jls-17.4.9)). A thread divergence action is only performed by a thread that is in an infinite loop in which no memory, synchronization, or external actions are performed. If a thread performs a thread divergence action, it will be followed by an infinite number of thread divergence actions.

*Thread divergence actions are introduced to model how a thread may cause all other threads to stall and fail to make progress.*


