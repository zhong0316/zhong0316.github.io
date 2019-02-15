---
layout: post
title: Java8ConcurrentHashMap
categories: [Java]
description: Java8ConcurrentHashMap
keywords: ConcurrentHashMap, 并发
---

<h1 align="center">Java8ConcurrentHashMap</h1>

在前面一篇文章中我们谈了Java7中的ConcurrentHashMap：[java7ConcurrentHashMap源码浅析]({{ site.url }}/2018/01/26/java7ConcurrentHashMap源码浅析)，主要分析了Java7中ConcurrentHashMap的分段锁设计，数据结构以及一些常用的操作。Java8中ConcurrentHashMap有了一些重要改进，本篇文章主要想谈谈Java8中ConcurrentHashMap中的一些改进。

## 数据结构
Java7中ConcurrentHashMap是采用了数组+链表的数据结构。我们都知道链表的查找效率是低下的，其平均的查找时间是O(logn)，其中n是链表的长度。当链表较长时，查找可能会成为哈希表的性能瓶颈。针对这个问题，Java8中对ConcurrentHashMap的数据结构进行了优化，采用了数组+链表+红黑树的数据结构。当数组中某个链表的长度超过一定的阈值之后，会将其转换成红黑树，以提高查找效率。当红黑树节点数少于一定阈值后，还能降红黑树逆退化成链表。下图是Java8ConcurrentHashMap的数据结构示意图：

<div align="center">
    <img src="{{ site.url }}/images/posts/java/Java8ConcurrentHashMap结构.png" alt="Java8ConcurrentHashMap"/>
</div>

Java8中还取消了Java7中的分段锁设计。Java7中ConcurrentHashMap有一个“currencyLevel”的概念，这个字段指定了哈希表共有多少把锁，在哈希表初始化完成之后不可更改。Java8中ConcurrentHashMap加锁的粒度改为数组中的节点。其实本质上区别不大，都是一种锁条带化(lock-stripping)的思想。

## put操作

```
public V put(K key, V value) {
    return putVal(key, value, false); // onlyIfAbsent为false，默认覆盖旧值
}

/** Implementation for put and putIfAbsent */
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
            tab = initTable(); // 初始化哈希表，第一次put时会触发这个操作
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) { // 如果table中当前节点还未初始化，则初始化当前节点并设置value
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        else if ((fh = f.hash) == MOVED) // 如果当前节点的hash为MOVED，则帮助重建哈希表，这是Java8中一个重要优化，后面会说到
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            synchronized (f) { // 加锁粒度为Node节点
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) { // 当前节点hashcode大于0，说明是链表
                        binCount = 1; // 当前链表的节点数，用于判断是否要将链表转换成红黑树
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value; // 覆盖旧值
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) { // 如果当前节点是红黑树节点则将key，value插入到红黑树中
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD) // 如果链表节点超过阈值，则将链表转换成红黑树，其实不一定会转换成红黑树也可能是扩容，也就是哈希重建。这个方法回头再看
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount);
    return null;
}
```
put操作时，首先会检查哈希表是否初始化过了，如果没有初始化则进行初始化。找到待新加的元素在哈希表中的索引位置，如果当前节点是null，则初始化这个位置的节点，并设置value。如果当前节点的hashcode为`MOVED`，则说明哈希表正在重建中。<font color="#FF0000">此时，线程不会阻塞等待哈希表重建完成，并且会帮助重建，"人多力量大"。多线程重建，rehash自然会更快一点。这是Java8中针对ConcurrentHashMap重要的一个优化。</font>具体怎么帮助重建后面会详细讲解。

