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

