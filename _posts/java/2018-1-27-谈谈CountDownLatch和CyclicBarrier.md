---
layout: post
title: 谈谈CountDownLatch和CyclicBarrier
categories: [Java]
description: 谈谈CountDownLatch和CyclicBarrier
keywords: Java, 同步, 多线程
---

<h1 align="center">谈谈CountDownLatch和CyclicBarrier</h1>
Java中CountDownLatch和CyclicBarrier都是用来做多线程同步的。下面分析一下他们功能的异同。

## CountDownLatch
CountDownLatch基于AQS([同步器AbstractQueuedSynchronizer浅析]({{ site.url }}/2018/01/27/同步器AbstractQueuedSynchronizer浅析))，CountDownLatch中有一个内部类Sync，Sync继承自AbstractQueuedSynchronizer。
我们先看一个CountDownLatch的例子，然后再具体分析源码。
### 一个CountDownLatch例子
```
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class CountDownLatchExample {

    private ExecutorService executorService;
    private CountDownLatch countDownLatch;
    private int parties;

    public static void main(String[] args){
        CountDownLatchExample countDownLatchExample = new CountDownLatchExample(10);
        try {
            countDownLatchExample.example();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public CountDownLatchExample(int parties) {
        executorService = Executors.newFixedThreadPool(parties);
        countDownLatch = new CountDownLatch(parties);
        this.parties = parties;
    }

    public void example() throws InterruptedException {
        for (int i = 0; i < parties; i++) {
            executorService.submit(() -> {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread().getName() + " gets job done");
                countDownLatch.countDown(); // 线程完成任务之后countDown
            });
        }
        // 等待所有的任务完成
        countDownLatch.await();
        System.out.println(Thread.currentThread().getName() + " reach barrier");
        executorService.shutdown();
    }
}

```
上面是一个CountDownLatch的例子，CountDownLatch的API还是很简单的，主要就是countDown和await两个方法。CountDownLatch实例化时将count设置为AQS的state，每次countDown时CAS将state设置为state - 1，await时首先会检查当前state是否为0，如果为0则代表所有的任务完成了，await结束，否则主线程将循环重试，直到线程被中断或者任务完成或者等待超时。下面具体看看CountDownLatch的源码。

### CountDownLatch源码分析
首先看一下CountDownLatch的构造函数和Sync的代码：
```
public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0");
    this.sync = new Sync(count); // 设置Sync（AQS）的state为count
}
    
private static final class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = 4982264981922014374L;

    Sync(int count) {
        setState(count); // 设置AQS的state为count
    }

    int getCount() {
        return getState();
    }

    // 
    protected int tryAcquireShared(int acquires) {
        return (getState() == 0) ? 1 : -1; // 重写了AQS的方法
    }

    protected boolean tryReleaseShared(int releases) {
        // Decrement count; signal when transition to zero
        for (;;) { // 循环CAS重试
            int c = getState();
            if (c == 0)
                return false;
            int nextc = c-1;
            if (compareAndSetState(c, nextc))
                return nextc == 0;
        }
    }
}
```
CountDownLatch构造参数count代表当前参与同步的线程数目，然后设置当前AQS的状态为count。Sync覆写了tryAcquireShared和tryReleaseShared方法。tryAcquireShared方法中会判断当前AQS的state是否为0，如果是0才能获取成功（返回1），否则，获取失败（返回-1）。tryReleaseShared通过for循环进行CAS设置状态。
CountDownLatch中最主要的两个方法是：countDown和await：
```
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}

public boolean await(long timeout, TimeUnit unit)
    throws InterruptedException {
    return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
}

public void countDown() {
    sync.releaseShared(1);
}
```
每次一个任务完成之后，调用CountDownLatch的countDown方法，将当前AQS的state减1（AQS初始state为count）。countDown的逻辑其实比较简单，就是通过重试CAS设置当前AQS的state为state - 1。这里着重看一下await方法，await方法体中调用AQS的acquireSharedInterruptibly或者tryAcquireSharedNanos方法。看一下AQS的acquireSharedInterruptibly方法：

