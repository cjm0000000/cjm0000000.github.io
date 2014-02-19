---
layout: post
title: "ReentrantReadWriteLock学习笔记"
description: ""
category: Java
tags: [concurrent]
---
{% include JB/setup %}
<?prettify?>
    public class ReentrantReadWriteLock extends Object implements ReadWriteLock, Serializable

[Java Doc 地址] (http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/locks/ReentrantReadWriteLock.html)

## 文档翻译

`ReentrantReadWriteLock`是`ReadWriteLock`接口的实现，提供类似`ReentrantLock`的语义（*作者觉得是指可重入*）

这个类具有有下面的一些属性:

- __获取锁的顺序__

    这个类不强制要求读线程或者写线程获得锁的顺序。但是它提供一个可选的公平策略。

 - __*非公平模式 (默认)*__  
    当用非公平模式构造`ReentrantReadWriteLock`(默认构造器)，进入读锁和写锁的顺序是没有指定的，以重入约束为准。一个非公平锁即：不断竞争，可能会使一个或多个读（写）线程无限期延迟，但是通常能提供比公平锁更高的吞吐量。

 - __*公平模式*__    
    当用公平模式构造`ReentrantReadWriteLock`, 线程使用一个近似的到达顺序策略来竞争锁。当当前保持的锁被释放，可能发生以下情况：  
    - 等待时间最长的一个写线程将被分配写锁；  
    - 如果有一组读线程等待的时间比所有的等待中的写线程要长，这组读线程将被分配读锁。  

    如果有写锁保持或者存在一个正在等待的写线程，那么一个线程试图获取一个公平的读锁（非重入获取）将会被阻塞。这个线程将无法获得读锁，直到最早等待的写线程获取到锁并且释放写锁。当然，如果一个等待的写线程放弃等待，留下一个或多个读线程作为队列里面等待时间最长的等待者等待写锁释放，那么这些线程将被分配读锁。
一个线程尝试获取一个公平的写锁（非重入获取）将会被阻塞除非所有的读锁和写锁都释放（意味着没有等待的线程）。(需要注意的是非阻塞方法`ReentrantReadWriteLock.ReadLock.tryLock()`和`ReentrantReadWriteLock.WriteLock.tryLock()`不履行这个公平设置并且将会获得锁，不管是否有等待的线程。)

- __重入__

    这个锁像`ReentrantLock`一样，允许读写线程两者都可以重新获取读锁或者写锁（*笔者注释：读线程只能重新获取读锁；写线程可以获取读锁和写锁*）。非重入的读线程不允许进入锁，除非所有写线程保持的写锁都被释放。
此外，一个写线程可以获取到读锁，但是反过来则不行。在某些应用中，当在调用或者回调的方法一直保持写锁并且需要在读锁下执行读操作的时候，重入很有用。

- __锁降级__

 重入同样允许写锁降级为读锁，通过获取写锁，然后获取读锁，然后释放写锁。然而从读锁升级成写锁是不可能的。

- __锁中断__

 读锁和写锁在锁定的时候都支持中断。

- __条件支持__

 写锁为相同行为提供了一个`Condition`的实现，和由`ReentrantLock.newCondition()`为`ReentrantLock`提供的`Condition`实现一样。这个条件只能被写锁使用。

 读锁不支持条件并且`readLock().newCondition()`抛出`UnsupportedOperationException`异常。

- __植入?__

 这个类支持让方法决定是继续保持锁还是竞争。这些方法被设计来做系统状态监控，不是为了同步控制。

此类行为同样以内置锁的方式序列化：反序列化的锁处于解锁状态，不管序列化的时候是什么状态。

__示例用法.__ 这里有一个代码草图演示如何在更新一个缓存后执行锁降级（当非嵌套方式处理多个锁的时候异常处理尤其棘手）：
<?prettify linenums=1?>
	class CachedData {
	   Object data;
	   volatile boolean cacheValid;
	   final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
	
	   void processCachedData() {
	     rwl.readLock().lock();
	     if (!cacheValid) {
	        // Must release read lock before acquiring write lock
	        rwl.readLock().unlock();
	        rwl.writeLock().lock();
	        try {
	          // Recheck state because another thread might have
	          // acquired write lock and changed state before we did.
	          if (!cacheValid) {
	            data = ...
	            cacheValid = true;
	          }
	          // Downgrade by acquiring read lock before releasing write lock
	          rwl.readLock().lock();
	        } finally {
	          rwl.writeLock().unlock(); // Unlock write, still hold read
	        }
	     }
	
	     try {
	       use(data);
	     } finally {
	       rwl.readLock().unlock();
	     }
	   }
	 }
	 

