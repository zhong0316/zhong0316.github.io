---
layout: post
title: 同步器AbstractQueuedSynchronizer浅析
categories: [Java]
description: 同步器AbstractQueuedSynchronizer浅析
keywords: Java, 锁, AQS, 多线程, 并发
---

<h1 align="center">同步器AbstractQueuedSynchronizer浅析</h1>
Java中的锁主要有：synchronized锁和JUC(java.util.concurrent)locks包中的锁。synchronized锁是JVM的内置锁，底层通过"monitorenter"和"monitorexit"字节码指令实现。JUC中的锁支持公平锁（synchronized锁是非公平锁），读写锁，锁请求中断，锁请求超时等。今天要说的AbstractQueuedSynchronizer（AQS）是JUC锁的基础。JUC中的ReentrantLock，ReentrantReadWriteLock，Semaphore，CountDownLatch等都用到了AQS作为同步器。可以说AQS是JUC(java.util.concurrent)locks包中这些锁的基础。

## 同步队列
AQS本质上是一个FIFO的队列，它的等待队列是“CLH”（Craig, Landin, and Hagersten）队列的变种。CLH队列通常用作自旋锁（spinlocks）。
<div align="center">
    <img src="{{ site.url }}/images/posts/java/AQS队列.png" alt="AQS队列"/>
</div>

AQS使用一个int的state来记录当前锁的状态：
```
private volatile int state; // 状态
protected final int getState() {
    return state;
}
protected final void setState(int newState) {
    state = newState;
}
protected final boolean compareAndSetState(int expect, int update) { // 通过CAS设置状态
    // See below for intrinsics setup to support this
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```

AQS支持两种锁模式：独占锁和共享锁。如果当前AQS独占锁被获取后，在独占锁线程未释放之前，其他的独占锁和共享锁请求都将被阻塞；如果共享锁被获取，独占锁请求将被阻塞，而其他的共享锁请求可以成功。在读写锁ReentrantReadWriteLock中，AQS的高16位表示读锁（共享锁）状态，低16位表示写锁（独占锁）状态。

### Node类
```
static final class Node {

    // 标识当前节点在等待共享锁
    static final Node SHARED = new Node();
    // 标识当前节点在等待独占锁
    static final Node EXCLUSIVE = null;
    
    // 当前节点等待被中断或者超时
    static final int CANCELLED =  1;
    // 当前节点等待取消或者释放锁之后需要unpark它的后继节点
    static final int SIGNAL    = -1;
    // 当前节点在condition queue中等待
    static final int CONDITION = -2;
    // 只有头节点才能设置改状态，当请求处于共享状态下时，当前线程被唤醒之后可能还需要唤醒其他线程。后续节点需要传播该唤醒动作
    static final int PROPAGATE = -3;
    // 当前节点的等待状态
    volatile int waitStatus;

    // 前驱等待节点
    volatile Node prev;
    // 后置等待节点
    volatile Node next;

    // 当前节点关联的线程
    volatile Thread thread;

    // 指向condition queue等待的节点，或者指向SHARE节点，表明当前处于共享模式
    Node nextWaiter;

    // 当前节点是否等待共享锁
    final boolean isShared() {
        return nextWaiter == SHARED;
    }

    final Node predecessor() throws NullPointerException {
        Node p = prev;
        if (p == null)
            throw new NullPointerException();
        else
            return p;
    }

    Node() {    // Used to establish initial head or SHARED marker
    }

    Node(Thread thread, Node mode) {     // Used by addWaiter
        this.nextWaiter = mode;
        this.thread = thread;
    }

    Node(Thread thread, int waitStatus) { // Used by Condition
        this.waitStatus = waitStatus;
        this.thread = thread;
    }
}
```
Node中定义了CANCELLED、SIGNAL、CONDITION和PROPAGATE四种状态：
1. CANCELLED（1）：代表当前节点等待超时或者被中断。
2. SIGNAL（-1）：当前节点取消或者释放锁之后通知它后继节点需要被唤醒。
3. CONDITION（-2）：当前节点在某个条件队列上等待。
4. PROPAGATE（-3）：只有头节点才会设置为改状态，表明当前处于共享模式中，节点被唤醒之后需要传播唤醒动作，继续唤醒其他的节点。

