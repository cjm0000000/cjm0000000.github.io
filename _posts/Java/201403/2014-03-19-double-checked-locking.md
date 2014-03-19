---
layout: post
title: "“双重检查锁被打破”声明"
description: ""
category: Java
tags: [JMM]
---
{% include JB/setup %}

本文译自[The "Double-Checked Locking is Broken" Declaration](http://www.cs.umd.edu/~pugh/java/memoryModel/DoubleCheckedLocking.html)。

双重检查锁被广泛引用，并且作为在*多线程环境*中实现延迟初始化的有效方法。

不幸的是，当用`Java`实现，而没有额外的同步时，它不会可靠地工作在一个独立于平台的方式。当用其他语言实现，比如`C++`，它依赖于处理器的内存模型，编译器执行的重排序以及编译器和同步类库之间的相互作用。因为这些都不在语言中指定，比如`C++`，很少可以说关于将在其中运行的情况。显示内存栅栏可用来使其在`C++`中工作，但是这些栅栏在`Java`中是无效的。

先解释一下所期望的行为，考虑下面的代码：
<?prettify linenums=1?>
    // Single threaded version
    class Foo { 
      private Helper helper = null;
      public Helper getHelper() {
        if (helper == null) 
            helper = new Helper();
        return helper;
        }
      // other functions and members...
    }

如果这个代码是在多线程上下文中使用的，许多事情可能出错。最明显的，两个或者更多`Helper`对象可以被分配。（我们稍后会带出其他问题）。修正这个问题仅仅需要同步`getHelper()`方法：

<?prettify linenums=1?>
    // Correct multithreaded version
    class Foo { 
      private Helper helper = null;
      public synchronized Helper getHelper() {
        if (helper == null) 
            helper = new Helper();
        return helper;
        }
      // other functions and members...
    }

上面的代码在每次调用`getHelper()`时执行同步。双重检查锁定习语试图在`helper`分配后避免同步：

<?prettify linenums=1?>
    // Broken multithreaded version
    // "Double-Checked Locking" idiom
    class Foo {
      private Helper helper = null;
      public Helper getHelper() {
        if (helper == null) 
          synchronized(this) {
            if (helper == null) 
              helper = new Helper();
          }    
        return helper;
        }
      // other functions and members...
    }

不幸的是，无论是最优化编译器或共享内存的多处理器的存在，这些代码都是行不通的：

### 它不起作用

有很多原因使它不起作用。我们将介绍的所有原因的第一个更明显。了解这些之后，你可能倾向于尝试想出一个方法“解决”双重检查锁定习语。你的修改将无法正常工作：还有更微妙的原因会造成你的修补程序将无法正常工作。了解了这些原因，想出了更好的解决，它仍然是行不通的，因为还有更微妙的理由。

很多聪明的人已经花了很多时间看这个（问题）。没有任何办法让它工作而不需要每个访问`helper`实例的线程执行同步。

#### 不起作用的第一个原因

它不起作用最明显的原因是初始化`Helper`对象的写线程和写`helper`字段的线程可以完成或觉察乱序。因此一个线程调用`getHelper()`方法可以看到一个`helper`对象的一个非空引用，但是看到`helper`对象字段的默认值，而不是构造器中设置的值。

如果编译器内联调用构造器，那么初始化对象的写线程和写入`helper`字段的线程可以自由地重排序如果编译器能够证明构造器不会抛出异常或者执行同步。

即使编译器不会重排序这些写入，在一个多处理器的系统，处理器或者内存系统可能会重排序这些写入，如同被运行在另一个处理器的线程感知。

道格.李已经写了[a more detailed description of compiler-based reorderings](http://gee.cs.oswego.edu/dl/cpj/jmm.html)。

##### 一个测试用例显示这是行不通的

Paul Jakubik找到一个使用双重检查锁定没有正常工作的例子[A slightly cleaned up version of that code is available here](http://www.cs.umd.edu/~pugh/java/memoryModel/DoubleCheckTest.java)。

当运行在一个使用`Symantec JIT`的系统，它不起作用。特别是，`Symantec JIT`把`singletons[i].reference = new Singleton();`编译成以下（注意，`Symantec JIT`使用基于句柄的对象分配系统）。

<?prettify linenums=1?>
    0206106A   mov      eax,0F97E78h
    0206106F   call     01F6B210                ; allocate space for
                                                ; Singleton, return result in eax
    02061074   mov      dword ptr [ebp],eax     ; EBP is &singletons[i].reference 
                                                ; store the unconstructed object here.
    02061077   mov      ecx,dword ptr [eax]     ; dereference the handle to
                                                ; get the raw pointer
    02061079   mov      dword ptr [ecx],100h    ; Next 4 lines are
    0206107F   mov      dword ptr [ecx+4],200h  ; Singleton's inlined constructor
    02061086   mov      dword ptr [ecx+8],400h
    0206108D   mov      dword ptr [ecx+0Ch],0F84030h

正如你可以看到，对`singletons[i].reference`的赋值比调用`Singleton`的构造器先执行。这在现有的`Java`内存模型下是完全合法的，并且在`C`和`C++`下也是合法的（因为他们都有一个内存模型）。

##### A fix that doesn't work

Given the explanation above, a number of people have suggested the following code:
<?prettify linenums=1?>
    // (Still) Broken multithreaded version
    // "Double-Checked Locking" idiom
    class Foo { 
      private Helper helper = null;
      public Helper getHelper() {
        if (helper == null) {
          Helper h;
          synchronized(this) {
            h = helper;
            if (h == null) 
                synchronized (this) {
                  h = new Helper();
                } // release inner synchronization lock
            helper = h;
            } 
          }    
        return helper;
        }
      // other functions and members...
    }
    
This code puts construction of the Helper object inside an inner synchronized block. The intuitive idea here is that there should be a memory barrier at the point where synchronization is released, and that should prevent the reordering of the initialization of the Helper object and the assignment to the field helper.

Unfortunately, that intuition is absolutely wrong. The rules for synchronization don't work that way. The rule for a monitorexit (i.e., releasing synchronization) is that actions before the monitorexit must be performed before the monitor is released. However, there is no rule which says that actions after the monitorexit may not be done before the monitor is released. It is perfectly reasonable and legal for the compiler to move the assignment helper = h; inside the synchronized block, in which case we are back where we were previously. Many processors offer instructions that perform this kind of one-way memory barrier. Changing the semantics to require releasing a lock to be a full memory barrier would have performance penalties.

##### More fixes that don't work

There is something you can do to force the writer to perform a full bidirectional memory barrier. This is gross, inefficient, and is almost guaranteed not to work once the Java Memory Model is revised. Do not use this. In the interests of science, [I've put a description of this technique on a separate page](http://www.cs.umd.edu/~pugh/java/memoryModel/BidirectionalMemoryBarrier.html). Do not use it.

*However*, even with a full memory barrier being performed by the thread that initializes the helper object, it still doesn't work.

The problem is that on some systems, the thread which sees a non-null value for the helper field also needs to perform memory barriers.

Why? Because processors have their own locally cached copies of memory. On some processors, unless the processor performs a cache coherence instruction (e.g., a memory barrier), reads can be performed out of stale locally cached copies, even if other processors used memory barriers to force their writes into global memory.

I've created [a separate web page](http://www.cs.umd.edu/~pugh/java/memoryModel/AlphaReordering.html) with a discussion of how this can actually happen on an Alpha processor.
