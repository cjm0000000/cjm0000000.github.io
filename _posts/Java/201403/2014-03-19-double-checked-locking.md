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

##### 不起作用的修复

鉴于上面的解释，一些人曾建议如下代码：

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

这段代码把构建`Helper`对象放在内部同步块内。这里直观地想法是，在同步释放点应该有一个内存屏障，并且应该防止`Helper`对象初始化和分配到`helper`字段的重排序。

不幸的是，直觉是完全错误的。同步规则不按这种方式工作的。对于`monitorexit`(比如释放同步)的规则是在`monitorexit`之前的操作必须在`monitor`释放之前执行。但是没有规则说`monitorexit`之后的操作不可以在`monitor`释放之前完成。编译器移动任务`helper = h;`到同步块内部是完全合理和合法的，在这种情况下，我们又回到了之前。许多处理器提供执行这种单向内存屏障的指令。改变语义要求释放锁是一个完整的内存屏障会有性能损失。

##### 更多不起作用的修复

你可以做一些事情去强制写入（操作）执行一个完整地双向内存屏障。这是严重，效率低下的，而且一旦`Java`内存模型被修改这几乎肯定无法正常工作的。不要使用这个。在科学的利益，[I've put a description of this technique on a separate page](http://www.cs.umd.edu/~pugh/java/memoryModel/BidirectionalMemoryBarrier.html)。不要用它。

*然而*，即使通过初始化`helper`对象的线程执行一个完整的内存屏障，它仍然无法正常工作。

问题是在某些系统上，看到`helper`字段非空值的线程仍旧需要执行内存屏障。

为什么？因为处理器有内存在本地缓存的副本。在某些处理器上，除非该处理器执行一个高速缓存一致性的指令（例如，一个内存屏障），否则会读到陈旧的本地缓存副本，即使其他处理器使用内存屏障强制他们写入全局内存。

我已经创建了[一个单独的网页](http://www.cs.umd.edu/~pugh/java/memoryModel/AlphaReordering.html)讨论在一个Alpha处理器这如何能够实际发生。

#### 是否得不偿失？

对于大多数的应用来说，简单的使`getHelper()`方法同步的成本并不高。你应该考虑这种详细的优化仅在如果你知道这在一个应用中造成大量开销。

很多时候，更高层次的小聪明，比如使用内置的归并排序而不是处理交换排序（见`SPECJVM DB`基准）将有更多的影响。

#### 使它适合静态单例

如果你创建的单例是静态的（比如，也只会创建一个`Helper`），相对于另一个对象的属性（比如，每个`Foo`对象将只有一个`Helper`），有一个简单而优雅的解决方案。

只要在一个单独的类定义`singleton`为一个静态字段。`Java`的语义保证该字段不会被初始化，直到字段被引用，并且访问该字段的所有线程都会看到所有初始化那个字段造成的写入。

<?prettify linenums=1?>
    class HelperSingleton {
      static Helper singleton = new Helper();
    }

#### 这将适用于32位的原始值

虽然双重锁定检查习语不能用于对对象的引用，它可以为32位基本类型值工作（例如，`int`的或者`float`的）。需要注意的是`long`或者`double`不起作用，因为非同步读/写64位基本类型不能保证原子性。

<?prettify linenums=1?>
    // Correct Double-Checked Locking for 32-bit primitives
    class Foo { 
      private int cachedHashCode = 0;
      public int hashCode() {
        int h = cachedHashCode;
        if (h == 0) 
        synchronized(this) {
          if (cachedHashCode != 0) return cachedHashCode;
          h = computeHashCode();
          cachedHashCode = h;
        }
        return h;
      }
      // other functions and members...
    }

事实上，假设`computeHashCode`函数总是返回相同的结果并且不会产生副作用（即幂等）,你甚至可以去掉所有同步。

<?prettify linenums=1?>
    // Lazy initialization 32-bit primitives
    // Thread-safe if computeHashCode is idempotent
    class Foo { 
      private int cachedHashCode = 0;
      public int hashCode() {
        int h = cachedHashCode;
        if (h == 0) {
          h = computeHashCode();
          cachedHashCode = h;
        }
        return h;
      }
      // other functions and members...
    }

#### 通过显示内存栅栏使其工作

如果你有明确的内存屏障指令，使双重检查锁定模式工作是可能的。例如，如果你用`C++`编程，你可以使用来自道格·施密特等人书中的代码：

<?prettify linenums=1?>
    // C++ implementation with explicit memory barriers
    // Should work on any platform, including DEC Alphas
    // From "Patterns for Concurrent and Distributed Objects",
    // by Doug Schmidt
    template <class TYPE, class LOCK> TYPE *
    Singleton<TYPE, LOCK>::instance (void) {
        // First check
        TYPE* tmp = instance_;
        // Insert the CPU-specific memory barrier instruction
        // to synchronize the cache lines on multi-processor.
        asm ("memoryBarrier");
        if (tmp == 0) {
            // Ensure serialization (guard
            // constructor acquires lock_).
            Guard<LOCK> guard (lock_);
            // Double check.
            tmp = instance_;
            if (tmp == 0) {
                tmp = new TYPE;
                // Insert the CPU-specific memory barrier instruction
                // to synchronize the cache lines on multi-processor.
                asm ("memoryBarrier");
                instance_ = tmp;
            }
        return tmp;
    }

### 使用线程本地存储修复双重检查锁定

Alexander Terekhov (TEREKHOV@de.ibm.com)提出了巧妙的建议，使用线程本地存储实现双重检查锁定。每个线程都保持一个线程的本地标志来确定线程是否做了必要的同步。

<?prettify linenums=1?>
    class Foo {
	 /** If perThreadInstance.get() returns a non-null value, this thread
		has done synchronization needed to see initialization
		of helper */
         private final ThreadLocal perThreadInstance = new ThreadLocal();
         private Helper helper = null;
         public Helper getHelper() {
             if (perThreadInstance.get() == null) createHelper();
             return helper;
         }
         private final void createHelper() {
             synchronized(this) {
                 if (helper == null)
                     helper = new Helper();
             }
	     // Any non-null value would do as the argument here
             perThreadInstance.set(perThreadInstance);
         }
	}

这种技术的性能相当多取决于你的`JDK`实现。`ThreadLocal`在Sun的`1.2`实现里是很慢的。`1.3`中显著变快，并预期`1.4`中还要更快。[Doug Lea analyzed the performance of some techniques for implementing lazy initialization](http://www.cs.umd.edu/~pugh/java/memoryModel/DCL-performance.html)。

### 在新的Java内存模型

如同`JDK5`，[有一个新的Java内存模型和线程规格](http://www.cs.umd.edu/~pugh/java/memoryModel)。

#### 使用`volatile`修复双重检查锁定

`JDK5`和更高版本扩展了`volatile`的语义，从而使系统将不允许`volatile`写相对于之前的任何读或写被重新排序，并且`volatile`读不能相对于任何随后的读或写重新排序。参见[Jeremy Manson的博客中该条目](http://jeremymanson.blogspot.com/2008/05/double-checked-locking.html)了解更多详情。

随着这种变化，通过声明`helper`字段为`volatile`，双重检查锁定习语可以工作。这在`JDK4`或者更早版本下不能工作。

<?prettify linenums=1?>
    // Works with acquire/release semantics for volatile
    // Broken under current semantics for volatile
    class Foo {
        private volatile Helper helper = null;
        public Helper getHelper() {
            if (helper == null) {
                synchronized(this) {
                    if (helper == null)
                        helper = new Helper();
                }
            }
            return helper;
        }
    }

#### 双检锁不可变对象

如果`Helper`是一个不可变对象，使得`Helper`的所有字段都是`final`的，那么双重检测锁定将会工作而不必使用`volatile`字段。这个想法是，引用一个不可变的对象（比如一个`String`对象或一个`Integer`对象）应该表现出和`int`或者`float`几乎相同的方式；读写不可变对象的引用是原子的。

### 双重检测习语的描述

- [Reality Check](http://www.cs.wustl.edu/~schmidt/editorial-3.html), Douglas C. Schmidt, C++ Report, SIGS, Vol. 8, No. 3, March 1996.
- [Double-Checked Locking: An Optimization Pattern for Efficiently Initializing and Accessing Thread-safe Objects](http://www.cs.wustl.edu/~schmidt/DC-Locking.ps.gz), Douglas Schmidt and Tim Harrison. 3rd annual Pattern Languages of Program Design conference, 1996
- [Lazy instantiation](http://www.javaworld.com/javaworld/javatips/jw-javatip67.html), Philip Bishop and Nigel Warren, JavaWorld Magazine
- [Programming Java threads in the real world, Part 7](http://www.javaworld.com/javaworld/jw-04-1999/jw-04-toolbox-3.html), Allen Holub, Javaworld Magazine, April 1999.
- [Java 2 Performance and Idiom Guide](http://www.phptr.com/ptrbooks/ptr_0130142603.html), Craig Larman and Rhett Guthrie, p100.
- [Java in Practice: Design Styles and Idioms for Effective Java](http://www.google.com/search?q=Java+Design+Styles+Nigel+Bishop), Nigel Warren and Philip Bishop, p142.
- Rule 99, [The Elements of Java Style](http://www.google.com/search?q=elements+java+style+Ambler), Allan Vermeulen, Scott Ambler, Greg Bumgardner, Eldon Metz, Trvor Misfeldt, Jim Shur, Patrick Thompson, SIGS Reference library
- [Global Variables in Java with the Singleton Pattern](http://gamelan.earthweb.com/journal/techfocus/022300_singleton.html), Wiebe de Jong, Gamelan