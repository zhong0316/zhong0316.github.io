---
layout: post
title: Java7 ConcurrentHashMap源码浅析
categories: [Java]
description: Java7 ConcurrentHashMap源码浅析
keywords: ConcurrentHashMap, 并发
---

<h1 align="center">Java7 ConcurrentHashMap源码浅析</h1>

Java中的哈希表主要包括：HashMap，HashTable，ConcurrentHashMap，LinkedHashMap和TreeMap等。HashMap是无序的，并且不是线程安全的，在多线程环境下会出现数据安全性问题，典型的问题是多线程同时rehash过程中产生的死循环问题。LinkedHashMap和TreeMap都是有序的，但是两者的有序机制不同：LinkedHashMap是通过链表的结构保证元素有序，而TreeMap是一种红黑树结构，通过堆排序保证元素有序。在Java6中HashMap，HashTable和ConcurrentHashMap都是采用数组+链表的数据结构，在Java8之后则采用数组+链表+红黑树的数据结构。HashTable和ConcurrentHashMap都是线程安全的，保证线程安全无外乎加锁，但是二者加锁的粒度不通，HashTable整个表就一把锁，它的get和put都是通过synchronized保证安全，在多线程竞争锁激烈的情况下，会出现性能问题。本文讲解的ConcurrentHashMap是Java7版本。

## 数据结构

Java7中ConcurrentHashMap采用数组+链表的数据结构，哈希表整体上是一个Segment的数组，而每个分段Segment又是一个HashEntry的数组，每个HashEntry是一个链表。

<div align="center">
    <img src="{{ site.url }}/images/posts/java/Java7ConcurrentHashMap结构.png" alt="Java7ConcurrentHashMap"/>
</div>

## ConcurrentHashMap锁分段

<font color="#FF0000">锁粗化是锁优化的一种重要措施，而锁粗化又包含"lock-splitting"（锁定拆分）和"lock-stripping"（锁条带化）。读写锁分离是一种典型的锁定拆分方式，JUC中的ReentrantReadWriteLock就是一种读写分离锁，锁定条带化是指将一把“大锁”拆分成若干个“小锁”来降低锁的竞争。</font>ConcurrentHashMap就是通过锁条带化来做锁的优化。我们都知道ConcurrentHashMap是分段的，它的表是一个Segment数组：
```
/**
 * The segments, each of which is a specialized hash table
 */
final Segment<K,V>[] segments;
```
而每个Segment都是继承了一个ReentrantLock：
```
static final class Segment<K,V> extends ReentrantLock implements Serializable {...}
```
所以ConcurrentHashMap的每个Segment都是一把锁，不同的Segment之间的读写不构成竞争，大大降低了锁的竞争。既然每个Segment都是一把锁，那么这个segment数组的长度是多少呢？也就是说整个表我们需要多少把锁呢？
```
public ConcurrentHashMap(int initialCapacity,
                         float loadFactor, int concurrencyLevel) {
    if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();

    if (concurrencyLevel > MAX_SEGMENTS) // MAX_SEGMENTS是指整个表最多能分成多少个segment，也即是多少把锁
        concurrencyLevel = MAX_SEGMENTS;

    // Find power-of-two sizes best matching arguments
    int sshift = 0;
    int ssize = 1;
    while (ssize < concurrencyLevel) { // 找到不小于我们指定的concurrencyLevel的2的幂次方的一个数作为segment数组的长度
        ++sshift;
        ssize <<= 1;
    }
    segmentShift = 32 - sshift;
    segmentMask = ssize - 1;
    this.segments = Segment.newArray(ssize);

    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    int c = initialCapacity / ssize;
    if (c * ssize < initialCapacity)
        ++c;
    int cap = 1;
    while (cap < c)
        cap <<= 1;

    for (int i = 0; i < this.segments.length; ++i)
        this.segments[i] = new Segment<K,V>(cap, loadFactor);
}
```
在ConcurrentHashMap的构造函数中我们指定了concurrencyLevel，也即是多少把锁。这个数量不能超过上限:MAX_SEGMENTS(1 << 16)，锁的个数必须是2的幂次方，如果我们指定的concurrencyLevel不是2的幂次方，构造函数会找到最接近的一个不小于我们指定的值的一个2的幂次方数作为segment数组长度。例如：我们指定concurrencyLevel为15，则最终segment数组长度为16，也即是表一共有16把锁。设想两个线程同时向表中插入元素，线程1插入的第0个segment，线程2插入的是第1个segment，线程1和线程2互不影响，能够同时并行。但是HashTable就做不到这一点。

