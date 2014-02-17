---
layout: post
title: "StampedLock学习笔记"
description: "Java8之StampedLock学习"
category: Java8
tags: [concurrent]
---
{% include JB/setup %}

    public class StampedLock implements java.io.Serializable

[Java Doc 地址] (http://cr.openjdk.java.net/~chegar/8005697/ver.00/javadoc/StampedLock.html)

### 文档翻译

一个基于性能具有三种模式控制读/写的锁。`StampedLock`的状态包含一个版本和模式。获取锁定的方法返回一个表示和控制锁定状态访问的标志；这些方法中`try`开头的版本反而返回一个特殊值0代表获取锁定失败。锁释放和转换方法需要标志（stamps）作为参数，并且如果他们不匹配锁的状态的话会失败。三个模式如下：

- __写。__ `writeLock()`方法可能阻塞等待独占访问，它返回一个标志（stamp）可以被`unlockWrite(long)`方法用于释放锁。`tryWriteLock`方法的不定时和定时版本也提供这个状态（stamp）。当锁保持在写模式，没有读锁可以被获得，并且所有乐观读校验将失败。

- __读。__ `readLock()`方法可能阻塞等待非独占访问，它返回一个标志（stamp）可以被`unlockRead(long)`方法用于释放锁。`tryReadLock`方法的不定时和定时版本也提供了这个状态（stamp）。

- __乐观读。__ `tryOptimisticRead()`方法只要在锁当前不处于写模式的情况下就返回一个非零标志（stamp）。自得到一个标志（stamp）后如果锁没有被写模式获取，`validate(long)`方法会返回`true`。这种模式可以被看作是一个读锁的极弱的版本，可以随时被一个写线程打断。为短只读代码段使用乐观模式通常会减少争用，提高吞吐量。然而它的使用在本质上是脆弱的。乐观读的部分只当读取字段并且在使用`validation`以后将他们保持在本地变量。在乐观读模式下读取变量可能会更加不一致，所以只适用于当你足够熟悉数据表示去检查一致性并且反复调用`validate()`方法。例如，当第一次读取一个对象或数组引用，然后访问其字段，元素或方法之一，通常需要这样的步骤。

这个类还支持有条件地提供跨三种模式转换的方法。例如，`tryConvertToWriteLock(long)`方法尝试升级模式，返回一个有效的写标志（stamp），如果（1）已经处于写模式（2）在读模式并且没有其他读线程或者（3）在乐观读模式并且锁是可用的。这个方法的形式旨在帮助减少一些代码膨胀，否则发生基于重试（retry-based）的设计。

`StampedLocks`作为内部工具设计用于开发线程安全的组件。他们的使用依赖于数据，对象，和方法所保护的内部属性知识。他们不是可重入的，所以锁定块不应该调用其他未知可能试图重新获得锁的方法（虽然你可以传递一个标志（stamp）给其他方法，标志可以使用或转换）。读锁模式的使用依赖于相关代码无副作用。未验证的乐观读部分不能调用不知道能否容忍潜在的不一致的方法（*指方法自己不知道是否能够容忍潜在的不一致*）。

`StampedLock`的调度策略并不总是喜欢读者（相比写者），反之亦然。所有`try`方法都尽力而为并且不一定遵守其他任何调度或者公平策略。任何`try`方法获取或者转换锁返回的0都不携带任何关于锁状态的信息；随后调用可能会成功。

因为它支持协调使用多个锁模式，这个类没有直接实现`Lock`或`ReadWriteLock`接口。然而，`StampedLock`在应用程序中只需要一组相关的功能，就可以被视为`asReadLock()`，`asWriteLock()`，`asReadWriteLock()`。

__使用示例。__ 下面举例说明在维护一个简单的二维类（Point）中的一些使用习惯。示例代码演示了一些`try/catch`约定，即使他们不是必须的因为没有异常会在他们里面发生。

    class Point {
       private double x, y;
       private final StampedLock sl = new StampedLock();

       void move(double deltaX, double deltaY) { // an exclusively locked method
         long stamp = sl.writeLock();
         try {
           x += deltaX;
           y += deltaY;
         } finally {
           sl.unlockWrite(stamp);
         }
       }

       double distanceFromOriginV1() { // A read-only method
         long stamp;
         if ((stamp = sl.tryOptimisticRead()) != 0L) { // optimistic
           double currentX = x;
           double currentY = y;
           if (sl.validate(stamp))
             return Math.sqrt(currentX * currentX + currentY * currentY);
         }
         stamp = sl.readLock(); // fall back to read lock
         try {
           double currentX = x;
           double currentY = y;
             return Math.sqrt(currentX * currentX + currentY * currentY);
         } finally {
           sl.unlockRead(stamp);
         }
       }

       double distanceFromOriginV2() { // combines code paths
         double currentX = 0.0, currentY = 0.0;
         for (long stamp = sl.tryOptimisticRead(); ; stamp = sl.readLock()) {
           try {
             currentX = x;
             currentY = y;
           } finally {
             if (sl.tryConvertToOptimisticRead(stamp) != 0L) // unlock or validate
               break;
           }
         }
         return Math.sqrt(currentX * currentX + currentY * currentY);
       }

       void moveIfAtOrigin(double newX, double newY) { // upgrade
         // Could instead start with optimistic, not read mode
         long stamp = sl.readLock();
         try {
           while (x == 0.0 && y == 0.0) {
             long ws = sl.tryConvertToWriteLock(stamp);
             if (ws != 0L) {
               stamp = ws;
               x = newX;
               y = newY;
               break;
             }
             else {
               sl.unlockRead(stamp);
               stamp = sl.writeLock();
             }
           }
         } finally {
           sl.unlock(stamp);
         }
       }
    }

### 内部实现

---

#### 算法笔记

该设计采用了元素的顺序锁（用于linux内核；见[Lameter's](http://www.lameter.com/gelato2005.pdf)以及其他；见[Boehm's](http://www.hpl.hp.com/techreports/2012/HPL-2012-68.html)）以及有序的读写锁（见[Shirako](http://dl.acm.org/citation.cfm?id=2312015)）  

从概念上讲，锁的主状态包括一个序列号，是奇数时，写锁定，偶数相反。然而这是一个读锁锁定时的非零读计数器值的偏移量。当验证乐观读锁的标志（stamp）时，读计数器被忽略。由于我们必须给读线程使用一个有限数量的位（JDK 7），当读线程的数量超过了计数字段时，会使用一个补充的读线程溢出单词。为此，我们处理最大读线程的值（RBITS）作为自旋锁保护更新溢出。
 
写线程采用一个在`AbstractQueuedSynchronizer`中使用的`CLH`锁的改进形式（在它内部文档查看详情），每个节点被标记（字段模式）为不是读线程就是写线程。等待的写线程集合被分组（链接的形式）分配在一个通用的节点下（字段`cowait`）因此作为一个关于大多数`CLH`技术的单独节点。凭借队列结构的优点，等待节点无需携带序列号；我们知道每个节点都比他前节点高。这种简化了调度策略的FIFO模式，包含`Phase-Fair`锁的元素（见 Brandenburg & Anderson, 特别是http://www.cs.unc.edu/~bbb/diss/）。特别是，我们使用`phase-fair` `anti-barging`规则，当读锁被保持但是存在一个等待的写线程时，如果一个读线程到达，这个读线程需要排队。（这个规则对于一些方法的复杂读获取`acquireRead`是可靠的，但是没有它，锁会变得非常不公平。）方法释放本身不会（有时不能）让读线程们`cowaiters`醒来。这是通过主线程做的，因为其他线程无法让方法`acquireRead`和`acquireWrite`做的更好。

这些规则适用于线程排队。所有`tryLock`形式的方法投机地尝试获取锁而不管偏好规则，所以可能“闯入”（*barge*）他们的入口。在获取（`acquire`）方法采用随机自旋的方式减少（越来越贵）上线文切换，同时也避免多个线程之间的持续内存颠簸。我们限制在队列头部自旋。一个线程在阻塞之前自旋的次数由`SPINS`次数（其中每个迭代有50%的概率减小自旋计数器）决定。如果它醒来以后未能获得锁，并且仍然是（或成为）第一个等待线程（这表明某个其他线程闯入并且获得锁），它升级自旋次数（直到`MAX_HEAD_SPINS`）以减少驳运线程不断失败的可能性。

几乎所有这些机制都在方法`acquireWrite`和`acquireRead`实现了，作为典型的代码扩展，因为动作（actions）和重试（retries）依赖于一致的本地缓存读取。

正如Boehm的[论文](http://www.hpl.hp.com/techreports/2012/HPL-2012-68.html)，序列验证（主要方法`validate()`）相比常规的`volatile`读（`state`变量）要求更严格的排序规则。我们使用`Unsafe.loadFence`方法强制读线程在`validation`方法以及`validation`方法本身还没有强制排序的情况下排序。

内存设计保持锁定状态和队列指针在一起（通常在同一缓存行）。这通常可以很好地用于读为主的负载。在大多数其他情况，自适应自旋锁（`adaptive-spin`）CLH的自然倾向是为了减少内存争用，减少更进一步展开竞争位置的积极性，但是可能是属于未来的改进。
 
 
-未完待续