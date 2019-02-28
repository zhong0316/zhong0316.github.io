---
layout: post
title: Guava RateLimiter限流
categories: [Java, 算法]
description: Guava RateLimiter限流
keywords: RateLimiter, 令牌桶算法, 漏桶算法, 限流, 分布式系统
---

<h1 align="center">Guava RateLimiter限流</h1>
缓存，降级和限流是大型分布式系统中的三把利剑。目前限流主要有漏桶和令牌桶两种算法。
1. 缓存：缓存的目的是减少外部调用，提高系统响速度。俗话说："缓存是网站优化第一定律"。缓存又分为本机缓存和分布式缓存，本机缓存是针对当前JVM实例的缓存，可以直接使用JDK Collection框架里面的集合类或者诸如Google Guava Cache来做本地缓存；分布式缓存目前主要有Memcached,Redis等。
2. 降级：所谓降级是指在系统调用高峰时，优先保证我们的核心服务，对于非核心服务可以选择将其关闭以保证核心服务的可用。例如在淘宝双11时，支付功能是核心，其他诸如用户中心等非核心功能可以选择降级，优先保证交易。
3. 限流：任何系统的性能都有一个上限，当并发量超过这个上限之后，可1能会对系统造成毁灭性地打击。因此在任何时刻我们都必须保证系统的并发请求数量不能超过某个阈值，限流就是为了完成这一目的。

## 限流之漏桶算法
漏桶算法的示意图如下：

![漏桶算法]({{ site.url }}/images/posts/java/漏桶算法.png)

**漏桶算法可以将系统处理请求限定到恒定的速率，当请求过载时，漏桶将直接溢出。漏桶算法假定了系统处理请求的速率是恒定的，但是在现实环境中，往往我们的系统处理请求的速率不是恒定的。漏桶算法无法解决系统突发流量的情况。**

## 限流之令牌桶算法
令牌桶算法相对漏桶算法的优势在于可以处理系统的突发流量，其算法示意图如下所示：

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
1. 响应本次请求之后，动态计算下一次可以服务的时间，如果下一次请求在这个时间之前则需要进行等待。SmoothRateLimiter 类中的 nextFreeTicketMicros 属性表示下一次可以响应的时间。例如，如果我们设置QPS为1，本次请求处理完之后，那么下一次最早的能够响应请求的时间一秒钟之后。
2. RateLimiter 的子类 SmoothBursty 支持处理突发流量请求，例如，我们设置QPS为1，在十秒钟之内没有请求，那么令牌桶中会有10个（假设设置的最大令牌数大于10）空闲令牌，如果下一次请求是 `acquire(20)` ，则不需要等待20秒钟，因为令牌桶中已经有10个空闲的令牌。SmoothRateLimiter 类中的 storedPermits 就是用来表示当前令牌桶中的空闲令牌数。 
3. RateLimiter 子类 SmoothWarmingUp 不同于 SmoothBursty ，它存在一个“热身”的概念。它将 storedPermits 分成两个区间值：[0, thresholdPermits) 和 [thresholdPermits, maxPermits]。当请求进来时，如果当前系统处于"cold"的状态，从 [thresholdPermits, maxPermits] 区间去拿令牌，所需要等待的时间会长于从区间 [0, thresholdPermits) 拿相同令牌所需要等待的时间。当请求增多，storedPermits 减少到 thresholdPermits 以下时，此时拿令牌所需要等待的时间趋于稳定。这也就是所谓“热身”的过程。这个过程后面会详细分析。


RateLimiter主要的类的类图如下所示：

![RateLimiter类图]({{ site.url }}/images/posts/java/RateLimiter类图.png)

RateLimiter 是一个抽象类，SmoothRateLimiter 继承自 RateLimiter，不过 SmoothRateLimiter 仍然是一个抽象类，SmoothBursty 和 SmoothWarmingUp 才是具体的实现类。