## 扩容分析
Java8中采用多线程方式对ConcurrentHashMap进行扩容，扩容时主要通过sizeCtl和transferIndex这两个属性来进行并发控制，sizeCtl标识当前表的状态：
```
/**
 * Table initialization and resizing control.  When negative, the
 * table is being initialized or resized: -1 for initialization,
 * else -(1 + the number of active resizing threads).  Otherwise,
 * when table is null, holds the initial table size to use upon
 * creation, or 0 for default. After initialization, holds the
 * next element count value upon which to resize the table.
 */
private transient volatile int sizeCtl;
```
1. 当sizeCtl为负数时候，如果为-1则表示当前哈希表正在初始化，否则为`-(1 + the number of active resizing threads)`，例如如果sizeCtl为-2，则表示当前已经有一个线程正在对哈希表进行扩容操作。
2. 当哈希表还未初始化时，如果指定了哈希表的容量，则sizeCtl等于指定的初始容量，否则缺省为0。
3. 当哈希表初始化后，sizeCtl表示下一次哈希表需要扩容时的大小，等于loadFactor * n。
注意到sizeCtl是volatile的，它的修改能够保证对其他线程的可见性。

transferIndex：
```
private transient volatile int transferIndex;
```
表示当前哈希表的扩容位置，在扩容前transferIndex指向当前表的最后一个节点，transferIndex=tab.length。每一个线程负责扩容一部分节点，从右向左划分成若干个独立任务，每个线程负责的这部分节点叫做一个stride：
```
if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
    stride = MIN_TRANSFER_STRIDE; // subdivide range
```
MIN_TRANSFER_STRIDE为16，就是说每个线程最少负责16个节点的迁移工作。

### 扩容时机
1. put时如果发现哈希表当前节点的hash==MOVED，代表当前哈希表正在扩容，当前线程尝试去帮助扩容：
```
final V putVal(K key, V value, boolean onlyIfAbsent) {
...
    else if ((fh = f.hash) == MOVED)
        tab = helpTransfer(tab, f);
...
```
2. 将链表转换成红黑树时可能触发帮助扩容操作：
```
private final void treeifyBin(Node<K,V>[] tab, int index) {
    Node<K,V> b; int n, sc;
    if (tab != null) {
        if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
            tryPresize(n << 1);
    ...
```
如果发现此时哈希表长度小于`MIN_TREEIFY_CAPACITY`（64）时触发扩容操作。
3. put操作之后'addCount'发现超过容量，则进行扩容：
```
final V putVal(K key, V value, boolean onlyIfAbsent) {
    ...
    addCount(1L, binCount);
    ...
}

private final void addCount(long x, int check) {
    ...
    while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
       (n = tab.length) < MAXIMUM_CAPACITY) {
       // 调用transfer
    }
    ...
}
```

### transfer操作
transfer操作是实际的迁移过程，多线程并发将tab的节点迁移到newTable中，由于哈希表的容量是成倍增长，因此迁移后节点要么在原先的位置，要么在原先位置+扩容前的容量。Doug Lea说了在哈希表扩容过程中大约只需要重新拷贝六分之一的节点：
`We eliminate unnecessary node creation by catching cases where old nodes can be reused because their next fields won't change.  On average, only about one-sixth of them need cloning when a table doubles.`

