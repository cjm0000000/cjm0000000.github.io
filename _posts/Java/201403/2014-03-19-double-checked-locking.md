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

#### Is it worth the trouble?

For most applications, the cost of simply making the getHelper() method synchronized is not high. You should only consider this kind of detailed optimizations if you know that it is causing a substantial overhead for an application.

Very often, more high level cleverness, such as using the builtin mergesort rather than handling exchange sort (see the SPECJVM DB benchmark) will have much more impact.

#### Making it work for static singletons

If the singleton you are creating is static (i.e., there will only be one Helper created), as opposed to a property of another object (e.g., there will be one Helper for each Foo object, there is a simple and elegant solution.

Just define the singleton as a static field in a separate class. The semantics of Java guarantee that the field will not be initialized until the field is referenced, and that any thread which accesses the field will see all of the writes resulting from initializing that field.
<?prettify linenums=1?>
    class HelperSingleton {
      static Helper singleton = new Helper();
    }

#### It will work for 32-bit primitive values

Although the double-checked locking idiom cannot be used for references to objects, it can work for 32-bit primitive values (e.g., int's or float's). Note that it does not work for long's or double's, since unsynchronized reads/writes of 64-bit primitives are not guaranteed to be atomic.
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
    
In fact, assuming that the computeHashCode function always returned the same result and had no side effects (i.e., idempotent), you could even get rid of all of the synchronization.
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

#### Making it work with explicit memory barriers

It is possible to make the double checked locking pattern work if you have explicit memory barrier instructions. For example, if you are programming in C++, you can use the code from Doug Schmidt et al.'s book:
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

### Fixing Double-Checked Locking using Thread Local Storage

Alexander Terekhov (TEREKHOV@de.ibm.com) came up clever suggestion for implementing double checked locking using thread local storage. Each thread keeps a thread local flag to determine whether that thread has done the required synchronization.
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

The performance of this technique depends quite a bit on which JDK implementation you have. In Sun's 1.2 implementation, ThreadLocal's were very slow. They are significantly faster in 1.3, and are expected to be faster still in 1.4. [Doug Lea analyzed the performance of some techniques for implementing lazy initialization](http://www.cs.umd.edu/~pugh/java/memoryModel/DCL-performance.html).

### Under the new Java Memory Model

As of JDK5, [there is a new Java Memory Model and Thread specification](http://www.cs.umd.edu/~pugh/java/memoryModel).

#### Fixing Double-Checked Locking using Volatile

JDK5 and later extends the semantics for volatile so that the system will not allow a write of a volatile to be reordered with respect to any previous read or write, and a read of a volatile cannot be reordered with respect to any following read or write. See [this entry in Jeremy Manson's blog](http://jeremymanson.blogspot.com/2008/05/double-checked-locking.html) for more details.

With this change, the Double-Checked Locking idiom can be made to work by declaring the helper field to be volatile. This does not work under JDK4 and earlier.
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

#### Double-Checked Locking Immutable Objects

If Helper is an immutable object, such that all of the fields of Helper are final, then double-checked locking will work without having to use volatile fields. The idea is that a reference to an immutable object (such as a String or an Integer) should behave in much the same way as an int or float; reading and writing references to immutable objects are atomic.

### Descriptions of double-check idiom

- [Reality Check](http://www.cs.wustl.edu/~schmidt/editorial-3.html), Douglas C. Schmidt, C++ Report, SIGS, Vol. 8, No. 3, March 1996.
- [Double-Checked Locking: An Optimization Pattern for Efficiently Initializing and Accessing Thread-safe Objects](http://www.cs.wustl.edu/~schmidt/DC-Locking.ps.gz), Douglas Schmidt and Tim Harrison. 3rd annual Pattern Languages of Program Design conference, 1996
- [Lazy instantiation](http://www.javaworld.com/javaworld/javatips/jw-javatip67.html), Philip Bishop and Nigel Warren, JavaWorld Magazine
- [Programming Java threads in the real world, Part 7](http://www.javaworld.com/javaworld/jw-04-1999/jw-04-toolbox-3.html), Allen Holub, Javaworld Magazine, April 1999.
- [Java 2 Performance and Idiom Guide](http://www.phptr.com/ptrbooks/ptr_0130142603.html), Craig Larman and Rhett Guthrie, p100.
- [Java in Practice: Design Styles and Idioms for Effective Java](http://www.google.com/search?q=Java+Design+Styles+Nigel+Bishop), Nigel Warren and Philip Bishop, p142.
- Rule 99, [The Elements of Java Style](http://www.google.com/search?q=elements+java+style+Ambler), Allan Vermeulen, Scott Ambler, Greg Bumgardner, Eldon Metz, Trvor Misfeldt, Jim Shur, Patrick Thompson, SIGS Reference library
- [Global Variables in Java with the Singleton Pattern](http://gamelan.earthweb.com/journal/techfocus/022300_singleton.html), Wiebe de Jong, Gamelan