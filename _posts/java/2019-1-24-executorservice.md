---
layout: post
title: ExecutorService shutdown()和shutdownNow()方法区别
categories: [Java]
description: ExecutorService shutdown()和shutdownNow()方法区别
keywords: executorservice, 线程池
---

<h1 text-align="center">ExecutorService shutdown()和shutdownNow()方法区别</h1>
ExecutorService是我们经常使用的线程池，当我们使用完线程池后，需要关闭线程池。ExecutorService的shutdown()和shutdownNow()方法都可以用来关闭线程池，那么他们有什么区别呢？

废话不多说，直接查看源码，shutdown方法：

```  
/**
      * Initiates an orderly shutdown in which previously submitted
      * tasks are executed, but no new tasks will be accepted.
      * Invocation has no additional effect if already shut down.
      *
      * <p>This method does not wait for previously submitted tasks to
      * complete execution.  Use {@link #awaitTermination awaitTermination}
      * to do that.
      *
      * @throws SecurityException if a security manager exists and
      *         shutting down this ExecutorService may manipulate
      *         threads that the caller is not permitted to modify
      *         because it does not hold {@link
      *         java.lang.RuntimePermission}{@code ("modifyThread")},
      *         or the security manager's {@code checkAccess} method
      *         denies access.
      */
     void shutdown();
```
当我们调用shutdown()方法后，线程池会等待我们已经提交的任务执行完成。但是此时线程池不再接受新的任务，如果我们再向线程池中提交任务，将会抛RejectedExecutionException异常。如果线程池的shutdown()方法已经调用过，重复调用没有额外效应。注意，当我们调用shutdown()方法后，会立即从该方法中返回而不会阻塞等待线程池关闭再返回，如果希望阻塞等待可以调用awaitTermination()方法。

再看shutdownNow()方法:
```
/**
     * Attempts to stop all actively executing tasks, halts the
     * processing of waiting tasks, and returns a list of the tasks
     * that were awaiting execution.
     *
     * <p>This method does not wait for actively executing tasks to
     * terminate.  Use {@link #awaitTermination awaitTermination} to
     * do that.
     *
     * <p>There are no guarantees beyond best-effort attempts to stop
     * processing actively executing tasks.  For example, typical
     * implementations will cancel via {@link Thread#interrupt}, so any
     * task that fails to respond to interrupts may never terminate.
     *
     * @return list of tasks that never commenced execution
     * @throws SecurityException if a security manager exists and
     *         shutting down this ExecutorService may manipulate
     *         threads that the caller is not permitted to modify
     *         because it does not hold {@link
     *         java.lang.RuntimePermission}{@code ("modifyThread")},
     *         or the security manager's {@code checkAccess} method
     *         denies access.
     */
    List<Runnable> shutdownNow();
```
首先shutdownNow()方法和shutdown()方法一样，当我们调用了shutdownNow()方法后，调用线程立马从该方法返回，而不会阻塞等待。这也算shutdownNow()和shutdown()方法的一个相同点。与shutdown()方法不同的是shutdownNow()方法调用后，线程池会通过调用worker线程的interrupt方法尽最大努力(best-effort)去"终止"已经运行的任务。
而对于那些在堵塞队列中等待执行的任务，线程池并不会再去执行这些任务，而是直接返回这些等待执行的任务，也就是该方法的返回值。
值得注意的是，当我们调用一个线程的interrupt()方法后（前提是caller线程有权限，否则抛异常），该线程并不一定会立马退出：
1. 如果线程处于被阻塞状态（例如处于sleep, wait, join 等状态），那么线程立即退出被阻塞状态，并抛出一个InterruptedException异常。
2. 如果线程处于正常的工作状态，该方法只会设置线程的一个状态位为true而已，线程会继续执行不受影响。如果想停止线程运行可以在任务中检查当前线程的状态(Thread.isInterrupted())自己实现停止逻辑。