`ReentrantReadWriteLocks`在某些类型的集合中可以用来提高并发。当集合容量预期很大，读线程多于写线程并且操作（*笔者认为是线程临界区里的逻辑运算*）需要的开销超过同步开销，这通常是值得做的。例如，这是一个将大并发访问大容量`TreeMap`的类。
<?prettify linenums=1?>
    class RWDictionary {
	    private final Map<String, Data> m = new TreeMap<String, Data>();
	    private final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
	    private final Lock r = rwl.readLock();
	    private final Lock w = rwl.writeLock();
	
	    public Data get(String key) {
	        r.lock();
	        try { return m.get(key); }
	        finally { r.unlock(); }
	    }
	    public String[] allKeys() {
	        r.lock();
	        try { return m.keySet().toArray(); }
	        finally { r.unlock(); }
	    }
	    public Data put(String key, Data value) {
	        w.lock();
	        try { return m.put(key, value); }
	        finally { w.unlock(); }
	    }
	    public void clear() {
	        w.lock();
	        try { m.clear(); }
	        finally { w.unlock(); }
	    }
	 }

__实现说明__

此锁支持最大65535次递归写锁和65535次读锁。试图超出这些限制将在锁定方法抛出`java.lang.Error`。

## 内部实现

`ReentrantReadWriteLock`的内部实现是基于`AbstractQueuedSynchronizer`抽象类。

