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

上面例子中我们提交10个工作任务，每个任务大概耗时1000微秒，开启10个线程，并且使用RateLimiter设置了qps为5，一秒内只允许五个并发请求被处理，虽然有10个线程，但是我们设置了qps为5，一秒之内只能有五个并发请求。我们预期的总耗时大概是2000微秒左右，结果为2805和预期的差不多。

## RateLimiter
RateLimiter基于令牌桶算法，它的核心思想主要有：
1. 响应本次请求之后，动态计算下一次可以服务的时间，如果下一次请求在这个时间之前则需要进行等待。SmoothRateLimiter 类中的 nextFreeTicketMicros 就是表示下一次可以响应的时间。例如，如果我们设置QPS为1，本次请求处理完之后，那么下一次最早的能够响应请求的时间一秒钟之后。
2. RateLimiter 的子类 SmoothBursty 支持处理突发流量请求，例如，我们设置QPS为1，在十秒钟之内没有请求，那么令牌桶中会有10个（假设设置的最大令牌数大于10）空闲令牌，如果下一次请求是 `acquire(20)` ，则不需要等待20秒钟，因为令牌桶中已经有10个空闲的令牌。SmoothRateLimiter 类中的 storedPermits 就是用来表示当前令牌桶中的空闲令牌数。 
3. RateLimiter 子类 SmoothWarmingUp 不同于 SmoothBursty ，它存在一个“热身”的概念。它将 storedPermits 分成两个区间值：[0, thresholdPermits) 和 [thresholdPermits, maxPermits]。当请求进来时，如果当前系统处于"cold"的状态，从 [thresholdPermits, maxPermits] 区间去拿令牌，所需要等待的时间会长于从区间 [0, thresholdPermits) 拿相同令牌所需要等待的时间。当请求增多，storedPermits 减少到 thresholdPermits 以下时，此时拿令牌所需要等待的时间趋于稳定。这也就是所谓“热身”的过程。这个过程后面会详细分析。


RateLimiter主要的类的类图如下所示：

![RateLimiter类图]({{ site.url }}/images/posts/java/RateLimiter类图.png)

RateLimiter 是一个抽象类，SmoothRateLimiter 继承自 RateLimiter，不过 SmoothRateLimiter 任然是一个抽象类，SmoothBursty 和 SmoothWarmingUp 才是具体的实现类。

## SmoothRateLimiter主要属性
SmoothRateLimiter 是抽象类，其中定义了一些关键的参数，我们先来看一下这些参数：
```
/**
* The currently stored permits.
*/
double storedPermits;

/**
* The maximum number of stored permits.
*/
double maxPermits;

/**
* The interval between two unit requests, at our stable rate. E.g., a stable rate of 5 permits
* per second has a stable interval of 200ms.
*/
double stableIntervalMicros;

/**
* The time when the next request (no matter its size) will be granted. After granting a request,
* this is pushed further in the future. Large requests push this further than small requests.
*/
private long nextFreeTicketMicros = 0L; // could be either in the past or future
```
storedPermits 表明当前令牌桶中有多少令牌。maxPermits 表示令牌桶最大令牌数目，storedPermits 的取值范围为：[0, maxPermits]。stableIntervalMicros 等于 `1/qps`，它代表系统在稳定期间，两次请求之间间隔的微秒数。例如：如果我们设置的 qps 为5，则 stableIntervalMicros 为200ms。nextFreeTicketMicros 表示系统处理完当前请求后，下一次请求被许可的最短微秒数，如果在这之前有请求进来，则必须等待。

**当我们设置了 qps 之后，系统需要计算一段时间只能能够生成的令牌数目，那么怎么计算呢？一种方式是开启一个后台任务去做计算，但是这样代价未免有点大。RateLimiter 中采取的是另一中惰性计算方式：在每次请求进来的时候先去计算两次请求之间应该生成多少个令牌，这样的好处是省去了后台任务带来的开销。**