## API
AQS中已经实现的与加锁和解锁有关的方法如下：

| 方法 | 作用 |
| :-----: | :-----: |
| acquire(int) | 获取独占锁，不可中断，可能线程会进入队列中等待 |
| acquireInterruptibly(int) | 获取独占锁，可以中断 |
| tryAcquireNanos(int, long) | 在指定时间之内尝试获取独占锁 |
| release(int) | 释放独占锁 |
| acquireShared(int) | 获取共享锁，不可中断 |
| acquireSharedInterruptibly(int) | 获取共享锁，可以中断 |
| tryAcquireSharedNanos(int, long) | 在指定时间之内尝试获取共享锁，可以中断 |
| releaseShared(int) | 释放共享锁 |

## 独占锁的获取和释放
### 独占锁的获取
acquire用于获取独占锁，请求不可中断，首先会通过tryAcquire方法获取锁，如果获取失败，则进入等待队列中。tryAcquire在AQS中是一个protected的方法，需要子类去实现具体的乐观获取独占锁的方式：
```
protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
}
```
如果tryAcquire获取失败，则通过addWaiter方法生成一个关联当前线程的waiter节点放入队列中，再通过acquireQueued方法获取锁。

```
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode); // 创建关联当前线程的等待节点
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) { // 通过CAS将节点放入队列的尾部
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}
    
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) { // 循环重试
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) { // head不关联任何线程，是一个dummy节点，如果当前节点的前置节点是head，则通过tryAcquire方法尝试获取锁
                setHead(node); // 获取锁成功，设置当前节点为head，在setHead中会将node的thread和prev指针置为null。因为head节点不关联任何线程
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            // shouldParkAfterFailedAcquire判断当前节点获取锁失败后是否需要挂起当前线程（park），如果需要挂起，则通过parkAndCheckInterrupt方法挂起线程（LockSupport.park），然后清除线程的中断状态(Thread.interrupted)。
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true; // 返回线程已经被中断
        }
    } finally {
        if (failed) // 最后如果失败，则取消锁请求
            cancelAcquire(node);
    }
}
```
acquireQueued方法中是一个死循环，如果判断当前节点是队列中的第一个节点并且通过tryAcquire获取到锁之后通过setHead将当前节点设置为head，在setHead方法中将head的thread和prev指针置为null，因为head不会关联任何线程。如果获取锁失败，则通过shouldParkAfterFailedAcquire判断当前节点获取锁失败后是否需要挂起当前线程，如果需要挂起当前线程，则通过parkAndCheckInterrupt方法来挂起当前线程，等待它的predecessor节点唤醒：
```
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL) // 当前节点的predecessor节点的waitStatus为SIGNAL，当predecessor释放锁时会唤醒当前线程
        /*
         * This node has already set status asking a release
         * to signal it, so it can safely park.
         */
        return true;
    if (ws > 0) { // 找到waitStatus不是CANCELLED的节点
        /*
         * Predecessor was cancelled. Skip over predecessors and
         * indicate retry.
         */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        /*
         * waitStatus must be 0 or PROPAGATE.  Indicate that we
         * need a signal, but don't park yet.  Caller will need to
         * retry to make sure it cannot acquire before parking.
         */
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL); // 设置predecessor的waitStatus为SIGNAL，代表当前节点需要predecessor的signal信号，但是当前线程还未挂起，线程需要在挂起之前需要重试
    }
    return false;
}

```
下面是获取独占锁的流程图：
<div align="center">
    <img src="{{ site.url }}/images/posts/java/获取独占锁.png" alt="获取独占锁"/>
</div>

