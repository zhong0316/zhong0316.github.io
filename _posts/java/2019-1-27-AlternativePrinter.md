---
layout: post
title: 如何让两个线程交替打印数字
categories: [Java, 算法]
description: 如何让两个线程交替打印数字
keywords: 线程, 线程通信, 同步 
---

<h1 align="center">如何让两个线程交替打印数字</h1>

## 问题
如何让两个线程交替打印1-100的数字？废话不多说，直接上代码：
## synchronized锁+AtomicInteger
```
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.atomic.AtomicInteger;

public class StrangePrinter {

    private int max;
    private AtomicInteger status = new AtomicInteger(1); // AtomicInteger保证可见性，也可以用volatile

    public StrangePrinter(int max) {
        this.max = max;
    }

    public static void main(String[] args) {
        StrangePrinter strangePrinter = new StrangePrinter(100);
        ExecutorService executorService = Executors.newFixedThreadPool(2);
        executorService.submit(strangePrinter.new MyPrinter("Print1", 0));
        executorService.submit(strangePrinter.new MyPrinter("Print2", 1));
        executorService.shutdown();
    }

    class MyPrinter implements Runnable {
        private String name;
        private int type; // 打印的类型，0：代表打印奇数，1：代表打印偶数

        public MyPrinter(String name, int type) {
            this.name = name;
            this.type = type;
        }

        @Override
        public void run() {
            if (type == 1) {
                while (status.get() <= max) {
                    synchronized (StrangePrinter.class) { // 加锁，保证下面的操作是一个原子操作
                        // 打印偶数
                        if (status.get() <= max && status.get() % 2 == 0) { // 打印偶数
                            System.out.println(name + " - " + status.getAndIncrement());
                        }
                    }
                }
            } else {
                while (status.get() <= max) {
                    synchronized (StrangePrinter.class) { // 加锁
                        // 打印奇数
                        if (status.get() <= max && status.get() % 2 != 0) { // 打印奇数
                            System.out.println(name + " - " + status.getAndIncrement());
                        }
                    }
                }
            }
        }
    }
}

```
这里需要注意两点：
1. 用AtomicInteger保证多线程数据可见性。
2. 不要觉得synchronized加锁是多余的，如果没有加锁，线程1和线程2就可能出现不是交替打印的情况。如果没有加锁，设想线程1打印完了一个奇数后，线程2去打印下一个偶数，当执行完`status.getAndIncrement()`后，此时status又是奇数了，当此时cpu将线程2挂起，调度线程1，就会出现线程2还没来得及打印偶数，线程1就已经打印了下一个奇数的情况。就不符合题目要求了。因此这里加锁是必须的，保证代码块中的是一个原子操作。

## 使用Object的wait和notify实现
```
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.atomic.AtomicInteger;

public class StrangePrinter2 {

    Object odd = new Object(); // 奇数条件锁
    Object even = new Object(); // 偶数条件锁
    private int max;
    private AtomicInteger status = new AtomicInteger(1); // AtomicInteger保证可见性，也可以用volatile

    public StrangePrinter2(int max) {
        this.max = max;
    }

    public static void main(String[] args) {
        StrangePrinter2 strangePrinter = new StrangePrinter2(100);
        ExecutorService executorService = Executors.newFixedThreadPool(2);
        executorService.submit(strangePrinter.new MyPrinter("偶数Printer", 0));
        executorService.submit(strangePrinter.new MyPrinter("奇数Printer", 1));
        executorService.shutdown();
    }

    class MyPrinter implements Runnable {
        private String name;
        private int type; // 打印的类型，0：代表打印奇数，1：代表打印偶数

        public MyPrinter(String name, int type) {
            this.name = name;
            this.type = type;
        }

        @Override
        public void run() {
            if (type == 1) {
                while (status.get() <= max) { // 打印奇数
                    if (status.get() % 2 == 0) { // 如果当前为偶数，则等待
                        synchronized (odd) {
                            try {
                                odd.wait();
                            } catch (InterruptedException e) {
                                e.printStackTrace();
                            }
                        }
                    } else {
                        System.out.println(name + " - " + status.getAndIncrement()); // 打印奇数
                        synchronized (even) { // 通知偶数打印线程
                            even.notify();
                        }
                    }
                }
            } else {
                while (status.get() <= max) { // 打印偶数
                    if (status.get() % 2 != 0) { // 如果当前为奇数，则等待
                        synchronized (even) {
                            try {
                                even.wait();
                            } catch (InterruptedException e) {
                                e.printStackTrace();
                            }
                        }
                    } else {
                        System.out.println(name + " - " + status.getAndIncrement()); // 打印偶数
                        synchronized (odd) { // 通知奇数打印线程
                            odd.notify();
                        }
                    }
                }
            }
        }
    }
}

```