## SmoothBursty
### 创建
RateLimiter 中提供了创建 SmoothBursty 的方法：
```
public static RateLimiter create(double permitsPerSecond) {
    return create(permitsPerSecond, SleepingStopwatch.createFromSystemTimer());
}

@VisibleForTesting
static RateLimiter create(double permitsPerSecond, SleepingStopwatch stopwatch) {
    RateLimiter rateLimiter = new SmoothBursty(stopwatch, 1.0 /* maxBurstSeconds */);  // maxBurstSeconds 用于计算 maxPermits
    rateLimiter.setRate(permitsPerSecond); // 设置生成令牌的速率
    return rateLimiter;
}
```
SmoothBursty 的 maxBurstSeconds 构造函数参数主要用于计算 maxPermits ：`maxPermits = maxBurstSeconds * permitsPerSecond;`。

我们再看一下 setRate 的方法，RateLimiter 中 setRate 方法最终后调用 doSetRate 方法，doSetRate 是一个抽象方法，SmoothRateLimiter 抽象类中覆写了 RateLimiter 的 doSetRate 方法：
```
// SmoothRateLimiter类中的doSetRate方法，覆写了 RateLimiter 类中的 doSetRate 方法，此方法再委托下面的 doSetRate 方法做处理。
@Override
final void doSetRate(double permitsPerSecond, long nowMicros) {
    resync(nowMicros);
    double stableIntervalMicros = SECONDS.toMicros(1L) / permitsPerSecond;
    this.stableIntervalMicros = stableIntervalMicros;
    doSetRate(permitsPerSecond, stableIntervalMicros);
}

// SmoothBursty 和 SmoothWarmingUp 类中覆写此方法
abstract void doSetRate(double permitsPerSecond, double stableIntervalMicros);
}
```
SmoothRateLimiter 类的 doSetRate方法中我们着重看一下 **resync** 这个方法：
```
void resync(long nowMicros) {
    // if nextFreeTicket is in the past, resync to now
    if (nowMicros > nextFreeTicketMicros) {
        double newPermits = (nowMicros - nextFreeTicketMicros) / coolDownIntervalMicros();
        storedPermits = min(maxPermits, storedPermits + newPermits);
        nextFreeTicketMicros = nowMicros;
    }
}
```
resync 方法就是 RateLimiter 中惰性计算 storedPermits 的实现。每一次请求来的时候，都会调用到这个方法。这个方法的过程大致如下：
1. 首先判断当前时间是不是大于 nextFreeTicketMicros ，如果是则代表系统已经"coolDown"， 这两次请求之间应该有新的 permit 生成。
2. 计算本次应该新添加的 permit 数量，这里分式的分母是 coolDownIntervalMicros 方法，它是一个抽象方法。在 SmoothBursty 和 SmoothWarmingUp 中分别有不同的实现。SmoothBursty 中返回的是 stableIntervalMicros 也即是 `1 / QPS`。coolDownIntervalMicros 方法在 SmoothWarmingUp 中的实现后面会说到。
3. 计算 storedPermits，这个逻辑比较简单。
4. 设置 nextFreeTicketMicros 为 nowMicros。代表初始化后 RateLimiter 可以立马接收请求。

SmoothRateLimiter 类的 doSetRate方法中计算了 stableIntervalMicros 这个参数，其他的逻辑委托给子类去实现。stableIntervalMicros 代表两个请求之间应该间隔的微秒数。SmoothBursty 类中 doSetRate 方法实现如下：
```
@Override
void doSetRate(double permitsPerSecond, double stableIntervalMicros) {
    double oldMaxPermits = this.maxPermits;
    maxPermits = maxBurstSeconds * permitsPerSecond;
    if (oldMaxPermits == Double.POSITIVE_INFINITY) {
        // if we don't special-case this, we would get storedPermits == NaN, below
        storedPermits = maxPermits;
    } else {
        storedPermits =
                (oldMaxPermits == 0.0)
                        ? 0.0 // initial state
                        : storedPermits * maxPermits / oldMaxPermits;
    }
}
```
上面的流程虽然有点复杂，但是其实也不难理解，主要就是计算了上面说到的 SmoothRateLimiter 类中主要的四个属性：storedPermits, maxPermits, stableIntervalMicros 和 nextFreeTicketMicros。这四个参数贯穿了整个流程。