### 独占锁的释放
```
public final boolean release(int arg) {
    if (tryRelease(arg)) { // tryRelease由子类复写
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h); // 唤醒后继节点
        return true;
    }
    return false;
}

private void unparkSuccessor(Node node) {
    /*
     * If status is negative (i.e., possibly needing signal) try
     * to clear in anticipation of signalling.  It is OK if this
     * fails or if status is changed by waiting thread.
     */
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    /*
     * Thread to unpark is held in successor, which is normally
     * just the next node.  But if cancelled or apparently null,
     * traverse backwards from tail to find the actual
     * non-cancelled successor.
     */
     // 如果当前节点的next节点为空或者next节点等待被取消则从从tail开始寻找可被唤醒的节点（waitStatus <= 0）
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread); // 唤醒线程
}
```
独占锁的释放过程比较简单：
1. 通过tryRelease释放占有的资源，子类需要复写该方法，修改AQS中的state。
2. 唤醒后继等待的节点：找到waitStatus不是CANCELLED的节点，然后通过LockSupport.unpark方法唤醒该线程。

释放独占锁的流程图如下所示：
<div align="center">
    <img src="{{ site.url }}/images/posts/java/释放独占锁.png" alt="释放独占锁"/>
</div>

## 共享锁的获取和释放
### 共享锁的获取
```
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0) // 首先tryAcquireShared，如果成功则直接return，否则doAcquireShared
        doAcquireShared(arg);
}

private void doAcquireShared(int arg) {
    final Node node = addWaiter(Node.SHARED); // 将当前线程包装成Node放入队列中
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) { // 循环重试
            final Node p = node.predecessor(); // 当前节点的前置节点
            if (p == head) { // 如果当前节点的predecessor节点为head，则tryAcquireShared
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r); // 设置当前节点为头节点，并判断后继节点是否需要唤醒
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            // 前面说过这段代码，shouldParkAfterFailedAcquire判断当前节点获取锁失败后是否需要挂起当前线程（park），如果需要挂起，则通过parkAndCheckInterrupt方法挂起线程（LockSupport.park），然后清除线程的中断状态(Thread.interrupted)。
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```
首先通过tryAcquireShared方法来获取共享锁，如果获取成功，则直接返回。否则将当前线程放入等待队列中，循环重试获取锁：如果当前节点的前置节点为head，则通过tryAcquireShared来获取锁，如果获取成功，则设置头节点，并判断后续节点是否需要唤醒；如果获取失败，则判断当前节点是否需要（shouldParkAfterFailedAcquire）挂起(LockSupport.lock)，如果需要挂起线程，则通过LockSupport.park方法将当前线程挂起。
下面是获取共享锁的流程图:
<div align="center">
    <img src="{{ site.url }}/images/posts/java/获取独占锁.png" alt="获取独占锁"/>
</div>

### 释放共享锁
```
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) { // 如果tryReleaseShared成功则直接返回，否则doReleaseShared。tryReleaseShared需要子类复写该方法
        doReleaseShared();
        return true;
    }
    return false;
}
    
private void doReleaseShared() {
    /*
     * Ensure that a release propagates, even if there are other
     * in-progress acquires/releases.  This proceeds in the usual
     * way of trying to unparkSuccessor of head if it needs
     * signal. But if it does not, status is set to PROPAGATE to
     * ensure that upon release, propagation continues.
     * Additionally, we must loop in case a new node is added
     * while we are doing this. Also, unlike other uses of
     * unparkSuccessor, we need to know if CAS to reset status
     * fails, if so rechecking.
     */
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) { // waitStatus为SIGNAL的状态，需要唤醒后续节点
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0)) // CAS充实将当前节点状态设为0
                    continue;            // loop to recheck cases
                unparkSuccessor(h); // 唤醒后继节点
            }
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE)) // 状态为0，没有后续节点需要唤醒，CAS设置状态为PROPAGATE
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
```
doReleaseShared方法中，判断当前节点的waitStatus，如果为SIGNAL，说明需要唤醒后续节点，唤醒之前先CAS重试把节点状态修改为0，然后unparkSuccessor唤醒后续节点。如果当前节点的waitStatus为0，则说明后续没有节点需要唤醒，CAS重试将当前节点waitStatus状态修改为PROPAGATE。
下图是释放共享锁的流程图：
<div align="center">
    <img src="{{ site.url }}/images/posts/java/释放共享锁.png" alt="释放共享锁"/>
</div>