## put操作

```
public V put(K key, V value) {
    Segment<K,V> s;
    if (value == null)
        throw new NullPointerException();
    int hash = hash(key);
    int j = (hash >>> segmentShift) & segmentMask; // 通过位运算得到segment的索引位置
    if ((s = (Segment<K,V>)UNSAFE.getObject          // nonvolatile; recheck
         (segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
        s = ensureSegment(j);
    return s.put(key, hash, value, false);
}
```
ConcurrentHashMap不支持插入null的值，因此首先校验value是否为null。如果value是null则抛出异常。
注意这里计算segment索引方式是: `(hash >>> segmentShift) & segmentMask;` 而不是hash % segment数组长度。<font color="#FF0000">这儿是一个优化：因为取模"%"操作相对位运算来说是很慢的，因此这里是用位运算来得到segment索引。而当segment数组长度是2的幂次方记为segmentSize时:`hash % segmentSize == hash & (segmentSize - 1)`。</font>这里不做证明。因此segmentSize必须是2的幂次方。来看看Segment中的put()方法：

```
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
    HashEntry<K,V> node = tryLock() ? null :  // 获取segment的锁，这里会有一个优化：获取锁的时候首先会通过 `tryLock()` 尝试若干次
        scanAndLockForPut(key, hash, value);  // 如果若干次之后还没有获取锁，则用 `lock()` 方法阻塞等待，直到获取锁
    V oldValue;
    try {
        HashEntry<K,V>[] tab = table;
        int index = (tab.length - 1) & hash; // 得到segment的table的索引，也是通过位运算
        HashEntry<K,V> first = entryAt(tab, index); // table中index位置的first节点
        for (HashEntry<K,V> e = first;;) {
            if (e != null) {
                K k;
                if ((k = e.key) == key ||
                    (e.hash == hash && key.equals(k))) { // 对应的key已经有了value
                    oldValue = e.value;
                    if (!onlyIfAbsent) { // 是否覆盖原来的value
                        e.value = value; // 覆盖原来的value
                        ++modCount;
                    }
                    break;
                }
                e = e.next;  // 遍历
            }
            else {
                if (node != null)
                    node.setNext(first); // 如果node已经在scanAndLockForPut()方法中初始化过
                else
                    node = new HashEntry<K,V>(hash, key, value, first); // 如果node为null，则初始化
                int c = count + 1;
                if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                    rehash(node); // 如果超过阈值，则扩容
                else
                    setEntryAt(tab, index, node); // 通过UNSAFE设置table数组的index位置的元素为node
                ++modCount;
                count = c;
                oldValue = null;
                break;
            }
        }
    } finally {
        unlock();
    }
    return oldValue;
}
```
首先，会获取segment的锁，然后判断添加元素后是否需要扩容。<font color="#FF000000">注意这里的扩容是指Segment中的HashEntry[] table表数组扩容，而不是最外层的segment[]数组扩容。segment[]数组是不可扩展的，在构造函数中已经确定了segment[]数组的长度。</font>
接着同样通过位运算得到待添加元素在HashEntry[] table数组中的位置。接着再判断这个链表中是否已经存在这个key，如果存在并且onlyIfAbsent为false，就覆盖原value；如果链表不存在key，则将新的node通过UNSAFE放到table数组指定的位置。

## get操作

get操作比较简单，不需要加锁。可见性由volatile来保证：HashEntry的value是volatile的，Segment中的HashEntry[] table数组也是volatile。这保证了其他线程对哈希表的修改能够及时地被读线程发现。
```
public V get(Object key) {
    Segment<K,V> s; // manually integrate access methods to reduce overhead
    HashEntry<K,V>[] tab;
    int h = hash(key);
    long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE; // 计算key应该落在segments数组的哪个segment中
    if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
        (tab = s.table) != null) {
        for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile
                 (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE); // 计算key应该落在table的哪个位置
             e != null; e = e.next) {
            K k;
            if ((k = e.key) == key || (e.hash == h && key.equals(k))) // 如果key和当前节点的key指向同一块内存地址或者当前节点的hash
                return e.value;                                       // 等于key的hash并且key"equals"当前节点的key则说明当前节点就是目标节点
        }
    }
    return null; // key不在当前哈希表中，返回null
}
```

## rehash

