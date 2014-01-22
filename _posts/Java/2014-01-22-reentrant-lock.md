---
layout: post
title: "ReentrantLock学习笔记"
description: "Java 多线程 ReentrantLock学习笔记"
category: Java
tags: [concurrent]
---
{% include JB/setup %}

    public class ReentrantLock extends Object implements Lock, Serializable
	
[Java Doc 地址] (http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/locks/ReentrantLock.html)

## 文档翻译

一个可重入的互斥锁，具有和`synchronized`修饰的方法和同步块相同的基本行为和隐式监视器锁语义，但是具备扩展功能。

一个`ReentrantLock`由最后成功获取锁但尚未解锁的线程拥有。当锁没有被其他线程占有的时候调用`lock`的线程成功获取锁并且返回，如果当前线程已经占有锁此方法将立即返回。可以使用`isHeldByCurrentThread()`和`getHoldCount()`来检测这种情况。

该类的构造器接受一个可选的公平参数。当设置为`true`，在竞争之下，倾向于让等待时间最长的线程获取锁。否则此锁不保证任何特定的访问顺序。多线程访问公平锁相比默认设置（_非公平锁_）可能表现出较低的总吞吐量（即，速度较慢，往往要慢很多），但是在获取锁在时候有更小的差异而且不会造成线程饥饿。但是请注意，锁的公平性不能保证线程调度的公平性。因此，当其他活动线程没有运行并且当前没有保持锁的时候，使用公平锁的线程中的一个可能会连续多次获得锁。还要注意的是不限期的`tryLock`方法不遵循公平设置。如果锁可用，它会成功，即使其他线程正在等待。


这是推荐的做法：在调用锁之后总是紧跟一个`try`块，最典型的是下面的前/后结构：

    class X {
	   private final ReentrantLock lock = new ReentrantLock();
	   // ...

	   public void m() {
		 lock.lock();  // block until condition holds
		 try {
		   // ... method body
		 } finally {
		   lock.unlock()
		 }
	   }
	 }

除了实现`Lock`接口的方法，这个类定义方法`isLocked`和`getLockQueueLength`，以及一些对植入和监控有用的相关的`protected`方法。

此类和内置锁有相同的序列化行为：不管序列化的时候锁处于什么状态，反序列化后的锁处于非锁定状态。

此锁支持被同一个线程递归锁定最多2147483647次。尝试超过这个限制会导致锁定方法抛出`Error`。

## 内部实现