## 超时和中断
AQS支持加锁超时和中断机制，这也是JUC锁相对synchronized锁的一个重要优势。AQS支持独占锁和共享锁的超时和中断，`public final boolean tryAcquireNanos(int arg, long nanosTimeout)`用于在超时时间之内获取独占锁，`public final boolean tryAcquireSharedNanos(int arg, long nanosTimeout)`用于在超时时间之内获取共享锁。
```
public final boolean tryAcquireNanos(int arg, long nanosTimeout)
        throws InterruptedException {
    if (Thread.interrupted()) // 支持响应中断请求，首先判断当前线程是否被中断，如果中断了，则抛出异常，结束
        throw new InterruptedException();
    // tryAcquire需要子类实现，主要的加锁逻辑在doAcquireNanos方法中
    return tryAcquire(arg) ||
        doAcquireNanos(arg, nanosTimeout);
}
```
tryAcquireNanos可以响应中断请求，首先检查线程是否被中断，如果被中断，则直接抛出异常，加锁失败。否则先通过tryAcquire尝试获取锁，如果成功则直接返回。如果失败，则通过doAcquireNanos来加锁。

```
private boolean doAcquireNanos(int arg, long nanosTimeout)
        throws InterruptedException {
    if (nanosTimeout <= 0L)
        return false;
    final long deadline = System.nanoTime() + nanosTimeout; // 计算本次请求的deadline
    final Node node = addWaiter(Node.EXCLUSIVE); // 将当前线程加入等待队列中
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) { // 如果当前节点的predecessor为head，则tryAcquire尝试获取锁，如果成功则设置当前节点为头节点，返回true
                setHead(node); // 设置当前节点为head，设置head节点的thread和prev指针为null
                p.next = null; // help GC
                failed = false;
                return true;
            }
            nanosTimeout = deadline - System.nanoTime(); // 截止deadline之前本次请求还剩余的时间
            if (nanosTimeout <= 0L) // nanosTimeout小于等于0，说明本次deadline已经到了，返回false，加锁失败
                return false;
            // 本次加锁失败，通过shouldParkAfterFailedAcquire判断是否需要挂起当前线程，如果需要挂起向前线程，并且nanosTimeout大于spinForTimeoutThreshold，则挂起当前线程nanosTime。
            // 这里的spinForTimeoutThreshold相当于一个挂起线程的最小时间阈值，如果小于等于这个时间，则直接重试就可以了，而不是挂起线程，因为挂起和唤醒线程是有性能开销的
            if (shouldParkAfterFailedAcquire(p, node) &&
                nanosTimeout > spinForTimeoutThreshold)
                LockSupport.parkNanos(this, nanosTimeout); // 挂起线程
            if (Thread.interrupted()) // 响应中断请求
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```
根据用户设置的超时时间，计算本次加锁的deadline，在循环体中判断当前时间是否已经超过deadline，如果超过返回false，加锁失败。循环体中首先判断当前节点的predecessor是不是head，如果是head，则尝试加锁，如果加锁成功则设置当前节点为head。如果加锁失败，计算本次离deadline还剩多长时间，如果已经到了deadline则返回false，加锁失败。如果还未到deadline，如果shouldParkAfterFailedAcquire为true并且nanosTimeout大于spinForTimeoutThreshold则挂起当前线程。否则先响应中断请求再循环重试。
流程图如下：
<div align="center">
    <img src="{{ site.url }}/images/posts/java/锁中断和超时.png" alt="锁中断和超时"/>
</div>

## 总结
1. AQS是JUC锁的基础，ReentrantLock，ReentrantReadWriteLock，Semaphore，CountDownLatch都用到了AQS作为其同步器。
2. AQS本质是上一个FIFO队列，它使用一个int state来表示当前同步状态，提供了setState，getState和compareAndSetWaitStatus来获取和设置状态。
3. AQS中已经实现了acquire，acquireInterruptibly，tryAcquireNanos，release，acquireShared，acquireSharedInterruptibly，tryAcquireSharedNanos和releaseShared等方法。tryAcquire，tryRelease，tryAcquireShared，tryReleaseShared这些方法在AQS中没有具体的实现（只抛异常），需要子类去覆写。