### tryAcquire方法
tryAcquire 方法用于尝试获取若干个 permit，此方法不会等待，如果获取失败则直接返回失败。canAcquire 方法用于判断当前的请求能否通过：
```
private boolean canAcquire(long nowMicros, long timeoutMicros) {
    return queryEarliestAvailable(nowMicros) - timeoutMicros <= nowMicros;
}

final long queryEarliestAvailable(long nowMicros) {
    return nextFreeTicketMicros;
}
```
此逻辑比较简单，就是看 nextFreeTicketMicros 减去 timeoutMicros 是否小于等于 nowMicros。如果当前需求能被满足，则继续往下走。

接着会调用 SmoothRateLimiter 类的 reserveEarliestAvailable 方法，该方法返回当前请求需要等待的时间。改方法在 acquire 方法中也会用到，我们来着重分析这个方法。

### reserveEarliestAvailable方法
```
final long reserveEarliestAvailable(int requiredPermits, long nowMicros) {
    resync(nowMicros);
    long returnValue = nextFreeTicketMicros;
    double storedPermitsToSpend = min(requiredPermits, this.storedPermits);
    double freshPermits = requiredPermits - storedPermitsToSpend;
    long waitMicros =
        storedPermitsToWaitTime(this.storedPermits, storedPermitsToSpend)
            + (long) (freshPermits * stableIntervalMicros);
    
    this.nextFreeTicketMicros = LongMath.saturatedAdd(nextFreeTicketMicros, waitMicros);
    this.storedPermits -= storedPermitsToSpend;
    return returnValue;
}

/**
* Translates a specified portion of our currently stored permits which we want to spend/acquire,
* into a throttling time. Conceptually, this evaluates the integral of the underlying function we
* use, for the range of [(storedPermits - permitsToTake), storedPermits].
*
* <p>This always holds: {@code 0 <= permitsToTake <= storedPermits}
*/
abstract long storedPermitsToWaitTime(double storedPermits, double permitsToTake);
```
上面的代码是 SmoothRateLimiter 中的具体实现。其主要有以下步骤：
1. resync，这个方法之前已经分析过，这里不再赘述。其主要用来计算当前请求和上次请求之间这段时间需要生成新的 ticket 数量。
2. 对于 requiredPermits ，RateLimiter 将其分为两个部分：storedPermits 和 freshPermits。storedPermits 代表令牌桶中已经存在的令牌，可以直接拿出来用，freshPermits 代表本次请求需要新生成的 ticket 数量。
3. 分别计算 storedPermits 和 freshPermits 拿出来的部分的令牌数所需要的时间，对于 freshPermits 部分的时间比较好计算：直接拿 freshPermits 乘以 
stableIntervalMicros 就可以得到。而对于需要从 storedPermits 中拿出来的部分则计算比较复杂，这个计算逻辑在 storedPermitsToWaitTime 方法中实现。这个方法在 SmoothBursty 和 SmoothWarmingUp 中有不同的实现。storedPermitsToWaitTime 意思就是表示当前请求从 storedPermits 中拿出来的令牌数需要等待的时间，因为 SmoothBursty 中没有“热身”的概念， storedPermits 中有多少个就可以用多少个，不需要等待，因此 storedPermitsToWaitTime 方法在 SmoothBursty 中返回的是0。而在 SmoothWarmingUp 中的实现后面会着重分析。
4. 计算到了本次请求需要等待的时间之后，会将这个时间加到 nextFreeTicketMicros 中去。最后从 storedPermits 减去本次请求从这部分拿走的令牌数量。

### acquire方法
acquire 方法没有等待超时的概念，会一直阻塞直到满足本次请求。
```
public double acquire(int permits) {
    long microsToWait = reserve(permits);
    stopwatch.sleepMicrosUninterruptibly(microsToWait);
    return 1.0 * microsToWait / SECONDS.toMicros(1L);
}
  
final long reserve(int permits) {
    checkPermits(permits);
    synchronized (mutex()) {
      return reserveAndGetWaitLength(permits, stopwatch.readMicros());
    }
}
  
final long reserveAndGetWaitLength(int permits, long nowMicros) {
    long momentAvailable = reserveEarliestAvailable(permits, nowMicros);
    return max(momentAvailable - nowMicros, 0);
}  
  
abstract long reserveEarliestAvailable(int permits, long nowMicros);  
```
acquire 方法最终还是通过 reserveEarliestAvailable 方法来计算本次请求需要等待的时间。这个方法上面已经分析过了，这里就不再过多阐述。

## SmoothWarmingUp
## 创建 