## SmoothRateLimiter主要属性
SmoothRateLimiter 是抽象类，其定义了一些关键的参数，我们先来看一下这些参数：
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

**当我们设置了 qps 之后，需要计算某一段时间系统能够生成的令牌数目，那么怎么计算呢？一种方式是开启一个后台任务去做，但是这样代价未免有点大。RateLimiter 中采取的是惰性计算方式：在每次请求进来的时候先去计算上次请求和本次请求之间应该生成多少个令牌。**

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

// SmoothBursty 中对 doSetRate的实现
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
### resync方法
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
resync 方法就是 RateLimiter 中**惰性计算** 的实现。每一次请求来的时候，都会调用到这个方法。这个方法的过程大致如下：
1. 首先判断当前时间是不是大于 nextFreeTicketMicros ，如果是则代表系统已经"cool down"， 这两次请求之间应该有新的 permit 生成。
2. 计算本次应该新添加的 permit 数量，这里分式的分母是 coolDownIntervalMicros 方法，它是一个抽象方法。在 SmoothBursty 和 SmoothWarmingUp 中分别有不同的实现。SmoothBursty 中返回的是 stableIntervalMicros 也即是 `1 / QPS`。coolDownIntervalMicros 方法在 SmoothWarmingUp 中的计算方式为`warmupPeriodMicros / maxPermits`，warmupPeriodMicros 是 SmoothWarmingUp 的“预热”时间。
3. 计算 storedPermits，这个逻辑比较简单。
4. 设置 nextFreeTicketMicros 为 nowMicros。

### tryAcquire方法
tryAcquire 方法用于尝试获取若干个 permit，此方法不会等待，如果获取失败则直接返回失败。canAcquire 方法用于判断当前的请求能否通过：
```
public boolean tryAcquire(int permits, long timeout, TimeUnit unit) {
    long timeoutMicros = max(unit.toMicros(timeout), 0);
    checkPermits(permits);
    long microsToWait;
    synchronized (mutex()) {
        long nowMicros = stopwatch.readMicros();
        if (!canAcquire(nowMicros, timeoutMicros)) { // 首先判断当前超时时间之内请求能否被满足，不能满足的话直接返回失败
            return false;
        } else {
            microsToWait = reserveAndGetWaitLength(permits, nowMicros); // 计算本次请求需要等待的时间，核心方法
        }
    }
    stopwatch.sleepMicrosUninterruptibly(microsToWait);
    return true;
}

final long reserveAndGetWaitLength(int permits, long nowMicros) {
    long momentAvailable = reserveEarliestAvailable(permits, nowMicros);
    return max(momentAvailable - nowMicros, 0);
}

private boolean canAcquire(long nowMicros, long timeoutMicros) {
    return queryEarliestAvailable(nowMicros) - timeoutMicros <= nowMicros;
}

final long queryEarliestAvailable(long nowMicros) {
    return nextFreeTicketMicros;
}
```
canAcquire 方法逻辑比较简单，就是看 nextFreeTicketMicros 减去 timeoutMicros 是否小于等于 nowMicros。如果当前需求能被满足，则继续往下走。

接着会调用 SmoothRateLimiter 类的 reserveEarliestAvailable 方法，该方法返回当前请求需要等待的时间。改方法在 acquire 方法中也会用到，我们来着重分析这个方法。

