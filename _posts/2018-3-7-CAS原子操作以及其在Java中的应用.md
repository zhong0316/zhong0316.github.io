---
layout: post
title: CAS原子操作以及其在Java中的应用
categories: [Java, 锁]
description: CAS原子操作以及其在Java中的应用
keywords: CAS, 乐观锁, 悲观锁
---

<h1 align="center">CAS原子操作以及其在Java中的应用</h1>
CAS（Compare And Set）意为比较并且交换，CAS它是一个原子操作。<font color="#FF0000">CAS操作涉及到三个值：当前内存中的值V，逾期内存中的值E和待更新的值U。如果当前内存中的值V等于预期值E，则将内存中的值更新为U，CAS操作成功。否则不更新CAS操作失败。</font>
CAS在JUC中有广泛的运用，可以说CAS是JUC的基础，没有CAS操作就没有JUC。CAS经常用来实现Java中的乐观锁，相对于Java中的悲观锁synchronized锁，乐观锁不需要挂起和唤醒线程，在高并发情况下，线程频繁挂起和唤醒会影响性能。为了弄清CAS操作，有必要先了解一下乐观锁和悲观锁以及它们之间的区别。

## 悲观锁和乐观锁
悲观锁认为其他线程的每一次操作都会更新共享变量，因此所有的操作必须互斥，通过悲观锁策略来让所有的操作串行化，所有的线程操作共享变量之前必须获取悲观锁，获取成功则进行操作，获取失败就阻塞当前线程悲观等待。当线程被阻塞住之后CPU将不再调度线程，在高并发的场景下如果线程激烈竞争某一个锁造成线程频繁挂起和唤醒，无疑将给我们的应用带来灾难性的打击。悲观锁的流程图如下：
<div align="center">
    <img src="{{ site.url }}/images/posts/java/悲观锁.png" alt="悲观锁"/>
</div> 
Java中的synchronized锁和ReentrantLock都是悲观的锁，在前面的文章中分析了Java中各种锁的区别，有兴趣的可以前去查看。[多线程安全性和Java中的锁]({{ site.url }}/2018/01/27/lock)

乐观锁相对悲观锁来说则是认为其他线程不一定会修改共享变量，因此线程不必阻塞等待。通过CAS来更新共享变量，如果CAS更新失败则证明其他线程修改了这个共享变量，自己循环重试直到更新成功就可以了。因为CAS操作不会挂起线程因此减少了线程挂起和唤醒的开销，在高并发情况下这个节省是非常可观的。循环重试虽然不会挂起线程但是会消耗CPU，因为线程需要一直循环重试，这也是CAS乐观锁的一个缺点。`java.util.concurrent.atomic`包下面的Atomic类都是通过CAS乐观锁来保证线程安全性的。CAS乐观锁还有另一个缺点就是无法解决“ABA”问题。这个后面会进行详细的分析。
基于CAS乐观锁流程图如下所示：

<div align="center">
    <img src="{{ site.url }}/images/posts/java/基于CAS的乐观锁.png" alt="基于CAS的乐观锁"/>
</div> 

## CAS操作以及在Java中的应用
CAS在Java中有很多应用，JUC以及`java.util.concurrent.atomic`下面的原子类都用到了CAS。
### Unsafe提供的CAS操作
Unsafe为Java提供了很多底层功能，其中Java中的CAS功能就是通过这个类来实现的。有兴趣的可以查看我前面专门讲解Unsafe这个类的文章：[Java中的Unsafe]({{ site.url }}/2018/03/05/Java中的Unsafe)。

```
public final native boolean compareAndSwapObject(Object var1, long var2, Object var4, Object var5);

public final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5);

public final native boolean compareAndSwapLong(Object var1, long var2, long var4, long var6);
```
Unsafe中提供了Object，int和long类型的CAS操作。其他类型的需要自己实现。CAS它是一个原子操作。要保证线程安全性，除了原子性，还有可见性和有序性。可见性和有序性在Java中都可以通过volatile来实现。

### Java中的原子类（`java.util.concurrent.atomic`）
`java.util.concurrent.atomic`包中的类通过volatile+CAS重试保证线程安全性。
`java.util.concurrent.atomic`包下面的原子类可以分为四种类型：
1. 原子标量：AtomicBoolean，AtomicInteger，AtomicLong，AtomicReference
2. 数组类：AtomicIntegerArray，AtomicLongArray，AtomicReferenceArray
3. 更新类：AtomicLongFieldUpdater，AtomicIntegerFieldUpdater，AtomicReferenceFieldUpdater
4. 复合变量类：AtomicMarkableReference，AtomicStampedReference

### 原子标量类
我们主要分析一下AtomicInteger这个类如何保证线程安全性：

**AtomicInteger的value是volatile的**
```
public class AtomicInteger extends Number implements java.io.Seriablizable {
    ...
    private volatile int value; // value是volatile的，保证了可见性和有序性
    ...
}
```
AtomicInteger中的value是volatile的，volatile可以保证可见性和有序性。