```
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    //计算需要迁移多少个hash桶（MIN_TRANSFER_STRIDE该值作为下限，以避免扩容线程过多）
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range
   
    if (nextTab == null) {            // initiating
        try {
            //扩容一倍
            @SuppressWarnings("unchecked")
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        nextTable = nextTab;
        transferIndex = n;
    }
    int nextn = nextTab.length;
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    boolean advance = true;
    boolean finishing = false; // to ensure sweep before committing nextTab
  
    //1 逆序迁移已经获取到的hash桶集合，如果迁移完毕，则更新transferIndex，获取下一批待迁移的hash桶
    //2 如果transferIndex=0，表示所以hash桶均被分配，将i置为-1，准备退出transfer方法
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        
        //更新待迁移的hash桶索引
        while (advance) {
            int nextIndex, nextBound;
            //更新迁移索引i。
            if (--i >= bound || finishing)
                advance = false;
            else if ((nextIndex = transferIndex) <= 0) {
                //transferIndex<=0表示已经没有需要迁移的hash桶，将i置为-1，线程准备退出
                i = -1;
                advance = false;
            }
            //当迁移完bound这个桶后，尝试更新transferIndex，，获取下一批待迁移的hash桶
            else if (U.compareAndSwapInt
                     (this, TRANSFERINDEX, nextIndex,
                      nextBound = (nextIndex > stride ?
                                   nextIndex - stride : 0))) {
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }
        //退出transfer
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            if (finishing) {
                //最后一个迁移的线程，recheck后，做收尾工作，然后退出
                nextTable = null;
                table = nextTab;
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                /**
                 第一个扩容的线程，执行transfer方法之前，会设置 sizeCtl = (resizeStamp(n) << RESIZE_STAMP_SHIFT) + 2)
                 后续帮其扩容的线程，执行transfer方法之前，会设置 sizeCtl = sizeCtl+1
                 每一个退出transfer的方法的线程，退出之前，会设置 sizeCtl = sizeCtl-1
                 那么最后一个线程退出时：
                 必然有sc == (resizeStamp(n) << RESIZE_STAMP_SHIFT) + 2)，即 (sc - 2) == resizeStamp(n) << RESIZE_STAMP_SHIFT
                */
                
                //不相等，说明不到最后一个线程，直接退出transfer方法
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                finishing = advance = true;
                //最后退出的线程要重新check下是否全部迁移完毕
                i = n; // recheck before commit
            }
        }
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);
        else if ((fh = f.hash) == MOVED)
            advance = true; // already processed
        //迁移node节点
        else {
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    Node<K,V> ln, hn;
                    //链表迁移
                    if (fh >= 0) {
                        int runBit = fh & n;
                        Node<K,V> lastRun = f;
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        if (runBit == 0) {
                            ln = lastRun;
                            hn = null;
                        }
                        else {
                            hn = lastRun;
                            ln = null;
                        }
                        //将node链表，分成2个新的node链表
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            if ((ph & n) == 0)
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        //将新node链表赋给nextTab
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                    //红黑树迁移
                    else if (f instanceof TreeBin) {
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                (h, e.key, e.val, null, null);
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                            (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                            (lc != 0) ? new TreeBin<K,V>(hi) : t;
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                }
            }
        }
    }
}
```

## get操作
1. get操作时首先会定位bin在table中的位置，如果bin的头节点就是我们当前要的直接返回。
2. 如果当前bin头节点hash小于0，则说明当前哈希表在扩容或者当前节点是红黑树。如果在扩容则说明当前节点是ForwardingNode，交给ForwardingNode去查找；如果是红黑树，则交由TreeBin节点查找。
3. 到这里说明当前bin是一个链表，则直接遍历链表进行查找。
```
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    int h = spread(key.hashCode());
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {
        if ((eh = e.hash) == h) { // 判断当前头节点是否就是我们需要查找的
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        else if (eh < 0) // 节点hash小于0，要么哈希表在扩容要么当前节点是红黑树
            return (p = e.find(h, key)) != null ? p.val : null;
        while ((e = e.next) != null) { // 当前节点是链表，直接遍历查找即可
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```

## 总结
1. Java8 ConcurrentHashMap取消了分段锁的设计，加锁粒度是哈希表Node[] table数组的每个元素。
2. 采用多线程方式扩容哈希表，每个线程负责一个stride，加速了扩容过程。
3. 在Java7 ConcurrentHashMap数组+链表的基础上加入了红黑树，进一步提高查找效率。

## 参考资料

[ConcurrentHashMap源码分析（JDK8） 扩容实现机制](https://www.jianshu.com/p/487d00afe6ca)  
[Java7/8 中的 HashMap 和 ConcurrentHashMap 全解析](https://javadoop.com/post/hashmap#%E6%89%A9%E5%AE%B9%3A%20rehash)