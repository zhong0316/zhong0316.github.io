---
layout: post
title: Guava RateLimiter限流
categories: [Java, 算法]
description: Guava RateLimiter限流
keywords: RateLimiter, 令牌桶算法, 漏斗算法, 限流, 分布式系统
---

<h1 align="center">Guava RateLimiter限流</h1>
缓存，降级和限流是大型分布式系统中的三把利剑。目前限流主要有漏斗和令牌桶两种算法。
1. 缓存：缓存的目的是减少外部调用，提高系统响速度。俗话说："缓存是网站优化第一定律"。缓存又分为本机缓存和分布式缓存，本机缓存是针对当前JVM实例的缓存，可以直接使用JDK Collection框架里面的集合类或者诸如Google Guava Cache来做本地缓存；分布式缓存目前主要有MemCached,Redis等。
2. 降级：所谓降级是指在系统调用高峰时，优先保证我们的核心服务，对于非核心服务可以选择将其关闭以保证核心服务的可用。例如在淘宝双11时，支付功能是核心，其他诸如用户中心等非核心功能可以选择降级，优先保证交易。
3. 限流：任何系统的性能都有一个上限，当并发量超过这个上限之后，可能会对系统造成毁灭性地打击。因此在任何时刻我们都必须保证系统的并发请求数量不能超过某个阈值，限流就是为了完成这一目的。

## 限流之漏斗算法
漏斗算法的示意图如下：

![漏斗算法]({{ site.url }}/images/posts/java/漏斗算法.png)

**漏斗算法可以将系统处理请求限定到恒定的速率，当请求过载时，漏斗将直接溢出。漏斗算法假定了系统处理请求的速率是恒定的，但是在现实环境中，往往我们的系统处理请求的速率不是恒定的。漏斗算法无法解决系统突发流量的情况。**

## 限流之令牌桶算法
令牌桶算法相对漏斗算法的优势在于可以处理系统的突发流量，其算法示意图如下所示：

![令牌桶算法]({{ site.url }}/images/posts/java/令牌桶算法.png)

**令牌桶有一定的容量（capacity），后台服务向令牌桶中以恒定的速率放入令牌（token），当令牌桶中的令牌数量超过capacity之后，多余的令牌直接丢弃。当一个请求进来时，需要从桶中拿到N个令牌，如果能够拿到则继续后面的处理流程，如果拿不到，则当前线程可以选择阻塞等待桶中的令牌数量够本次请求的数量或者不等待直接返回失败。**

## Guava RateLimiter限流
Guava RateLimiter是一个谷歌提供的限流工具，RateLimiter基于令牌桶算法，可以有效限定单个JVM实例上某个接口的流量。

### RateLimiter使用的一个例子
```
import com.google.common.util.concurrent.RateLimiter;

import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class RateLimiterExample {

    public static void main(String[] args) throws InterruptedException {
        // qps设置为5，代表一秒钟只允许处理五个并发请求
        RateLimiter rateLimiter = RateLimiter.create(5);
        ExecutorService executorService = Executors.newFixedThreadPool(5);
        int nTasks = 10;
        CountDownLatch countDownLatch = new CountDownLatch(nTasks);
        long start = System.currentTimeMillis();
        for (int i = 0; i < nTasks; i++) {
            final int j = i;
            executorService.submit(() -> {
                rateLimiter.acquire(1);
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                }
                System.out.println(Thread.currentThread().getName() + " gets job " + j + " done");
                countDownLatch.countDown();
            });
        }
        executorService.shutdown();
        countDownLatch.await();
        long end = System.currentTimeMillis();
        System.out.println("10 jobs gets done by 5 threads concurrently in " + (end - start) + " milliseconds");
    }
}

```

**输出结果**:
```
pool-1-thread-1 gets job 0 done
pool-1-thread-2 gets job 1 done
pool-1-thread-3 gets job 2 done
pool-1-thread-4 gets job 3 done
pool-1-thread-5 gets job 4 done
pool-1-thread-6 gets job 5 done
pool-1-thread-7 gets job 6 done
pool-1-thread-8 gets job 7 done
pool-1-thread-9 gets job 8 done
pool-1-thread-10 gets job 9 done
10 jobs gets done by 5 threads concurrently in 2805 milliseconds
```

上面例子中我们提交10个工作任务，每个任务大概耗时1000毫秒，开启10个线程，并且使用RateLimiter设置了qps为5，一秒内只允许五个并发请求被处理，虽然有10个线程，但是我们设置了qps为5，一秒之内只能有五个并发请求。我们预期的总耗时大概是2000毫秒左右，结果为2805和预期的差不多。

## RateLimiter源码分析
RateLimiter基于令牌桶算法，它的核心思想主要有：
1. 响应本次请求之后，动态计算下一次可以服务的时间，如果下一次请求在这个时间之前则需要进行等待。SmoothRateLimiter 类中的 nextFreeTicketMicros 就是表示下一次可以响应的时间。例如，如果我们设置QPS为1，本次请求处理完之后，那么下一次最早的能够响应请求的时间一秒钟之后。
2. RateLimiter 的子类 SmoothBursty 支持处理突发流量请求，例如，我们设置QPS为1，在十秒钟之内没有请求，那么令牌桶中会有10个（假设设置的最大令牌数大于10）空闲令牌，如果下一次请求是 `acquire(20)` ，则不需要等待20秒钟，因为令牌桶中已经有10个空闲的令牌。SmoothRateLimiter 类中的 storedPermits 就是用来表示当前令牌桶中的空闲令牌数。 
3. 


RateLimiter主要的类的类图如下所示：

![RateLimiter类图]({{ site.url }}/images/posts/java/RateLimiter类图.png)

RateLimiter是一个抽象类，主要定义了一些相关的方法：

![RateLimiter主要方法]({{ site.url }}/images/posts/java/RateLimiter主要方法.png)