**get操作**
```
public final int get() {
    return value;
}
```
可以看到AtomicInteger的get操作是不加锁的，对于非volatile类型的共享变量，并发操作时，一个读线程未必能立马读取到其他线程对这个共享变量的修改。但是这里的value是volatile的，因此可以立马看到其他线程对value的修改。

**incrementAndGet操作**
```
public final int incrementAndGet() {
    return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
}
```
incrementAndGet操作会先将value加1，然后返回新的值。这个方法内部会调用Unsafe的getAndAddInt方法：
```
public final int getAndAddInt(Object var1, long var2, int var4) {
    int var5;
    do {
        var5 = this.getIntVolatile(var1, var2);
    } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4)); // CAS原子更新+循环重试

    return var5;
}
```
可以看到Unsafe中在循环体内先读取内存中的value值，然后CAS更新，如果CAS更新成功则退出，如果更新失败，则循环重试直到更新成功。

在前面的文章中([Java中的Unsafe]({{ site.url }}/2018/03/05/Java中的Unsafe))我们分析了Unsafe中提供了三种类型对象的CAS操作：Object，int和long类型。AtomicLong是通过Unsafe提供的long类型的CAS操作实现的，AtomicReference是通过Unsafe提供的Object类型的CAS操作实现的，而AtomicBoolean中的value也是一个int类型，AtomicBoolean对int做了一个转换：
```
public AtomicBoolean(boolean initialValue) {
    value = initialValue ? 1 : 0;
}
```
1表示true，0表示false。因此AtomicBoolean也是通过Unsafe提供的int类型的CAS操作来实现的。

### 数组类
在[Java中的Unsafe]({{ site.url }}/2018/03/05/Java中的Unsafe)中我们分析了Unsafe提供了数组相关的两个主要功能：
```
public native int arrayBaseOffset(Class<?> var1);

public native int arrayIndexScale(Class<?> var1);
```
AtomicIntegerArray主要就是使用这两个方法来实现数组CAS操作。AtomicIntegerArray中主要的一些功能如下：
```
private static final Unsafe unsafe = Unsafe.getUnsafe();  
private static final int base = unsafe.arrayBaseOffset(int[].class);  // 获取数组中第一个元素实际地址相对整个数组对象的地址的偏移量
private static final int scale = unsafe.arrayIndexScale(int[].class); // 获取数组中第一个元素所占用的内存空间
private final int[] array;  
public final int get(int i) {  
    return unsafe.getIntVolatile(array, rawIndex(i));  
}  
public final void set(int i, int newValue) {  
    unsafe.putIntVolatile(array, rawIndex(i), newValue);  
}
```
AtomicLongArray和AtomicReferenceArray实现原理和AtomicIntegerArray类似。

### 更新类
AtomicLongFieldUpdater用来更新一个对象的 `volatile long` 类型的属性，这个属性是实例的属性而不是类的属性，也就是说只能用来更新一个对象的实例的非static属性。
与AtomicLongFieldUpdater类似AtomicIntegerFieldUpdater用来更新一个对象的 `volatile int` 类型的属性。

AtomicLongFieldUpdater和AtomicIntegerFieldUpdater都是用来更新一个对象的原生属性(int long)，而AtomicReferenceFieldUpdater用来更新一个对象的包装类型属性。

### 复合变量类
基于CAS乐观锁无法解决[“ABA”问题](https://en.wikipedia.org/wiki/ABA_problem)。解决“ABA”问题的主要思路就是给value打戳，AtomicStampedReference就是通过对值加一个戳(stamp)来解决“ABA”问题的。
```
public class AtomicStampedReference<V> {

    private static class Pair<T> {
        final T reference; // 值
        final int stamp; // 值的戳
        private Pair(T reference, int stamp) {
            this.reference = reference;
            this.stamp = stamp;
        }
        static <T> Pair<T> of(T reference, int stamp) {
            return new Pair<T>(reference, stamp);
        }
    }

    private volatile Pair<V> pair; // volatile保证可见性和有序性
    ...
    public boolean compareAndSet(V   expectedReference,
                                 V   newReference,
                                 int expectedStamp,
                                 int newStamp) {
        Pair<V> current = pair;
        return
            expectedReference == current.reference &&
            expectedStamp == current.stamp &&
            ((newReference == current.reference &&
              newStamp == current.stamp) ||
             casPair(current, Pair.of(newReference, newStamp)));
    }
    ...
    private boolean casPair(Pair<V> cmp, Pair<V> val) {
        return UNSAFE.compareAndSwapObject(this, pairOffset, cmp, val); // 通过Unsafe的compareAndSwapObject来更新
    }
    ...
}
```

## 总结
1. `java.util.concurrent.atomic`通过基于CAS的乐观锁保证线程安全性。在多读少写的场景下，较synchronized锁和ReentrantLock的悲观锁性能会更好。
2. JUC中大量运行了Unsafe的CAS操作，Unsafe的CAS是JUC的基础。
3. 基于CAS的乐观锁无法解决“ABA”问题，AtomicStampedReference通过加戳来解决“ABA”问题。
4. 基于CAS+循环的乐观锁会大量消耗CPU。
