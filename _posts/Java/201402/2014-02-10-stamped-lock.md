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