## 使用ReentrantLock+Condition实现
```
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

public class StrangePrinter3 {

    private int max;
    private AtomicInteger status = new AtomicInteger(1); // AtomicInteger保证可见性，也可以用volatile
    private ReentrantLock lock = new ReentrantLock();
    private Condition odd = lock.newCondition();
    private Condition even = lock.newCondition();

    public StrangePrinter3(int max) {
        this.max = max;
    }

    public static void main(String[] args) {
        StrangePrinter3 strangePrinter = new StrangePrinter3(100);
        ExecutorService executorService = Executors.newFixedThreadPool(2);
        executorService.submit(strangePrinter.new MyPrinter("偶数Printer", 0));
        executorService.submit(strangePrinter.new MyPrinter("奇数Printer", 1));
        executorService.shutdown();
    }

    class MyPrinter implements Runnable {
        private String name;
        private int type; // 打印的类型，0：代表打印奇数，1：代表打印偶数

        public MyPrinter(String name, int type) {
            this.name = name;
            this.type = type;
        }

        @Override
        public void run() {
            if (type == 1) {
                while (status.get() <= max) { // 打印奇数
                    lock.lock();
                    try {
                        if (status.get() % 2 == 0) {
                            odd.await();
                        }
                        if (status.get() <= max) {
                            System.out.println(name + " - " + status.getAndIncrement()); // 打印奇数
                        }
                        even.signal();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    } finally {
                        lock.unlock();
                    }
                }
            } else {
                while (status.get() <= max) { // 打印偶数
                    lock.lock();
                    try {
                        if (status.get() % 2 != 0) {
                            even.await();
                        }
                        if (status.get() <= max) {
                            System.out.println(name + " - " + status.getAndIncrement()); // 打印偶数
                        }
                        odd.signal();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    } finally {
                        lock.unlock();
                    }
                }
            }
        }
    }
}

```
这里的实现思路其实和使用Object的wait和notify机制差不多。

## 通过flag标识实现
```
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.atomic.AtomicInteger;

public class StrangePrinter4 {

    private int max;
    private AtomicInteger status = new AtomicInteger(1); // AtomicInteger保证可见性，也可以用volatile
    private boolean oddFlag = true;

    public StrangePrinter4(int max) {
        this.max = max;
    }

    public static void main(String[] args) {
        StrangePrinter4 strangePrinter = new StrangePrinter4(100);
        ExecutorService executorService = Executors.newFixedThreadPool(2);
        executorService.submit(strangePrinter.new MyPrinter("偶数Printer", 0));
        executorService.submit(strangePrinter.new MyPrinter("奇数Printer", 1));
        executorService.shutdown();
    }

    class MyPrinter implements Runnable {
        private String name;
        private int type; // 打印的类型，0：代表打印奇数，1：代表打印偶数

        public MyPrinter(String name, int type) {
            this.name = name;
            this.type = type;
        }

        @Override
        public void run() {
            if (type == 1) {
                while (status.get() <= max) { // 打印奇数
                    if (oddFlag) {
                        System.out.println(name + " - " + status.getAndIncrement()); // 打印奇数
                        oddFlag = !oddFlag;
                    }
                }
            } else {
                while (status.get() <= max) { // 打印偶数
                    if (!oddFlag) {
                        System.out.println(name + " - " + status.getAndIncrement()); // 打印奇数
                        oddFlag = !oddFlag;
                    }
                }
            }
        }
    }
}

```
这是最简单最高效的实现方式，因为不需要加锁。比前面两种实现方式都要好一些。