```
public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0) // tryAcquireShared这里已经在Sync中重写了
        doAcquireSharedInterruptibly(arg);
}

private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) { // 循环重试
            final Node p = node.predecessor(); // 对于CountDownLatch来说，等待队列中其实只有一个main线程在等待，因此这里第一次就应该判断条件`p == head`成立
            if (p == head) {
                int r = tryAcquireShared(arg); // 方法已经在CountDownLatch$Sync中重写了
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
            // 如果当前失败，则挂起线程，循环重试
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```
前面已经分析过AQS了，这里对AQS部分不多做解释了。主要是这里的tryAcquireShared方法，CountDownLatch的Sync内部类中重写了该方法。如果当前state等于0才能获取成功，因此只有当所有任务完成，此时AQS的state为0，await方法才会返回（不考虑中断和超时）。

## CyclicBarrier
CyclicBarrier较CountDownLatch而言主要多了两个功能：
1. 支持重置状态，达到循环利用的目的。这也是Cyclic的由来。CyclicBarrier中有一个内部类Generation，代表当前的同步处于哪一个阶段。当最后一个任务完成，执行任务的线程会通过nextGeneration方法来重置Generation。也可以通过CyclicBarrier的reset方法来重置Generation。
2. 支持barrierCommand，当最后一个任务运行完成，执行任务的线程会检查CyclicBarrier的barrierCommand是否为null，如果不为null，则运行该任务。

### 一个CyclicBarrier例子
```
import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class CyclicBarrierExample {

    private ExecutorService executorService;
    private CyclicBarrier cyclicBarrier;
    private int parties;

    public CyclicBarrierExample(int parties) {
        executorService = Executors.newFixedThreadPool(parties);
        cyclicBarrier = new CyclicBarrier(parties, () -> System.out.println(Thread.currentThread().getName() + " gets barrierCommand done"));
        this.parties = parties;
    }

    public static void main(String[] args) {
        CyclicBarrierExample cyclicBarrierExample = new CyclicBarrierExample(10);
        cyclicBarrierExample.example();
    }

    public void example() {
        for (int i = 0; i < parties; i++) {
            executorService.submit(() -> {
                try {
                    Thread.sleep(1000);
                    cyclicBarrier.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } catch (BrokenBarrierException e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread().getName() + " gets job done");
            });
        }
        executorService.shutdown();
    }
}

```
可以看到这里CyclicBarrier主要的API就是await方法，每个任务最后调用这个方法等待最后一个任务完成，在这之前所有的线程都会等待。

### CyclicBarrier源码分析
CyclicBarrier主要有以下属性：
```
/** The lock for guarding barrier entry */
private final ReentrantLock lock = new ReentrantLock(); // 锁
/** Condition to wait on until tripped */
private final Condition trip = lock.newCondition(); // 锁关联的Condition，用于线程同步
/** The number of parties */
private final int parties; // 多少个任务参与同步
/* The command to run when tripped */
private final Runnable barrierCommand; // 最后一个任务运行线程应该执行的command
/** The current generation */
private Generation generation = new Generation(); // 当前CyclicBarrier所处的Generation

/**
 * Number of parties still waiting. Counts down from parties to 0
 * on each generation.  It is reset to parties on each new
 * generation or when broken.
 */
private int count; // 剩余的等待的任务数目，取值范围：[0-parties]
```

