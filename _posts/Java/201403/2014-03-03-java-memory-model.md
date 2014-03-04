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

*意外结果的另一个例子可以在表17.3看到。最初， `p == q 并且 p.x == 0` 。这个程序也是不正确的同步；其写入共享内存并没有在这些写入操作之间强制排序。*

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

#### Table 17.4. Surprising results caused by forward substitution