### reserveEarliestAvailable方法
```
// 计算本次请求需要等待的时间
final long reserveEarliestAvailable(int requiredPermits, long nowMicros) {

    resync(nowMicros); // 本次请求和上次请求之间间隔的时间是否应该有新的令牌生成，如果有则更新 storedPermits
    long returnValue = nextFreeTicketMicros;
    
    // 本次请求的令牌数 requiredPermits 由两个部分组成：storedPermits 和 freshPermits，storedPermits 是令牌桶中已有的令牌
    // freshPermits 是需要新生成的令牌数
    double storedPermitsToSpend = min(requiredPermits, this.storedPermits);
    double freshPermits = requiredPermits - storedPermitsToSpend;
    
    // 分别计算从两个部分拿走的令牌各自需要等待的时间，然后总和作为本次请求需要等待的时间，SmoothBursty 中从 storedPermits 拿走的部分不需要等待时间
    long waitMicros =
        storedPermitsToWaitTime(this.storedPermits, storedPermitsToSpend)
            + (long) (freshPermits * stableIntervalMicros);
            
    // 更新 nextFreeTicketMicros，这里更新的其实是下一次请求的时间，是一种“预消费”
    this.nextFreeTicketMicros = LongMath.saturatedAdd(nextFreeTicketMicros, waitMicros);
    
    // 更新 storedPermits
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
1. resync，这个方法之前已经分析过，这里不再赘述。其主要用来计算当前请求和上次请求之间这段时间需要生成新的 permit 数量。
2. 对于 requiredPermits ，RateLimiter 将其分为两个部分：storedPermits 和 freshPermits。storedPermits 代表令牌桶中已经存在的令牌，可以直接拿出来用，freshPermits 代表本次请求需要新生成的 permit 数量。
3. 分别计算 storedPermits 和 freshPermits 拿出来的部分的令牌数所需要的时间，对于 freshPermits 部分的时间比较好计算：直接拿 freshPermits 乘以 
stableIntervalMicros 就可以得到。而对于需要从 storedPermits 中拿出来的部分则计算比较复杂，这个计算逻辑在 storedPermitsToWaitTime 方法中实现。storedPermitsToWaitTime 方法在 SmoothBursty 和 SmoothWarmingUp 中有不同的实现。storedPermitsToWaitTime 意思就是表示当前请求从 storedPermits 中拿出来的令牌数需要等待的时间，因为 SmoothBursty 中没有“热身”的概念， storedPermits 中有多少个就可以用多少个，不需要等待，因此 storedPermitsToWaitTime 方法在 SmoothBursty 中返回的是0。而它在 SmoothWarmingUp 中的实现后面会着重分析。
4. 计算到了本次请求需要等待的时间之后，会将这个时间加到 nextFreeTicketMicros 中去。最后从 storedPermits 减去本次请求从这部分拿走的令牌数量。
5. reserveEarliestAvailable 方法返回的是本次请求需要等待的时间，该方法中算出来的 waitMicros 按理来说是应该作为返回值的，但是这个方法返回的却是开始时的 nextFreeTicketMicros ，而算出来的 waitMicros 累加到 nextFreeTicketMicros 中去了。这里其实就是“预消费”，让下一次消费来为本次消费来“买单”。

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
SmoothWarmingUp 相对 SmoothBursty 来说主要区别在于 storedPermitsToWaitTime 方法。其他部分原理和 SmoothBursty 类似。
### 创建 
SmoothWarmingUp 是 SmoothRateLimiter 的子类，它相对于 SmoothRateLimiter 多了几个属性：
```
static final class SmoothWarmingUp extends SmoothRateLimiter {
    private final long warmupPeriodMicros;
    /**
     * The slope of the line from the stable interval (when permits == 0), to the cold interval
     * (when permits == maxPermits)
     */
    private double slope;
    private double thresholdPermits;
    private double coldFactor;
    ...
}
```
这四个参数都是和 SmoothWarmingUp 的“热身”（warmup）机制相关。warmup 可以用如下的图来表示：
```
*          ^ throttling
*          |
*    cold  +                  /
* interval |                 /.
*          |                / .
*          |               /  .   ← "warmup period" is the area of the trapezoid between
*          |              /   .     thresholdPermits and maxPermits
*          |             /    .
*          |            /     .
*          |           /      .
*   stable +----------/  WARM .
* interval |          .   UP  .
*          |          . PERIOD.
*          |          .       .
*        0 +----------+-------+--------------→ storedPermits
*          0 thresholdPermits maxPermits
```
上图中横坐标是当前令牌桶中的令牌 storedPermits，前面说过 SmoothWarmingUp 将 storedPermits 分为两个区间：[0, thresholdPermits) 和 [thresholdPermits, maxPermits]。纵坐标是请求的间隔时间，stableInterval 就是 `1 / QPS`，例如设置的 QPS 为1，则 stableInterval 就是200ms，`coldInterval = stableInterval * coldFactor`，这里的 coldFactor "hard-code"写死的是3。

**当系统进入 cold 阶段时，图像会向右移，直到 storedPermits 等于 maxPermits；当系统请求增多，图像会像左移动，直到 storedPermits 为0。**

### storedPermitsToWaitTime方法
上面"矩形+梯形"图像的面积就是 waitMicros 也即是本次请求需要等待的时间。计算过程在 SmoothWarmingUp 类的 storedPermitsToWaitTime 方法中覆写:
```
@Override
long storedPermitsToWaitTime(double storedPermits, double permitsToTake) {
    double availablePermitsAboveThreshold = storedPermits - thresholdPermits;
    long micros = 0;
    // measuring the integral on the right part of the function (the climbing line)
    if (availablePermitsAboveThreshold > 0.0) { // 如果当前 storedPermits 超过 availablePermitsAboveThreshold 则计算从 超过部分拿令牌所需要的时间（图中的 WARM UP PERIOD）
        // WARM UP PERIOD 部分计算的方法，这部分是一个梯形，梯形的面积计算公式是 “（上底 + 下底） * 高 / 2”
        double permitsAboveThresholdToTake = min(availablePermitsAboveThreshold, permitsToTake);
        // TODO(cpovirk): Figure out a good name for this variable.
        double length = permitsToTime(availablePermitsAboveThreshold)
                + permitsToTime(availablePermitsAboveThreshold - permitsAboveThresholdToTake);
        micros = (long) (permitsAboveThresholdToTake * length / 2.0); // 计算出从 WARM UP PERIOD 拿走令牌的时间
        permitsToTake -= permitsAboveThresholdToTake; // 剩余的令牌从 stable 部分拿
    }
    // measuring the integral on the left part of the function (the horizontal line)
    micros += (stableIntervalMicros * permitsToTake); // stable 部分令牌获取花费的时间
    return micros;
}

