---
layout: post
title: ThreadLocal解析
categories: [Java]
description: ThreadLocal解析
keywords: java, ThreadLocal
---

<h1 align="center">ThreadLocal解析</h1>

## 原理
线程安全问题的根源在于多线程之间的数据共享，如果没有数据共享，则就没有多线程并发安全问题。ThreadLocal就是用来避免多线程数据共享从而避免多线程并发安全问题。它为每个线程保留一个对象的副本，避免了多线程数据共享。每个线程作用的对象都是线程私有的一个对象拷贝。一个线程的对象副本无法被其他线程访问到（InheritableThreadLocal除外）。
注意ThreadLocal并不是一种多线程并发安全问题的解决方案，因为ThreadLocal的原理在于避免多线程数据共享从而实现线程安全。
来看一下JDK文档中ThreadLocal的描述：
```
This class provides thread-local variables.  These variables differ from
their normal counterparts in that each thread that accesses one (via its
{@code get} or {@code set} method) has its own, independently initialized
copy of the variable.  {@code ThreadLocal} instances are typically private
static fields in classes that wish to associate state with a thread (e.g.,
a user ID or Transaction ID).

Each thread holds an implicit reference to its copy of a thread-local
variable as long as the thread is alive and the {@code ThreadLocal}
instance is accessible; after a thread goes away, all of its copies of
thread-local instances are subject to garbage collection (unless other
references to these copies exist).
```
其大致的意思是：ThreadLocal为每个线程保留一个对象的副本，通过set()方法和get()方法来设置和获取对象。ThreadLocal通常被定义为私有的和静态的，用于关联线程的某些状态。当关联ThreadLocal的线程死亡后，ThreadLocal实例才可以被GC。也就是说如果ThreadLocal关联的线程如果没有死亡，则ThreadLocal就一直不能被回收。

## 用法
### 创建
`ThreadLocal<String> mThreadLocal = new ThreadLocal<>();`

### set方法
`mThreadLocal.set(Thread.currentThread().getName());`

### get方法
`String mThreadLocalVal = mThreadLocal.get();`

### 设置初始值
`ThreadLocal<String> mThreadLocal = ThreadLocal.withInitial(() -> Thread.currentThread().getName());`

### 完整示例
```
import java.util.concurrent.atomic.AtomicInteger;

public class ThreadId {

    // Atomic integer containing the next thread ID to be assigned
    private static final AtomicInteger nextId = new AtomicInteger(0);
    // Thread local variable containing each thread's ID
    private static final ThreadLocal<Integer> threadId =
            ThreadLocal.withInitial(() -> nextId.getAndIncrement());

    // Returns the current thread's unique ID, assigning it if necessary
    public static int get() {
        return threadId.get();
    }
}
```

## 源码分析
ThreadLocal为每个线程维护一个哈希表，用于保存线程本地变量，哈希表的key是ThreadLocal实例，value就是需要保存的对象。
```
publlic class Thread {
    ...
    /* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;
    ...
}

public class ThreadLocal {
    ...
    static class ThreadLocalMap {
        static class Entry extends WeakReference<ThreadLocal<?>> {
        ...
    ...
}
```
threadLocals是非静态的，也就是说每个线程都会有一个ThreadLocalMap哈希表用来保存本地变量。ThreadLocalMap的Entry的键(ThreadLocal<?>)是弱引用的，也就是说当垃圾收集器发现这个弱引用的键时不管内存是否足够多将其回收。这里回收的是ThreadLocalMap的Entry的ThreadLocal而不是Entry，因此还是可能会造成内存泄露。

### get()方法
```
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t); // 获取线程关联的ThreadLocalMap哈希表
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this); // 获取entry
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value; // 返回entry关联的对象
            return result;
        }
    }
    return setInitialValue(); // 如果当前线程关联的本地变量哈希表为空，则创建一个
}

ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}

```
首先通过getMap()方法获取当前线程的ThreadLocalMap实例，ThreadLocalMap是线程私有的，因此这里是线程安全的。ThreadLocalMap的getEntry()方法用于获取当前线程关联的ThreadLocalMap键值对，如果键值对不为空则返回值。如果键值对为空，则通过setInitialValue()方法设置初始值，并返回。注意setInitialValue()方法是private，是不可以覆写的。

### 设置初始值
```
private T setInitialValue() {
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
    return value;
}

protected T initialValue() {
    return null;
}
```
<font color="#FF0000">设置初始值会调用initialValue()方法获取初始值，该方法默认返回null，该方法可以被覆写，用于设置初始值。</font>例如上面的例子中，通过匿名内部类覆写了initialValue()方法设置了初始值。获取到初始值后，判断当前线程关联的本地变量哈希表是否为空，如果非空则设置初始值，否则先新建本地变量哈希表再设置初始值。最后返回这个初始值。

### set()方法
```
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```
该方法先获取该线程的 ThreadLocalMap 对象，然后直接将 ThreadLocal 对象（即代码中的 this）与目标实例的映射添加进 ThreadLocalMap 中。当然，如果映射已经存在，就直接覆盖。另外，如果获取到的 ThreadLocalMap 为 null，则先创建该 ThreadLocalMap 对象。

