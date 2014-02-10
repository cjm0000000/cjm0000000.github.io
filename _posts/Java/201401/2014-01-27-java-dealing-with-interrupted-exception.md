---
layout: post
title: "[译] Java理论和实践：处理InterruptedException"
description: "Java中InterruptedException的推荐处理方式"
category: Java
tags: [Multi-Thread]
---
{% include JB/setup %}

__本文译自[Java theory and practice: Dealing with InterruptedException](http://www.ibm.com/developerworks/java/library/j-jtp05236/index.html) | [PDF](http://www.ibm.com/developerworks/java/library/j-jtp05236/j-jtp05236-pdf.pdfhttp://www.ibm.com/developerworks/java/library/j-jtp05236/j-jtp05236-pdf.pdf)__

这个故事大概很熟悉：你正在写一个测试程序并且你需要暂停一段时间，一次你调用了`Thread.sleep()`方法。但随后编译器或者IDE指出你还没有处理检查异常`InterruptedException`。`InterruptedException`是什么？为什么你需要处理它？  
对`InterruptedException`异常最常见的反应是吞掉它 -- 捕获它并且什么都不做（或者记录它，这没有任何改进） --正如我们将在[清单4](#code4)看到的。不幸的是这种方法扔掉了中断发生的事实这个重要信息，这可能会损害应用程序取消活动的线程或者及时关闭的能力。

### 阻塞方法

当一个方法抛出`InterruptedException`异常，这是告诉你它除了可以抛出一个特定的检查异常以外的几件事情。这是告诉你它是一个阻塞方法而且它将试图解除阻塞提早返回 -- 如果你处理得当。  

一个阻塞方法与仅仅需要很长时间运行的普通方法不同。普通方法的结束仅仅依赖于你给它做的工作量以及是否有足够的计算资源（CPU周期和内存）。阻塞方法的结束，换句话说，也需要依赖一些外部事件，比如计时器超时，I/O结束，或者另一个线程的作用（释放锁，设置状态，或者把任务放到工作队列）。普通方法当他们的工作一完成就结束，但是阻塞方法由于依赖外部事件难以预料。阻塞方法可能影响响应能力,因为很难预测它们何时会结束。  
让阻塞方法可以被取消通常很有用，因为阻塞方法可能会永远等下去，如果他们等待的事件从来没有发生（对于长时间运行的非阻塞方法同样有用）。可取消操作是：一个相比方法正常结束，它可以从外部提前结束方法。线程提供的并且由`Thread.sleep()`和`Object.wait()`支持的中断机制是一个取消机制。它允许一个线程请求另一个线程停止正在做的事情。当一个方法抛出`InterruptedException`异常，这是告诉你如果线程执行的方法被中断，它将试图阻止正在做的事情提前返回，并且通过抛出`InterruptedException`异常表明它提前返回了。行为端正的阻塞类库方法应该响应中断并且抛出`InterruptedException`异常，这样他们可以被用来内部取消任务而不影响响应能力。

### 线程中断
 
每个线程都有一个与之关联的`Boolean`属性，代表它的中断状态。中断状态初始值是`false`；当一个线程被某个其他线程用`Thread.interrupt()`中断的时候，会发生下面两种情况之一：如果线程正在执行一个像`Thread.sleep()`，`Thread.join()`，`Object.wait()`这样的低可中断阻塞方法，它会马上解除阻塞并且抛出一个`InterruptedException`异常。否则`interrupt()`方法仅仅设置线程的中断状态。中断线程中运行的代码之后可以轮询中断状态，看它是否被要求停止正在做的事情；中断状态可以通过`Thread.isInterrupted()`方法来读取并且可以被一个叫做`Thread.interrupted()`的方法清除。  
中断是一种协同机制。当一个线程中断另外一个，被中断的线程不需要立即停止正在做的事情。取而代之，中断是一种礼貌的方式要求另一个线程在他方便的时候停止正在做的事情，如果它愿意去做。有些方法，比如`Thread.sleep()`，认真对待这个请求，但是方法不必注意中断。非阻塞但是还可能需要很长时间来执行的方法可以通过轮训中断状态并且提前返回（如果中断的话）来遵守中断请求。你可以自由地忽略中断请求，但是这样做会影响响应。  
中断的协作特性的好处之一是，它为安全地构造可取消活动提供了更多的灵活性。我们很少要一个活动线程立即停止；如果活动线程正在更新的时候被取消，程序数据结构可能处于不一致状态。中断允许一个可取消的活动线程清理正在进行的工作，恢复不变量，通知其他活动线程我已经取消，然后终止。

### 处理InterruptedException异常

如果抛出`InterruptedException`异常意味着这是一个阻塞方法，那么调用一个阻塞方法意味着你的方法也是一个阻塞方法，并且你需要一种策略处理`InterruptedException`异常。通常最简单的一种策略是自己把`InterruptedException`异常抛出去，如 [清单1](#code1)中的方法`putTask()`和`getTask()`所示。这样做让你的方法响应中断，只不过通常需要添加`InterruptedException`异常到你的`throws`子句。

#### <span id="code1">清单1</span>. 传递`InterruptedException`给调用者  
    public class TaskQueue {
        private static final int MAX_TASKS = 1000;

        private BlockingQueue<Task> queue 
            = new LinkedBlockingQueue<Task>(MAX_TASKS);

        public void putTask(Task r) throws InterruptedException { 
            queue.put(r);
        }

        public Task getTask() throws InterruptedException { 
            return queue.take();
        }
    }

有时候需要在异常传播之前做一些清理工作。在这种情况下，你可以捕获`InterruptedException`异常，执行清理，然后重新抛出异常。[清单2](#code2)，一个匹配在线游戏服务中玩家的机制，阐明了这种技术。`matchPlayers()`将等待两名玩家到达，然后开始一个新的游戏。如果线程在一个玩家到达后但是另外一个玩家还没到达的时候被中断，它在重新抛出`InterruptedException`异常之前把玩家放回队列，这样玩家的游戏请求就不会丢失。
	
#### <span id="code2">清单2</span>. 在重新抛出`InterruptedException`异常之前执行特定任务  
    public class PlayerMatcher {
        private PlayerSource players;

        public PlayerMatcher(PlayerSource players) { 
            this.players = players; 
        }

        public void matchPlayers() throws InterruptedException { 
            Player playerOne, playerTwo;
            try {
                while (true) {
                    playerOne = playerTwo = null;
                    // Wait for two players to arrive and start a new game
                    playerOne = players.waitForPlayer(); // could throw IE
                    playerTwo = players.waitForPlayer(); // could throw IE
                    startNewGame(playerOne, playerTwo);
                }
            }
            catch (InterruptedException e) {  
                // If we got one player and were interrupted, put that player back
                if (playerOne != null)
                    players.addFirst(playerOne);
                // Then propagate the exception
                throw e;
            }
        }
    }

### 不要吞下中断

有时抛出`InterruptedException`异常不是一个选择，比如当一个`Runnable`定义的任务调用一个可中断的方法。在这种情况下，你不能重新抛出`InterruptedException`异常，但是你也不想什么都不做。当一个阻塞方法发现中断并且抛出`InterruptedException`异常，它清除了中断状态。如果你捕获了`InterruptedException`异常但是无法重新抛出，你应该保存中断发生的证据以便上层代码在调用堆栈可以获悉中断并且在它想要应对的时候应对它。这个任务是通过调用`interrupt()`来重新中断当前线程，如[清单3](#code3)所示。至少，无论何时你捕获了`InterruptedException`异常，不要重新抛出它，在返回之前重新中断当前线程。

#### <span id="code3">清单3</span>. 在捕获`InterruptedException`异常后恢复中断状态  
    public class TaskRunner implements Runnable {
        private BlockingQueue<Task> queue;

        public TaskRunner(BlockingQueue<Task> queue) { 
            this.queue = queue; 
        }

        public void run() { 
            try {
                 while (true) {
                     Task task = queue.take(10, TimeUnit.SECONDS);
                     task.execute();
                 }
             }
             catch (InterruptedException e) { 
                 // Restore the interrupted status
                 Thread.currentThread().interrupt();
             }
        }
    }

处理`InterruptedException`异常最糟糕的方式是你可以吞咽它 -- 捕获它并且既不重新抛出也不重申线程的中断状态。标准的方式处理异常你没有计划 -- 捕获它并且记录它 -- 也算是吞咽中断，因为上层代码在调用堆栈将不能了解它。（记录`InterruptedException`异常也仅仅是愚蠢的，因为当人去阅读这个记录的时候，对它做任何事都太晚了）[清单4](#code4) 显示了通常吞下一个中断模式：  

#### <span id="code4">清单4</span>. 吞下一个中断 -- 不要这样做  
    // Don't do this 
    public class TaskRunner implements Runnable {
        private BlockingQueue<Task> queue;

        public TaskRunner(BlockingQueue<Task> queue) { 
            this.queue = queue; 
        }

        public void run() { 
            try {
                 while (true) {
                     Task task = queue.take(10, TimeUnit.SECONDS);
                     task.execute();
                 }
             }
             catch (InterruptedException swallowed) { 
                 /* DON'T DO THIS - RESTORE THE INTERRUPTED STATUS INSTEAD */
             }
        }
    }

如果你无法重新抛出异常，不论你是否计划采取行动中断请求，你仍然需要重新中断当前线程，因为一个单一的中断请求可能有多个接受者。标准的线程池`ThreadPoolExecutor`工作线程实现了响应中断，因此中断一个运行在线程池中的任务可能产生的影响是取消任务并且通知执行线程线程池正在关闭。如果任务吞咽了中断请求，工作线程可能不知道产生了一个中断请求，这可能推迟关闭应用程序或服务。

### 实现可取消任务

语言规范中并没有为中断提供特定的语义，但是在大型程序中，很难维持任何中断语义除了取消。根据不同的活动，用户可以通过图形界面或者通过网路机制（`JMX`或者`Web Services`）请求取消。也可以通过程序逻辑请求取消。例如，一个网络爬虫如果检测到磁盘满了，会自动关闭自身，或者一个并行算法可能会启动多个线程搜索解空间的不同的区域，一旦其中一个线程找到解决方案就会取消这些线程。  
这仅仅因为一个任务是可取消的，并不意味着它需要立即响应中断请求。对于在一个循环中执行的代码，通常每次循环只检测一次中断。根据循环执行的时间，在任务代码通知线程被中断之前可能需要一段时间（要么通过`Thread.isInterrupted()`轮询中断状态，要么调用一个阻塞方法）。如果这个任务需要更多的响应，它可以更频繁地轮询中断状态。阻塞方法通常在入口就立即轮询中断状态，如果为了提高响应，直接抛出`InterruptedException`异常。  
一次可以接受生吞中断的情况是你知道线程即将退出。这种情况只发生在类调用可中断的方法是线程的一部分，而不是`Runnable`或通用库代码，如[清单5](#code5)所示。它会创建一个线程，该线程列举素数，直到它被中断，并且允许线程在被中断时退出。在两个地方用于循环检测中断：一处是在`while`循环的开头循环检测`isInterrupted()`方法，另一处是在调用阻塞方法`BlockingQueue.put()`的时候。

#### <span id="code5">清单5</span>. 中断可以吞下如果你知道线程即将退出  
    public class PrimeProducer extends Thread {
        private final BlockingQueue<BigInteger> queue;

        PrimeProducer(BlockingQueue<BigInteger> queue) {
            this.queue = queue;
        }

        public void run() {
            try {
                BigInteger p = BigInteger.ONE;
                while (!Thread.currentThread().isInterrupted())
                    queue.put(p = p.nextProbablePrime());
            } catch (InterruptedException consumed) {
                /* Allow thread to exit */
            }
        }

        public void cancel() { interrupt(); }
    }

### 不响应中断阻塞

并不是所有的阻塞方法都会抛出`InterruptedException`异常。输入和输出流类可能阻塞等待I/O完成，但是他们不会抛出`InterruptedException`异常，并且他们不会提前返回如果他们被中断。然而，在套接字I/O中，如果一个线程关闭套接字，其他线程阻塞在那个套接字I/O上的操作会通过抛出一个`SocketException`异常提前结束。在`java.nio`中的非阻塞I/O类也不支持可中断I/O，但是阻塞操作可以通过关闭通道同样被取消或请求一个`Selector`上的唤醒。类似的，尝试获取一把内在锁（进入一个同步块）无法被中断，但是`ReentrantLock`支持可中断获取锁模式。

### 无法取消的任务

一些任务简单拒绝被中断，使他们不可被取消。然而，即使是不可被取消的任务应该试图保留中断状态，万一调用堆栈的上层代码想要在无阻塞任务结束后对中断进行处理。[清单6](#code6)显示了一个方法等待一个阻塞队列直到它可用，不管它是否被中断。作为一个好公民，它在完成后，在`finally`块中恢复了线程的中断状态，为的是不剥夺中断调用者的请求。（它不能提早恢复中断状态，因为它会导致无线循环 -- `BlockingQueue.take()`会在进入方法的时候立即轮询中断状态并且抛出`InterruptedException`异常，如果它发现中断状态被设置。）

#### <span id="code6">清单6</span>. Noncancelable task that restores interrupted status before returning
    public Task getNextTask(BlockingQueue<Task> queue) {
        boolean interrupted = false;
        try {
            while (true) {
                try {
                    return queue.take();
                } catch (InterruptedException e) {
                    interrupted = true;
                    // fall through and retry
                }
            }
        } finally {
            if (interrupted)
                Thread.currentThread().interrupt();
        }
    }

### 总结

你可以使用Java平台提供的合作中断机制来构造灵活的取消策略。活动线程决定他们是否可以取消，他们想怎样响应中断，并且如果立即返回会危害程序的完整性，他们可以推迟中断，先去执行特定于任务的清理。即使你想完全忽视代码中的中断，确保你捕获`InterruptedException`异常后恢复中断状态并且不要重新抛出它以便调用它的代码不会被剥夺知晓一个中断的发生。