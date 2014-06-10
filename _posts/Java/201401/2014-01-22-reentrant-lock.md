---
layout: post
title: "ReentrantLock学习笔记"
description: "Java 多线程 ReentrantLock学习笔记"
category: Java
tags: [concurrent]
---
{% include JB/setup %}
<?prettify ?>
    public class ReentrantLock extends Object implements Lock, Serializable
	
[Java Doc 地址] (http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/locks/ReentrantLock.html)

## 文档翻译

一个可重入的互斥锁，具有和`synchronized`修饰的方法和同步块相同的基本行为和隐式监视器锁语义，但是具备扩展功能。

一个`ReentrantLock`由最后成功获取锁但尚未解锁的线程拥有。当锁没有被其他线程占有的时候调用`lock`的线程成功获取锁并且返回，如果当前线程已经占有锁此方法将立即返回。可以使用`isHeldByCurrentThread()`和`getHoldCount()`来检测这种情况。

该类的构造器接受一个可选的公平参数。当设置为`true`，在竞争之下，倾向于让等待时间最长的线程获取锁。否则此锁不保证任何特定的访问顺序。多线程访问公平锁相比默认设置（_非公平锁_）可能表现出较低的总吞吐量（即，速度较慢，往往要慢很多），但是在获取锁在时候有更小的差异而且不会造成线程饥饿。但是请注意，锁的公平性不能保证线程调度的公平性。因此，当其他活动线程没有运行并且当前没有保持锁的时候，使用公平锁的线程中的一个可能会连续多次获得锁。还要注意的是不限期的`tryLock`方法不遵循公平设置。如果锁可用，它会成功，即使其他线程正在等待。


这是推荐的做法：在调用锁之后总是紧跟一个`try`块，最典型的是下面的前/后结构：

<?prettify linenums=1?>
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

`ReentrantLock`内部是基于`AbstractQueuedSynchronizer`实现的，不同于`ReentrantReadWriteLock`，它仅仅实现了AQS的排他模式，它有一个可选的公平策略。处于公平模式下的线程，获取一把锁之前需要查询当前线程之前是否有其他线程在等待锁，如果有，则当前线程无法立即获取到锁。

### 非公平模式

这种模式锁的操作相对简单一些。

##### 获取锁

<?prettify linenums=1?>
    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }

可以看出这里获取锁是通过委托`nonfairTryAcquire`方法来实现的：

<?prettify linenums=1?>
    final boolean nonfairTryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            if (compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0) // overflow
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }

从代码可以看出获取一个非公平的锁只可能是两种情况之一：
1. `getState()`获取到的状态是0，可以通过CAS来更改状态，需要注意的是：如果成功更改了状态，那么设置排他线程为当前线程，返回`true`。
2. 在第一种情况没满足的情况下，如果当前线程就是排他线程，那么恭喜你，你可以获取锁（重入获取，需要更新状态计数器），除非状态计数器值溢出了，那么会抛出`Error`。
3. 其他情况都无法获取到锁，返回`false`。

##### 释放锁

<?prettify linenums=1?>
    protected final boolean tryRelease(int releases) {
        int c = getState() - releases;
        if (Thread.currentThread() != getExclusiveOwnerThread())
            throw new IllegalMonitorStateException();
        boolean free = false;
        if (c == 0) {
            free = true;
            setExclusiveOwnerThread(null);
        }
        setState(c);
        return free;
    }

释放锁的时候需要注意两点(首先得更新状态计数器 --- _有点像废话_)：
1. 需要判断当前线程是否是排他线程，如果不是则需要抛出`IllegalMonitorStateException`异常。
2. 需要检查状态计数器值，如果等于0，则代表完全释放这把锁，需要设置排他线程为`null`并且返回`true`(非完全释放的情况则返回`false`)。

### 公平模式

这个模式下获取锁的逻辑相对复杂一些。

##### 获取锁

<?prettify linenums=1?>
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }

和非公平模式下获取锁的方式大体相同，唯一的一个区别是当状态计数器值等于0的时候，需要通过方法`hasQueuedPredecessors`来检测当前线程之前是否有其他线程在等待获取锁，如果有的话，当前线程将无法获取到锁，并且返回`false`。
    
##### 释放锁

公平模式下释放锁的逻辑和非公平模式下的一模一样。

`ReentrantLock`有一个需要注意的方法，那就是`tryLock`( _上文也已经提及过_ )，在设置了公平策略的情况下，你不要期望这个方法遵守公平策略，因为：

<?prettify linenums=1?>
    public boolean tryLock() {
        return sync.nonfairTryAcquire(1);
    }
    
它内部hard code了调用`nonfairTryAcquire`方法。`tryLock(long timeout, TimeUnit unit)`这个方法则遵守公平策略。

全文完-