```
private void rehash(HashEntry<K,V> node) {
    /*
     * Reclassify nodes in each list to new table.  Because we
     * are using power-of-two expansion, the elements from
     * each bin must either stay at same index, or move with a
     * power of two offset. We eliminate unnecessary node
     * creation by catching cases where old nodes can be
     * reused because their next fields won't change.
     * Statistically, at the default threshold, only about
     * one-sixth of them need cloning when a table
     * doubles. The nodes they replace will be garbage
     * collectable as soon as they are no longer referenced by
     * any reader thread that may be in the midst of
     * concurrently traversing table. Entry accesses use plain
     * array indexing because they are followed by volatile
     * table write.
     */
    HashEntry<K,V>[] oldTable = table;
    int oldCapacity = oldTable.length;
    int newCapacity = oldCapacity << 1; // 每次扩容成原来capacity的2倍，这样元素在新的table中的索引要么不变要么是原来的索引加上2的一个倍数
    threshold = (int)(newCapacity * loadFactor); // 新的扩容阈值
    HashEntry<K,V>[] newTable =
        (HashEntry<K,V>[]) new HashEntry[newCapacity]; // 新的segment table数组
    int sizeMask = newCapacity - 1;
    for (int i = 0; i < oldCapacity ; i++) {
        HashEntry<K,V> e = oldTable[i];
        if (e != null) {
            HashEntry<K,V> next = e.next;
            int idx = e.hash & sizeMask;
            if (next == null)   //  Single node on list
                newTable[idx] = e;
            else { // Reuse consecutive sequence at same slot
                HashEntry<K,V> lastRun = e;
                int lastIdx = idx;
                for (HashEntry<K,V> last = next;  // 在拷贝原来链表的元素到新的table中时有个优化：通过遍历找到原先链表中的lastRun节点，这个节点以及它的后续节点都不需要重新拷贝，直接放到新的table中就行
                     last != null;
                     last = last.next) {
                    int k = last.hash & sizeMask;
                    if (k != lastIdx) {
                        lastIdx = k;
                        lastRun = last;
                    }
                }
                newTable[lastIdx] = lastRun; // lastRun节点以及lastRun后续节点都不需要重新拷贝，直接赋值引用
                // Clone remaining nodes
                for (HashEntry<K,V> p = e; p != lastRun; p = p.next) { // 循环拷贝原先链表lastRun之前的节点到新的table链表中
                    V v = p.value;
                    int h = p.hash;
                    int k = h & sizeMask;
                    HashEntry<K,V> n = newTable[k];
                    newTable[k] = new HashEntry<K,V>(h, p.key, v, n);
                }
            }
        }
    }
    int nodeIndex = node.hash & sizeMask; // add the new node
    node.setNext(newTable[nodeIndex]); // rehash之后，执行添加新的节点
    newTable[nodeIndex] = node;
    table = newTable;
}
```
由于rehash过程中是加排它锁的，这样其他的写入请求将被阻塞等待。而对于读请求，需要分情况讨论：读请求在rehash之前，此时segment中的table数组指针还是指向原先旧的数组，所以读取是安全的；如果读请求在rehash之后，因为table数组和HashEntry的value都是volatile，所以读线程也能及时读取到更新的值，因此也是线程安全的。所以rehash不会影响到读。

## remove操作

```
public V remove(Object key) {
    int hash = hash(key);
    Segment<K,V> s = segmentForHash(hash); // key落在哪个segment中
    return s == null ? null : s.remove(key, hash, null); // 如果segment为null，则说明哈希表中没有key，直接返回null，否则调用Segment的remove
}

final V remove(Object key, int hash, Object value) { // Segment的remove方法
    if (!tryLock()) // 获取Segment的锁，套路还是一样的首先进行若干次 `tryLock()`, 如果失败了则通过 `lock()` 方法阻塞等待直到获取锁
        scanAndLock(key, hash);
    V oldValue = null;
    try {
        HashEntry<K,V>[] tab = table;
        int index = (tab.length - 1) & hash;
        HashEntry<K,V> e = entryAt(tab, index); // 找到key具体在table的哪个链表中，e代表链表当前节点
        HashEntry<K,V> pred = null; // pred代表e节点的前置节点
        while (e != null) {
            K k;
            HashEntry<K,V> next = e.next;
            if ((k = e.key) == key ||
                (e.hash == hash && key.equals(k))) { // 找到了这个key对应的HashEntry
                V v = e.value;
                if (value == null || value == v || value.equals(v)) {
                    if (pred == null) // 如果当前节点的前置节点为空，说明要删除的节点是当前链表的头节点，直接将当前链表的头节点指向当前节点的next就可以了
                        setEntryAt(tab, index, next);
                    else
                        pred.setNext(next); // 否则修改前置节点的next指针，指向当前节点的next节点，这样当前节点将不再"可达"，可以被GC回收
                    ++modCount;
                    --count;
                    oldValue = v;
                }
                break;
            }
            pred = e;
            e = next;
        }
    } finally {
        unlock(); // 解锁
    }
    return oldValue;
}
```
remove时，首先会找到这个key落在哪个Segment中，如果key没有落在任何一个Segment中，说明key不存在，直接返回null。找到具体的Segment后，调用Segment的remove方法来进行删除：找到key落在Segment的table数组中的哪个链表中，遍历链表，如果要删除的节点是当前链表的头节点，则直接修改链表的头指针为当前节点的next节点；如果要删除的节点不是头节点，继续遍历找到目标节点，修改目标节点的前置节点的next指针指向目标节点的next节点完成操作。
安全性分析：remove时首先会加锁，其他mutable请求都是会被阻塞的，对于读请求也是安全的。如果读取的key不是当前要删除的key不会有任何问题。如果读取的key恰巧是当前需要删除key：读请求在remove之前，这时可以读取到；如果读请求在remove操作之后，由于HashEntry的next指针都是volatile的，所以读线程也是可以及时发现这个key已经被删除了的。也是安全的。

