---
layout: post
title: Java中断机制
categories: [Java]
description: Java中断机制
keywords: Java, 中断, 线程
---

<h1 align="center">Java中断机制</h1>

## 引言  
Java中断机制为我们提供了一种"试图"停止一个线程的方法。设想我们有一个线程阻塞在一个耗时的I/O中，我们又不想一直等下去，那么我们怎么样才能停止这个线程呢？答案就是Java的中断机制。
## 从Java线程的状态说起   
Java线程的状态包括：NEW，RUNNABLE，BLOCKED，WAITING，TIMED_WAITING和TERMINATED总共六种状态:
```
    public enum State {
        /**
         * Thread state for a thread which has not yet started.
         */
        NEW,

        /**
         * Thread state for a runnable thread.  A thread in the runnable
         * state is executing in the Java virtual machine but it may
         * be waiting for other resources from the operating system
         * such as processor.
         */
        RUNNABLE,

        /**
         * Thread state for a thread blocked waiting for a monitor lock.
         * A thread in the blocked state is waiting for a monitor lock
         * to enter a synchronized block/method or
         * reenter a synchronized block/method after calling
         * {@link Object#wait() Object.wait}.
         */
        BLOCKED,

        /**
         * Thread state for a waiting thread.
         * A thread is in the waiting state due to calling one of the
         * following methods:
         * <ul>
         *   <li>{@link Object#wait() Object.wait} with no timeout</li>
         *   <li>{@link #join() Thread.join} with no timeout</li>
         *   <li>{@link LockSupport#park() LockSupport.park}</li>
         * </ul>
         *
         * <p>A thread in the waiting state is waiting for another thread to
         * perform a particular action.
         *
         * For example, a thread that has called <tt>Object.wait()</tt>
         * on an object is waiting for another thread to call
         * <tt>Object.notify()</tt> or <tt>Object.notifyAll()</tt> on
         * that object. A thread that has called <tt>Thread.join()</tt>
         * is waiting for a specified thread to terminate.
         */
        WAITING,

        /**
         * Thread state for a waiting thread with a specified waiting time.
         * A thread is in the timed waiting state due to calling one of
         * the following methods with a specified positive waiting time:
         * <ul>
         *   <li>{@link #sleep Thread.sleep}</li>
         *   <li>{@link Object#wait(long) Object.wait} with timeout</li>
         *   <li>{@link #join(long) Thread.join} with timeout</li>
         *   <li>{@link LockSupport#parkNanos LockSupport.parkNanos}</li>
         *   <li>{@link LockSupport#parkUntil LockSupport.parkUntil}</li>
         * </ul>
         */
        TIMED_WAITING,

        /**
         * Thread state for a terminated thread.
         * The thread has completed execution.
         */
        TERMINATED;
    }
```
一个线程新建后，在调用该线程的start()方法之前，该线程的状态就是NEW；当线程的start()方法被调用之后，该线程已经准备被CPU调度，此时线程的状态就是RUNNABLE；如果一个对象的monitor lock被其他线程获取了，这个时候我们再通过synchronized关键字去获取改对象的monitor lock时，线程就会进入BLOCKED状态；通过调用Thread#join()方法或者Object#wait()方法（不设置超时时间，with no timeout）或者LockSupport#park()方法可以让一个线程从RUNNABLE状态转为WAITING状态；TIMED_WAITING指线程处于等待中，但是这个等待是有期限的（），通过调用Thread#sleep()，Object#wait(long timeout)，Thread#join(long timeout)，LockSupport#parkNanos()，LockSupport#partUnitl()都可使线程切换到TIMED_WAITING状态。最后一种状态TERMINATED是指线程正常退出或者异常退出后的状态。下面这张图展示了这六种线程状态之间的相互转换关系：

<div align="center">
    <img src="{{ site.url }}/images/posts/java/threadstatus.jpeg" alt="线程状态图"/>
</div>

当线程处于等待状态或者有超时的等待状态时（TIMED_WAITING，WAITING）我们可以通过调用线程的interrupt()方法来中断线程的等待，此时线程会抛InterruptedException异常。例如Thread.sleep()方法:
`public static native void sleep(long millis) throws InterruptedException;`
此方法就会抛出这类异常。
但是当线程处于BLOCKED状态或者RUNNABLE（RUNNING）状态时，调用线程的interrupt()方法也只能将线程的状态位设置为true。停止线程的逻辑需要我们自己去实现。

## 中断原理  
查看java源码可以看到Thread类中提供了跟中断相关的一些方法：
```
    public void interrupt() {
        if (this != Thread.currentThread())
            checkAccess();                                                 1

        synchronized (blockerLock) {
            Interruptible b = blocker;
            if (b != null) {                                               2
                interrupt0();           // Just to set the interrupt flag  3
                b.interrupt(this);                                         4
                return;
            }
        }
        interrupt0();
    }
```
interrupt()方法用于中断一个线程，首先在第1处中断方法会判断当前caller线程是否具有中断本线程的权限，如果没有权限则抛出SecurityException异常。然后在2处方法会判断阻塞当前线程的对象是否为空，如果不为空，则在3处先设置线程的中断flag为true，然后再由Interruptible对象（例如可以中断的I/O操作）去中断操作。从中我们也可以看到如果当前线程不处于WAITING或者TIMED_WAITING状态，则interrupt()方法也只是仅仅设置了该线程的中断flag为true而已。
```
    public static boolean interrupted() {
        return currentThread().isInterrupted(true);
    }
```
再看interrupted()方法，该方法是一个静态的方法，最终会调用具体线程实例的isInterrupted()方法：
```
    public boolean isInterrupted() {
        return isInterrupted(false);
    }
    
    private native boolean isInterrupted(boolean ClearInterrupted);
```
interrupted()方法第一次调用时会返回当前线程的中断flag，然后就清除了这个状态位，但是isInterrupted()方法不会清除该状态位。因此一个线程中断后，第一次调用interrupted()方法会返回true，第二次调用就返回false，因此第一次调用后该状态位已经被清除了。但是同样一个线程中断后，多次调用isInterrupted()方法都会返回true。这是两者的不同之处，但是两者最终都会调用一个名为isInterrupted的native方法。

## 中断的处理  
java中一般方法申明了throws InterruptedException的就是可以中断的方法，比如：ReentrantLock#lockInterruptibly，BlockingQueue#put，Thread#sleep，Object#wait等。当这些方法被中断后会抛出InterruptedException异常，我们可以捕获这个异常，并实现自己的处理逻辑，也可以不处理，继续向上层抛出异常。但是记住不要捕获异常后什么都不做并且不向上层抛异常，也就是说我们不能"**吞掉**"异常。
对于一些没有抛出InterruptedException的方法的中断逻辑只能由我们自己去实现了。例如在一个大的循环中，我们可以自己判断当前线程的中断状态然后选择是否中断当前操作：
```
public class InterruptTest {

    public static void main(String[] args) throws InterruptedException {
        Thread thread = new Thread(new Runnable() {
            @Override
            public void run() {
                // 循环中检测当前线程的中断状态 
                for (int i = 0; i < Integer.MAX_VALUE && !Thread.currentThread().isInterrupted(); i++) {
                    System.out.println(i);
                }
            }
        });
        thread.start();
        Thread.sleep(100);
        thread.interrupt();
    }
}
```