AQS相关的内容请参考：  
[AbstractQueuedSynchronizer的介绍和原理分析](http://ifeve.com/introduce-abstractqueuedsynchronizer/)  
[AQS的原理浅析](http://ifeve.com/java-special-troops-aqs/)  


__ReentrantReadWriteLock中的AQS__

  - __写锁__

  写锁的操作相对简单，`ReentrantReadWriteLock`中的AQS主要实现了这几个方法：
<?prettify linenums=1?>
      // 尝试获取写锁
      boolean tryAcquire(int acquires);
      // 尝试释放写锁
      boolean tryRelease(int releases);
      // 写锁是否被保持
      boolean isHeldExclusively();
      
  下面开始分析`tryAcquire`源码：
<?prettify linenums=1?>
      protected final boolean tryAcquire(int acquires) {
            /*
             * Walkthrough:
             * 1. If read count nonzero or write count nonzero
             *    and owner is a different thread, fail.
             * 2. If count would saturate, fail. (This can only
             *    happen if count is already nonzero.)
             * 3. Otherwise, this thread is eligible for lock if
             *    it is either a reentrant acquire or
             *    queue policy allows it. If so, update state
             *    and set owner.
             */
            Thread current = Thread.currentThread();
            int c = getState();// 获取当前计数器值
            int w = exclusiveCount(c);// 获取写锁持有的计数器值
            if (c != 0) {// 计数器不等于0，说明有线程保持了锁（可能是读锁也可能是写锁）
                // (Note: if c != 0 and w == 0 then shared count != 0)
                if (w == 0 || current != getExclusiveOwnerThread())// 没有写锁，或者当前线程不是当前保持写锁的线程（前提是计数器不等于0），所以获取写锁失败了
                    return false;
                if (w + exclusiveCount(acquires) > MAX_COUNT)// 写计数器溢出了
                    throw new Error("Maximum lock count exceeded");
                // Reentrant acquire	在排除了各种不符合的条件后，开始获取写锁（重入获取）
                setState(c + acquires);// 设置计数器状态，为什么不用compareAndSetState？
                return true;
            }
            // 代码走到这里说明了计数器状态是0，全新获取写锁，非重入方式获取
            if (writerShouldBlock() ||
                !compareAndSetState(c, c + acquires))// 如果需要阻塞写锁或者设置计数器状态出错了，返回失败
                return false;
            // 执行到这里只有一种情况：正常获取写锁
            setExclusiveOwnerThread(current);// 设置当前线程为排他线程
            // 为什么不用设置计数器状态？其实上面的代码compareAndSetState(c, c + acquires)已经设置过了，执行到这里，证明compareAndSetStat执行成功了。
            return true;
        }
        
  这个方法用于获取写锁，实现思路是这样的：  
1. 如果读计数器不为0或者写计数器不为0并且持有锁的线程不是当前线程，则失败。  
2. 如果计数器饱和，则失败。（这只能在计数器非0的时候发生，意味着只能是重入获取锁的时候发生）
3. 否则，这个线程符合上锁条件，不管这个线程是重入获取锁还是排队策略允许它获取锁。如果是这样的话，更新状态（`compareAndSetState`）并且设置锁的所有者(`setExclusiveOwnerThread`)。

 成功获取锁则返回`true`，失败有两种情况：普通失败返回`false`，计数器饱和直接抛出`java.lang.Error`。
  
 下面是`tryRelease`的源码：
<?prettify linenums=1?>
        /*
         * Note that tryRelease and tryAcquire can be called by
         * Conditions. So it is possible that their arguments contain
         * both read and write holds that are all released during a
         * condition wait and re-established in tryAcquire.
         */

        protected final boolean tryRelease(int releases) {
            if (!isHeldExclusively())// 如果当前线程没有保持写锁，释放写锁失败，抛出异常
                throw new IllegalMonitorStateException();
            int nextc = getState() - releases;// 计算这次释放后计数器剩余的值
            boolean free = exclusiveCount(nextc) == 0;// 计算释放后是不是彻底放弃写锁
            if (free)// 如果彻底放弃写锁，需要设置保持排他的线程为空
                setExclusiveOwnerThread(null);
            setState(nextc);
            return free;
        }
        
  这个方法用于释放写锁，需要知道的是`tryRelease`和`tryAcquire`可以通过条件调用。所以在某个条件等待的时候他们的包含读和写计数值的参数被全部释放并且在`tryAcquire`的时候重新建立。
  
  `isHeldExclusively`的原理很简单：
<?prettify linenums=1?>
        protected final boolean isHeldExclusively() {
            // While we must in general read state before owner, 我们需要在设置写锁所有者之前读取状态
            // we don't need to do so to check if current thread is owner  我们不需要检测当前状态是否是写锁所有者
            return getExclusiveOwnerThread() == Thread.currentThread();
        }
        
  写锁还需要提供几个额外的API：
<?prettify linenums=1?>
      boolean tryWriteLock();// 和`tryAcquire`逻辑一致，除了一点：它不调用`writerShouldBlock()`
      
      ConditionObject newCondition();// 新建一个条件
      
      boolean isWriteLocked();// 判断是否有写锁锁定
      
      int getWriteHoldCount();// 获取写锁计数器值
      
      boolean writerShouldBlock();// 这个取决于具体的实现，比如锁是否是公平的；  
      如果是公平锁，需要通过hasQueuedPredecessors查询是否存在比当前线程等待时间更长的需要获取锁的线程来决定是否阻塞当前线程。
      如果是非公平锁，不需要阻塞。
      
  
  - __读锁__

 读锁的实现相对复杂一些，`ReentrantReadWriteLock`中的AQS读锁相关的方法如下：  
<?prettify linenums=1?>
        // 尝试加读锁  
        int tryAcquireShared(int unused);  
        // 尝试释放读锁  
        boolean tryReleaseShared(int unused);
        
 此外还有一些辅助的变量和方法：  
<?prettify linenums=1?>
         int sharedCount(int c);
         
         private transient ThreadLocalHoldCounter readHolds;
         
         private transient HoldCounter cachedHoldCounter;
         
         private transient Thread firstReader = null;
         
         private transient int firstReaderHoldCount;
         
         boolean readerShouldBlock();
         
         int getReadHoldCount();
     

 下面详细分析这些字段和方法：
<?prettify linenums=1?>
        /**
         * A counter for per-thread read hold counts.
         * Maintained as a ThreadLocal; cached in cachedHoldCounter
         */
        static final class HoldCounter {
            int count = 0;
            // Use id, not reference, to avoid garbage retention
            final long tid = Thread.currentThread().getId();
        }
        
 `HoldCounter`为每个线程保存读计数器，缓存在变量`cachedHoldCounter`中，作为`ThreadLocal`维护。
<?prettify linenums=1?>
        /**
         * ThreadLocal subclass. Easiest to explicitly define for sake
         * of deserialization mechanics.
         */
        static final class ThreadLocalHoldCounter
            extends ThreadLocal<HoldCounter> {
            public HoldCounter initialValue() {
                return new HoldCounter();
            }
        }
        
 `ThreadLocalHoldCounter`是一个`ThreadLocal`的子类，它里面保存的对象是`HoldCounter`，从代码中可以看出它会为一个新的线程创建一个新的`HoldCounter`.
<?prettify linenums=1?>
        /**
         * The hold count of the last thread to successfully acquire
         * readLock. This saves ThreadLocal lookup in the common case
         * where the next thread to release is the last one to
         * acquire. This is non-volatile since it is just used
         * as a heuristic, and would be great for threads to cache.
         *
         * <p>Can outlive the Thread for which it is caching the read
         * hold count, but avoids garbage retention by not retaining a
         * reference to the Thread.
         *
         * <p>Accessed via a benign data race; relies on the memory
         * model's final field and out-of-thin-air guarantees.
         */
        private transient HoldCounter cachedHoldCounter;
        
 再来看看变量`cachedHoldCounter`的注释：  
 最后一个成功持有读锁的线程的计数器。这样做可以节省在通常情况下`ThreadLocal`的查找，其中的下一个线程的释放是最后一个获取的。这是非易失性的因为这只是用作启发式，线程缓存他们很好。被保存的线程读计数器可以活的比线程长，但是通过不保持它到线程的引用来避免垃圾滞留。  
通过一个良性数据争用存取；依赖于内存模型的常量字段和最低限度的安全性保证。
<?prettify linenums=1?>
        /**
         * The number of reentrant read locks held by current thread.
         * Initialized only in constructor and readObject.
         * Removed whenever a thread's read hold count drops to 0.
         */
        private transient ThreadLocalHoldCounter readHolds;

 `readHolds`是指当前线程保持的读计数器。只在构造器和`readObject`方法中初始化。每当一个线程的读计数器下降到0，移除当前线程`readHolds`的值。
<?prettify linenums=1?>
        /**
         * firstReader is the first thread to have acquired the read lock.
         * firstReaderHoldCount is firstReader's hold count.
         *
         * <p>More precisely, firstReader is the unique thread that last
         * changed the shared count from 0 to 1, and has not released the
         * read lock since then; null if there is no such thread.
         *
         * <p>Cannot cause garbage retention unless the thread terminated
         * without relinquishing its read locks, since tryReleaseShared
         * sets it to null.
         *
         * <p>Accessed via a benign data race; relies on the memory
         * model's out-of-thin-air guarantees for references.
         *
         * <p>This allows tracking of read holds for uncontended read
         * locks to be very cheap.
         */
        private transient Thread firstReader = null;
        private transient int firstReaderHoldCount;
        
 `firstReader`是一个线程实例，它是第一个获取到读锁的线程。同样的道理，`firstReaderHoldCount`是`firstReader`线程保持的读计数器。  
更确切地说，`firstReader`是把读计数器从0变到1的那个独特的线程，并且从那以后没有释放读锁；如果没有这样的线程，这个字段为空。  
 因为`tryReleaseShared`方法会把它设置为空,所以无法造成垃圾滞留除非线程线程终止而没有放弃它的读锁。  
通过一个良性数据争用存取；依赖于内存模型的最低限度的安全性保证引用。  
这种允许跟踪读计数器的方式对无竞争的读锁来说是非常便宜的。
<?prettify linenums=1?>
        protected final int tryAcquireShared(int unused) {
            /*
             * Walkthrough:
             * 1. If write lock held by another thread, fail.
             * 2. Otherwise, this thread is eligible for
             *    lock wrt state, so ask if it should block
             *    because of queue policy. If not, try
             *    to grant by CASing state and updating count.
             *    Note that step does not check for reentrant
             *    acquires, which is postponed to full version
             *    to avoid having to check hold count in
             *    the more typical non-reentrant case.
             * 3. If step 2 fails either because thread
             *    apparently not eligible or CAS fails or count
             *    saturated, chain to version with full retry loop.
             */
            Thread current = Thread.currentThread();
            int c = getState();// 获取锁计数器
            if (exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current)// 如果排他计数器不等于0并且当前线程不是排它锁保持的线程，则获取共享锁失败
                return -1;
            int r = sharedCount(c);// 共享计数器值
            if (!readerShouldBlock() &&
                r < MAX_COUNT &&
                compareAndSetState(c, c + SHARED_UNIT)) {// 读线程不需要阻塞并且共享计数器没有饱和并且用CAS更新读计数器值成功
                if (r == 0) {// 表示第一次进入读锁
                    firstReader = current;
                    firstReaderHoldCount = 1;
                } else if (firstReader == current) {// 表示重入读锁，并且当前线程就是firstReader
                    firstReaderHoldCount++;
                } else {// 重入读锁，当前线程不是firstReader
                    HoldCounter rh = cachedHoldCounter;// 从缓存中获取HoldCounter，可以参考上面HoldCounter的注释
                    if (rh == null || rh.tid != current.getId())// 没有获取到HoldCounter，或者获取到的HoldCounter不是当前线程的
                        cachedHoldCounter = rh = readHolds.get(); // 从当前线程中读取HoldCounter存到变量cachedHoldCounter和rh
                    else if (rh.count == 0) // count等于0的情况需要特殊处理，其他情况只需要count加1
                        readHolds.set(rh);
                    rh.count++;
                }
                return 1;
            }
            return fullTryAcquireShared(current);// 完整版本的获取共享锁
        }
        
 `tryAcquireShared`这个方法的实现思路是这样的：  
1. 如果另一个线程保持写锁，则失败。
2. 否则，这个线程符合锁的状态，由于排队策略，所以需要询问是否应该阻塞。如果不阻塞，尝试通CAS更改状态和更新计数器值取得授权。注意那个步骤没有检测重入获取，这会推迟到`fullTryAcquireShared`去避免在典型的不可重入的场景必须检测保持的计数器。
3. 如果步骤2失败，要么是线程不符合条件，要么CAS失败，要么计数器值溢出，链接到方法`fullTryAcquireShared`。

接下去看看完整版本的获取读锁`fullTryAcquireShared`的代码：
<?prettify linenums=1?>
        /**
         * Full version of acquire for reads, that handles CAS misses
         * and reentrant reads not dealt with in tryAcquireShared.
         */
        final int fullTryAcquireShared(Thread current) {
            /*
             * This code is in part redundant with that in
             * tryAcquireShared but is simpler overall by not
             * complicating tryAcquireShared with interactions between
             * retries and lazily reading hold counts.
             */
            HoldCounter rh = null;
            for (;;) {
                int c = getState();
                if (exclusiveCount(c) != 0) {
                    if (getExclusiveOwnerThread() != current)
                        return -1;
                    // else we hold the exclusive lock; blocking here
                    // would cause deadlock.
                } else if (readerShouldBlock()) {
                    // Make sure we're not acquiring read lock reentrantly
                    if (firstReader == current) {
                        // assert firstReaderHoldCount > 0;
                    } else {
                        if (rh == null) {
                            rh = cachedHoldCounter;
                            if (rh == null || rh.tid != current.getId()) {
                                rh = readHolds.get();
                                if (rh.count == 0)
                                    readHolds.remove();
                            }
                        }
                        if (rh.count == 0)
                            return -1;
                    }
                }
                if (sharedCount(c) == MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                if (compareAndSetState(c, c + SHARED_UNIT)) {
                    if (sharedCount(c) == 0) {
                        firstReader = current;
                        firstReaderHoldCount = 1;
                    } else if (firstReader == current) {
                        firstReaderHoldCount++;
                    } else {
                        if (rh == null)
                            rh = cachedHoldCounter;
                        if (rh == null || rh.tid != current.getId())
                            rh = readHolds.get();
                        else if (rh.count == 0)
                            readHolds.set(rh);
                        rh.count++;
                        cachedHoldCounter = rh; // cache for release
                    }
                    return 1;
                }
            }
        }

 获取读锁的完整版本，用于处理CAS失误和`tryAcquireShared`没有处理的重入读。

 此代码部分和`tryAcquireShared`方法冗余，但是通过不让重试和延迟读保持计数器与`tryAcquireShared`产生复杂的相互作用让总体比较简单。
<?prettify linenums=1?>
        protected final boolean tryReleaseShared(int unused) {
            Thread current = Thread.currentThread();
            if (firstReader == current) {
                // assert firstReaderHoldCount > 0;
                if (firstReaderHoldCount == 1)
                    firstReader = null;
                else
                    firstReaderHoldCount--;
            } else {
                HoldCounter rh = cachedHoldCounter;
                if (rh == null || rh.tid != current.getId())
                    rh = readHolds.get();
                int count = rh.count;
                if (count <= 1) {
                    readHolds.remove();
                    if (count <= 0)
                        throw unmatchedUnlockException();
                }
                --rh.count;
            }
            for (;;) {
                int c = getState();
                int nextc = c - SHARED_UNIT;
                if (compareAndSetState(c, nextc))
                    // Releasing the read lock has no effect on readers,
                    // but it may allow waiting writers to proceed if
                    // both read and write locks are now free.
                    return nextc == 0;
            }
        }
        
 这个方法用于释放共享锁，实现思路是这样的：  
1. 判断当前线程是不是`firstReader`，如果是的话需要把`firstReaderHoldCount`计数器减一（当`firstReaderHoldCount`值为1 的时候，需要清空`firstReader`）。  
2. 否则，从`cachedHoldCounter`获取当前线程的HoldCounter，如果获取到的HoldCounter是空的或者它不是当前线程的HoldCounter，那么从`readHolds`获取HoldCounter，然后对HoldCounter的count减一（这里需要处理count为1的情况）。  
3. 通过`compareAndSetState`设置计数器值。

 需要注意的是：释放读锁不会影响读线程，但是同时释放读锁和写锁可能允许等待的写线程继续执行。
 
__公平锁 VS 非公平锁__

 这两者的区别在于读和写的时候是否需要排队，`ReentrantReadWriteLock`通过下面两个方法来实现的(在上面的分析中提到过)：
<?prettify linenums=1?>
         boolean writerShouldBlock();
         
         boolean readerShouldBlock();
         
 `writerShouldBlock`在`tryAcquire`方法中被调用，`readerShouldBlock`在`tryAcquireShared`和`fullTryAcquireShared`方法中被调用。
 
 非公平锁中的实现：  
<?prettify linenums=1?>
        final boolean writerShouldBlock() {
            return false; // writers can always barge
        }
        final boolean readerShouldBlock() {
            /* As a heuristic to avoid indefinite writer starvation,
             * block if the thread that momentarily appears to be head
             * of queue, if one exists, is a waiting writer.  This is
             * only a probabilistic effect since a new reader will not
             * block if there is a waiting writer behind other enabled
             * readers that have not yet drained from the queue.
             */
            return apparentlyFirstQueuedIsExclusive();
        }

 从代码看出，在非公平锁中`writerShouldBlock`永远返回false（意味着不需要阻塞）。`readerShouldBlock`则调用AQS的`apparentlyFirstQueuedIsExclusive`方法，主要是为了避免写线程不确定的饥饿，如果这个线程暂时出现在队列的头部则阻塞，如果存在的话，它是一个写线程。这只是一定概率的影响，因为如果有落后于其它运行的读线程的写线程尚未从队列中弹出，则一个新的读线程不会阻塞。
 
 公平锁中的实现：
<?prettify linenums=1?>
        final boolean writerShouldBlock() {
            return hasQueuedPredecessors();
        }
        final boolean readerShouldBlock() {
            return hasQueuedPredecessors();
        }
        
 这两个方法都是通过`hasQueuedPredecessors`的逻辑来实现的。如果在当前线程之前，有线程在等待获取锁，那么返回`true`，当前线程将被阻塞，否则当前线程不需要阻塞。
 
## 锁API使用

以下是通用API：
<?prettify linenums=1?>
    final ReadWriteLock rwl = new ReentrantReadWriteLock();  
    final Lock readLock = rwl.readLock();  
    final Lock writeLock = rwl.writeLock();
        
如果想要使用更加强大的功能，请使用下面的API：  
<?prettify linenums=1?>
    final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();   
    final ReentrantReadWriteLock.ReadLock readLock = rwl.readLock();  
    final ReentrantReadWriteLock.WriteLock writeLock = rwl.writeLock();
    
不难看出，`ReentrantReadWriteLock`对于锁的操作都是委托给`ReadLock`和`WriteLock`这两个内部类来实现的（最终还是委托给继承AQS的`Sync`）。

全文完-