## size操作
ConcurrentHashMap的size操作在Java7实现还是比较有意思的。<font color="#FF0000">其首先会进行若干次尝试，每次对各个Segment的count求和，如果任意前后两次求和结果相同，则说明在这段时间之内各个Segment的元素个数没有改变，直接返回当前的求和结果就行了。如果超过一定重试次数之后，会采取悲观策略，直接锁定各个Segment，然后依次求和。</font>注意这里是锁定所有Segment，因此在采取悲观策略时整个哈希表都不能有写入操作。这里先乐观再悲观的策略和前面的put操作中的scanAndLockForPut有异曲同工之妙。

```
public int size() {
    // Try a few times to get accurate count. On failure due to
    // continuous async changes in table, resort to locking.
    final Segment<K,V>[] segments = this.segments; // 首先不加锁，每次对各个Segment的count累加求和，如果任意两次的累加结果相同，则直接返回这个结果；超过一定的次数之后悲观锁定所有的Segment，再求和。锁定之后整个哈希表不能有任何的写入操作。
    int size;
    boolean overflow; // true if size overflows 32 bits
    long sum;         // sum of modCounts
    long last = 0L;   // previous sum
    int retries = -1; // first iteration isn't retry
    try {
        for (;;) {
            if (retries++ == RETRIES_BEFORE_LOCK) { // 最大乐观重试次数
                for (int j = 0; j < segments.length; ++j)
                    ensureSegment(j).lock(); // force creation
            }
            sum = 0L;
            size = 0;
            overflow = false;
            for (int j = 0; j < segments.length; ++j) { // 对各个Segment的count累加，不加锁
                Segment<K,V> seg = segmentAt(segments, j);
                if (seg != null) {
                    sum += seg.modCount;
                    int c = seg.count;
                    if (c < 0 || (size += c) < 0)
                        overflow = true;
                }
            }
            if (sum == last) // 如果本次累加结果和上次相同，说明这中间没有插入或者删除操作，直接返回这个结果
                break;
            last = sum;
        }
    } finally {
        if (retries > RETRIES_BEFORE_LOCK) {
            for (int j = 0; j < segments.length; ++j)
                segmentAt(segments, j).unlock();
        }
    }
    return overflow ? Integer.MAX_VALUE : size; // 如果溢出，返回最大整型作为结果，否则返回累加结果
}
```

## 总结
1. ConcurrentHashMap采取分段锁，是一种典型的"lock-stripping"策略，目的是为了降低高并发情况下的锁竞争。
2. rehash过程的扩容不是segment数组的扩容，而是Segment中HashEntry[] table数组的扩容，Segment[] segments数组是final的，在哈希表初始化完成后不再更改。
3. ConcurrentHashMap中很多地方用到了volatile，保证了可见性，例如，Segment中的HashEntry[] table数组，HashEntry中的value和next指针都是volatile的。
4. 加锁是低效的，线程的上下文切换需要消耗性能，因此ConcurrentHashMap很多地方都用到了乐观重试的策略，在超过一定次数之后再采取悲观策略。例如size操作。
5. 在使用ConcurrentHashMap之前请明确你的数据结构是否真的会有多线程并发操作，如果没有，仅仅是单线程操作，请使用HashMap，因为在不考虑并发安全性问题时，不论是HashTable还是ConcurrentHashMap他们的性能都没有HashMap好。