CyclicBarrier主要的API就是await方法和reset方法。
```
public int await() throws InterruptedException, BrokenBarrierException {
    try {
        return dowait(false, 0L); // 等待所有任务完成
    } catch (TimeoutException toe) {
        throw new Error(toe); // cannot happen
    }
}

public void reset() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        breakBarrier();   // break the current generation
        // 开启下一个Generation，达到循环使用的目的
        nextGeneration(); // start a new generation
    } finally {
        lock.unlock();
    }
}
```
在每一个任务结束时我们调用CyclicBarrier的await方法，所有的线程都会等待最后一个任务完成才会退出。注意CountDownLatch中任务执行线程调用完了countDown方法之后就会退出，不会等待最后一个任务完成才会退出，这也是CountDownLatch和CyclicBarrier的一个区别。
我们主要看一下dowait方法：
```
private int dowait(boolean timed, long nanos)
    throws InterruptedException, BrokenBarrierException,
           TimeoutException {
    final ReentrantLock lock = this.lock; // 加锁
    lock.lock();
    try {
        final Generation g = generation; // 当前CyclicBarrier所处的Generation

        if (g.broken) // 检查Generation的broken标志位
            throw new BrokenBarrierException(); // 被中断

        if (Thread.interrupted()) { // 检查当前线程是否被中断，响应中断
            breakBarrier();
            throw new InterruptedException();
        }

        int index = --count; // 剩余等待任务数
        if (index == 0) {  // tripped
            boolean ranAction = false;
            try {
                final Runnable command = barrierCommand;
                if (command != null) // 如果当前任务是最后一个任务，则执行任务线程运行barrierCommand
                    command.run();
                ranAction = true;
                nextGeneration(); // 重置Generation
                return 0;
            } finally {
                if (!ranAction)
                    breakBarrier();
            }
        }

        // loop until tripped, broken, interrupted, or timed out
        for (;;) {
            try {
                // 阻塞当前线程，等待最后一个任务完成，然后线程会通过nextGeneration方法调用trip.signallAll唤醒等待线程。注意当线程进入WAITING（调用await）状态或者TIMED_WAITING（awaitNanos）状态后会让出锁。
                if (!timed)
                    trip.await();
                else if (nanos > 0L)
                    nanos = trip.awaitNanos(nanos);
            } catch (InterruptedException ie) {
                if (g == generation && ! g.broken) {
                    breakBarrier();
                    throw ie;
                } else {
                    // We're about to finish waiting even if we had not
                    // been interrupted, so this interrupt is deemed to
                    // "belong" to subsequent execution.
                    Thread.currentThread().interrupt();
                }
            }

            if (g.broken) // 响应中断
                throw new BrokenBarrierException();

            if (g != generation) // 已经是下一个generation，说明最后一个任务已经执行完成，返回
                return index;

            if (timed && nanos <= 0L) { // 检查是否超时
                breakBarrier();
                throw new TimeoutException();
            }
        }
    } finally {
        lock.unlock();
    }
}
```
1. 首先会检查当前Generation的broken标志位，如果为true，则抛出异常。再响应中断请求。
2. 检查当前是否为最后一个任务，如果是则检查barrierCommand是否为null，如果不为null则执行任务。如果不是最后一个任务则到步骤3。
3. 判断是否需要阻塞当前线程，如果需要则通过trip的await或者awaitNanos方法使其进入休眠，注意休眠后线程会让出锁。如果休眠超时则抛出TimeoutException异常。休眠完成后（要么休眠超时要么被最有一个任务执行线程唤醒）会响应中断请求，再判断当前generation是否改变（最后一个执行任务线程会通过nextGeneration方法改变generation），如果改变直接返回。

其流程图如下：
<div align="center">
    <img src="{{ site.url }}/images/posts/java/CyclicBarrierDoAwait.png" alt="CyclicBarrierDoAwait"/>
</div>

## 总结
1. CountDownLatch和CyclicBarrier都是用作多线程同步，CountDownLatch基于AQS，CyclicBarrier基于ReentrantLock。
2. CyclicBarrier支持复用和`barrierCommand`，但是CountDownLatch不支持。
3. CyclicBarrier会阻塞线程，在最后一个任务执行线程完成之前，其余线程都必须等待，而线程在调用CountDownLatch的countDown方法之后就会结束。