// WARM UP PERIOD 部分 获取相应令牌所对应的的时间
private double permitsToTime(double permits) {
    return stableIntervalMicros + permits * slope;
}
```

SmoothWarmingUp 类中 storedPermitsToWaitTime 方法将 permitsToTake 分为两部分，一部分从 WARM UP PERIOD 部分拿，这部分是一个梯形，面积计算就是（上底 + 下底）* 高 / 2。另一部分从 stable 部分拿，它是一个长方形，面积就是 长 * 宽。最后返回两个部分的时间总和。

## 总结
1. RateLimiter中采用惰性方式来计算两次请求之间生成多少新的 permit，这样省去了后台计算任务带来的开销。
2. 最终的 requiredPermits 由两个部分组成：storedPermits 和 freshPermits 。SmoothBursty 中 storedPermits 都是一样的，不做区分。而 SmoothWarmingUp 类中将其分成两个区间：[0, thresholdPermits) 和 [thresholdPermits, maxPermits]，存在一个"热身"的阶段，thresholdPermits 是系统 stable 阶段和 cold 阶段的临界点。从 thresholdPermits 右边的部分拿走 permit 需要等待的时间更长。左半部分是一个矩形，由半部分是一个梯形。
3. RateLimiter 能够处理突发流量的请求，采取一种"预消费"的策略。
4. RateLimiter 只能用作单个JVM实例上接口或者方法的限流，不能用作全局流量控制。

## 参考资料
[使用Guava RateLimiter限流以及源码解析](https://segmentfault.com/a/1190000016182737)  
[Guava API文档](https://javadoc.scijava.org/Guava/)