## 防止内存泄露
前面分析得知ThreadLocalMap的Entry的key是弱引用的，key可以在垃圾收集器工作的时候就被回收掉，但是存在 当前线程->ThreadLocal->ThreadLocalMap->Entry的一条强引用链，因此如果当前线程没有死亡，或者还持有ThreadLocal实例的引用Entry就无法被回收。从而造成内存泄露。
当我们使用线程池来处理请求的时候，一个请求处理完成，线程并不一定会被回收，因此线程还会持有ThreadLocal实例的引用，即使ThreadLocal已经没有作用了。此时就发生了ThreadLocal的内存泄露。
针对该问题，ThreadLocalMap 的 set 方法中，通过 replaceStaleEntry 方法将所有键为 null 的 Entry 的值设置为 null，从而使得该值可被回收。另外，会在 rehash 方法中通过 expungeStaleEntry 方法将键和值为 null 的 Entry 设置为 null 从而使得该 Entry 可被回收。通过这种方式，ThreadLocal 可防止内存泄漏。

```
private void set(ThreadLocal<?> key, Object value) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);

    for (Entry e = tab[i]; e != null; e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();
        if (k == key) {
            e.value = value;
            return;
        }
        if (k == null) {
            replaceStaleEntry(key, value, i); // key为空，则代表该Entry不再需要，设置Entry的value指针和Entry指针为null，帮助GC
            return;
        }
    }
    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```

## 使用场景
ThreadLocal场景的使用场景：
* 每个线程需要有自己单独的实例
* 实例需要在多个方法中共享，但不希望被多线程共享
例如用来解决数据库连接、Session管理等。

## InheritableThreadLocal
ThreadLocal为每个线程保留一个线程私有的对象副本，线程之间无法共享访问，但是有一个例外：InheritableThreadLocal，InheritableThreadLocal可以实现在子线程中访问父线程中的对象副本。下面是一个InheritableThreadLocal和ThreadLocal区别的例子：
```
public class InheritableThreadLocalExample {

    public static void main(String[] args) throws InterruptedException {
        InheritableThreadLocal<Integer> integerInheritableThreadLocal = new InheritableThreadLocal<>();
        ThreadLocal<Integer> integerThreadLocal = new ThreadLocal<>();
        integerInheritableThreadLocal.set(1);
        integerThreadLocal.set(0);
        Thread thread = new Thread(() -> System.out.println(Thread.currentThread().getName() + ", " + integerThreadLocal.get() + " / " + integerInheritableThreadLocal.get()));
        thread.start();
        thread.join();
    }
}

```
运行上述代码会发现在子线程中可以获取父线程的InheritableThreadLocal中的变量，但是无法获取父线程的ThreadLocal中的变量。
分析InheritableThreadLocal的源码可以知道其是ThreadLocal的子类：
```
public class InheritableThreadLocal<T> extends ThreadLocal<T> {
    ...
    // 覆写了ThreadLocal的getMap方法，返回的是Thread中的inheritableThreadLocals
    ThreadLocalMap getMap(Thread t) {
       return t.inheritableThreadLocals;
    }

    // 覆写了createMap方法，创建的也是Thread中的inheritableThreadLocals这个哈希表
    void createMap(Thread t, T firstValue) {
        t.inheritableThreadLocals = new ThreadLocalMap(this, firstValue);
    }
}
```
而Thread的inheritableThreadLocals会在线程初始化的时候进行初始化。这个过程在Thread类的init()方法中：
```
private void init(ThreadGroup g, Runnable target, String name,
                  long stackSize, AccessControlContext acc) {
    ...
    Thread parent = currentThread();
    ...
    if (parent.inheritableThreadLocals != null)
        this.inheritableThreadLocals =
            ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
    ...
}
```
可以看到一个线程在初始化的时候，会判断创建这个线程的父线程的inheritableThreadLocals是否为空，如果不为空，则会拷贝父线程inheritableThreadLocals到当前创建的子线程的inheritableThreadLocals中去。
当我们在子线程调用get()方法时，InheritableThreadLocal的getMap()方法返回的是Thread中的inheritableThreadLocals，而子线程的inheritableThreadLocals已经拷贝了父线程的inheritableThreadLocals，因此在子线程中可以读取父线程中的inheritableThreadLocals中保存的对象。


## 总结
* ThreadLocal为每个线程保留对象副本，多线程之间没有数据共享。因此它并不解决线程间共享数据的问题。
* 每个线程持有一个 Map 并维护了 ThreadLocal 对象与具体实例的映射，该 Map 由于只被持有它的线程访问，故不存在线程安全以及锁的问题。
* ThreadLocalMap 的 Entry 对 ThreadLocal 的引用为弱引用，避免了 ThreadLocal 对象无法被回收的问题。
* ThreadLocalMap 的 set 方法通过调用 replaceStaleEntry 方法回收键为 null 的 Entry 对象的值（即为具体实例）以及 Entry 对象本身从而防止内存泄漏。
* ThreadLocal 适用于变量在线程间隔离且在方法间共享的场景。
* ThreadLocal中的变量是线程私有的，其他线程无法访问到另外一个线程的变量。但是InheritableThreadLocal是个例外，通过InheritableThreadLocal可以在子线程中访问到父线程中的变量。