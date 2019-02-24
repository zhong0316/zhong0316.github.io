---
layout: post
title: Java Object类的各个方法
categories: [Java]
description: Java Object类的各个方法
keywords: Object, Java
---

<h1 align="center">Java Object类的各个方法</h1>

Java中所有的类都继承自`java.lang.Object`类，Object类中一共有11个方法：
```
public final native Class<?> getClass();

public native int hashCode();

public boolean equals(Object obj) {
    return (this == obj);
}

protected native Object clone() throws CloneNotSupportedException;

public String toString() {
    return getClass().getName() + "@" + Integer.toHexString(hashCode());
}

public final native void notify();

public final native void notifyAll();

public final native void wait(long timeout) throws InterruptedException;

public final void wait(long timeout, int nanos) throws InterruptedException {
    if (timeout < 0) {
        throw new IllegalArgumentException("timeout value is negative");
    }

    if (nanos < 0 || nanos > 999999) {
        throw new IllegalArgumentException(
                            "nanosecond timeout value out of range");
    }

    if (nanos > 0) {
        timeout++;
    }

    wait(timeout);
}

public final void wait() throws InterruptedException {
    wait(0);
}

protected void finalize() throws Throwable { }
```

## getClass方法
这是一个native方法，并且是'final'的，也就是说这个方法不允许在子类中覆写。
getClass方法返回的是当前实例对应的Class类，也就是说不管一个类有多少个实例，每个实例的getClass返回的Class对象是一样的。请看下面的例子:
```
Integer i1 = new Integer(1);
Class i1Class = i1.getClass();

Integer i2 = new Integer(1);
Class i2Class = i2.getClass();
System.out.println(i1Class == i2Class);
```
上面的代码运行结果为`true`，也就是说两个Integer的实例的getClass方法返回的Class对象是同一个。


### Integer.class和int.class
Java中还有一个方法可以获取Class，例如我们想获取一个Integer实例对应的Class，可以直接通过`Integer.class`来获取，请看下面的例子：
```
Integer num = new Integer(1);
Class numClass = num.getClass();
Class integerClass = Integer.class;
System.out.println(numClass == integerClass);
```
上面代码的运行结果为`true`，也就是说通过调用实例的getClass方法和`类.class`返回的Class是一样的。与Integer对象的还有int类型的原生类，与`Integer.class`对应，`int.class`用于获取一个原生类型int的Class。但是他们两个返回的不是同一个对象。请看下面的例子：
```
Class intClass = int.class;
Class intTYPE = Integer.TYPE;
Class integerClass = Integer.class;
System.out.println(intClass == integerClass);
System.out.println(intClass == intTYPE);
```
上面的代码运行结果是：
```
false
true
```
`Integer.class`和`int.class`返回的不是一个对象，而`int.class`返回的和`Integer.TYPE`是同一个对象。`Integer.TYPE`定义如下：
```
public static final Class<Integer>  TYPE = (Class<Integer>) Class.getPrimitiveClass("int");
```
**Java中为原生类型(boolean,byte,char,short,int,long,float,double)和void都创建了一个预先定义好的类型，可以通过包装类的TYPE静态属性获取。上述的Integer类中TYPE和`int.class`是等价的。**

## hashCode方法
对象的哈希码主要用于在哈希表中的存放和查找等。Java中对于对象hashCode方法的规约如下：
1. 在java程序执行过程中，在一个对象没有被改变的前提下，无论这个对象被调用多少次，hashCode方法都会返回相同的整数值。对象的哈希码没有必要在不同的程序中保持相同的值。
2. 如果2个对象使用equals方法进行比较并且相同的话，那么这2个对象的hashCode方法的值也必须相等。
3. 如果根据equals方法，得到两个对象不相等，那么这2个对象的hashCode值不需要必须不相同。但是，不相等的对象的hashCode值不同的话可以提高哈希表的性能。

为了理解这三条规约，我们来先看一下HashMap中put一个entry的过程：
```
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    if ((p = tab[i = (n - 1) & hash]) == null) // 这里key的hashcode被用来定位当前entry在哈希表中的index
        tab[i] = newNode(hash, key, value, null); // 如果当前哈希表中没有key对应的entry，则直接插入
    else {
        // 当前哈希表中已经有了key对应的entry了，则找到这个节点，然后看是否需要更新这个entry的value
        Node<K,V> e; K k;
        // 判断当前节点就是已经存在的entry的条件：1：hashcode相等；2：当前节点的key和key是同一个对象（==）或者两者的equals方法判定相等
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        // 是否需要更新旧值，如果需要更新则更新
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```
HashMap中put一个键值对时有以下几个步骤：
1. 计算key的哈希码，通过哈希码定位新增的entry应该处于哈希表中的哪个位置，如果当前位置为null，则代表哈希表中没有key对应的entry，直接新插入一个节点就好了。
2. 如果当前key在哈希表中已经有了映射，则先查找这个节点，判定当前是否为目标节点的条件有两个：**1）两者的哈希码必须相等；2）两者是同一个对象（==成立），或者两者的equals方法判定两者相等。**
3. 判断是否需要更新旧值，需要的话就更新。

分析完了HashMap的put操作后，我们再来看看这三条规约：
1. 第一条规约要求多次调用一个对象的hashCode方法返回的值要相等。设想如果一个key的hashCode方法每次返回值如果不同，则在put的时候就可能定位到哈希表中不同的位置，就产生了歧义：明明两个key是同一个，但是哈希表中存在同一个key的多个不同映射。这就违背了哈希表的key不能重复的原则了。
2. 第二条规约也很好理解：如果两个key的equals方法判定两者相等，则说明哈希表中只需要保留一个key就行了。如果equals判定相等，而hashcode不同，则就会违背这个事实。
3. 第三条规约是用来优化哈希表的性能的，如果哈希表put时"碰撞"太多，势必会造成查找性能下降。

## equals方法
equals方法用于判定两个对象是否相等。Object中的equals方法其实默认比较的是两个对象是否拥有相同的地址。也就是"=="对应的内存语义。

但是在子类中我们可以覆写equals来判定子类的两个实例是否为同一个对象。例如对于一个人来说，他有很多属性，但是每个人的id是唯一的，如果两个人的id一样则证明两个人是同一个人，而不管其他属性是否一样。这个时候我们就可以覆写人的equals方法来保证这点了。
```
public class Person {
    
    private long id;
    private String name;
    private int age;
    private String nation;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Person person = (Person) o;
        return id == person.id; // 如果两个person的id相同，则我们认为他们是同一个对象
    }

    @Override
    public int hashCode() {
        return Objects.hash(id);
    }
}
```

equals方法在非空对象引用上的特性：

1. reflexive，自反性。任何非空引用值x，对于x.equals(x)必须返回true
2. symmetric，对称性。任何非空引用值x和y，如果x.equals(y)为true，那么y.equals(x)也必须为true
3. transitive，传递性。任何非空引用值x、y和z，如果x.equals(y)为true并且y.equals(z)为true，那么x.equals(z)也必定为true
4. consistent，一致性。任何非空引用值x和y，多次调用 x.equals(y) 始终返回 true 或始终返回 false，前提是对象上 equals 比较中所用的信息没有被修改
5. 对于任何非空引用值 x，x.equals(null) 都应返回 false

Java要求一个类的**equals方法和hashCode方法同时覆写**。我们刚刚分析了HashMap中对于key的处理过程：首先根据key的哈希码定位哈希表中的位置，其次根据"=="或者equals方法判定两个key是否相同。如果Person的equals方法没有被覆写，则两个Person对象即使id一样，但是不是指向同一块内存地址，那么哈希表中就查找不到已经存在的映射entry了。

## clone方法
用于克隆一个对象，被克隆的对象需要implements Cloneable接口，否则调用这个对象的clone方法，将会抛出CloneNotSupportedException异常。克隆的对象通常情况下满足以下三条规则：
1. x.clone() != x，克隆出来的对象和原来的对象不是同一个，指向不同的内存地址
2. x.clone().getClass() == x.getClass()
3. x.clone().equals(x)

**一个对象进行clone时，原生类型和包装类型的field的克隆原理不同。对于原生类型是直接复制一个，而对于包装类型，则只是复制一个引用而已，并不会对引用类型本身进行克隆。**

### 浅拷贝
浅拷贝例子：
```
public class ShallowCopy {

    public static void main(String[] args){
        Man man = new Man();
        Man manShallowCopy = (Man) man.clone();
        System.out.println(man == manShallowCopy);
        System.out.println(man.name == manShallowCopy.name);
        System.out.println(man.mate == manShallowCopy.mate);
    }
}

class People implements Cloneable {

    // primitive type
    public int id;
    // reference type
    public String name;

    @Override
    public Object clone() {
        try {
            return super.clone();
        } catch (CloneNotSupportedException e) {
            e.printStackTrace();
        }
        return null;
    }
}

class Man extends People implements Cloneable {
    // reference type
    public People mate = new People();

    @Override
    public Object clone() {
        return super.clone();
    }
}
```
上面的代码的运行结果是:
```
false
true
true
```
通过浅拷贝出来的Man对象manShallowCopy的name和mate属性和原来的对象man都指向了相同的内存地址。在对Man的name和mate进行拷贝时浅拷贝只是对引用进行了拷贝，指向的还是同一块内存地址。

### 深拷贝
对一个对象深拷贝时，对于对象的包装类型的属性，会对其再进行拷贝，从而达到深拷贝的目的，请看下面的例子：
```
public class DeepCopy {

    public static void main(String[] args){
        Man man = new Man();
        Man manDeepCopy = (Man) man.clone();
        System.out.println(man == manDeepCopy);
        System.out.println(man.mate == manDeepCopy.mate);
    }
}

class People implements Cloneable {

    // primitive type
    public int id;
    // reference type
    public String name;

    @Override
    public Object clone() {
        try {
            return super.clone();
        } catch (CloneNotSupportedException e) {
            e.printStackTrace();
        }
        return null;
    }
}

class Man extends People implements Cloneable {
    // reference type
    public People mate = new People();

    // 深拷贝Man
    @Override
    public Object clone() {
        Man man = (Man) super.clone();
        man.mate = (People) this.mate.clone(); // 再对mate属性进行clone，从而达到深拷贝
        return man;
    }
}
```
上面代码的运行结果为：
```
false
false
```
Man对象的clone方法中，我们先对Man进行了clone，然后对mate属性也进行了拷贝。因此man的mate和manDeepCopy的mate指向了不同的内存地址。也就是深拷贝。

**通常来说对一个对象进行完完全全的深拷贝是不现实的，例如上面的例子中，虽然我们对Man的mate属性进行了拷贝，但是无法对name（String类型）进行拷贝，拷贝的还是引用而已。**

## toString方法
Object中默认的toString方法如下：
```
public String toString() {
    return getClass().getName() + "@" + Integer.toHexString(hashCode());
}
```
也就是class的名称+对象的哈希码。一般在子类中我们可以对这个方法进行覆写。

## notify方法
notify方法是一个final类型的native方法，子类不允许覆盖这个方法。

notify方法用于唤醒正在等待当前对象监视器的线程，唤醒的线程是随机的。一般notify方法和wait方法配合使用来达到多线程同步的目的。
在一个线程被唤醒之后，线程必须先重新获取对象的监视器锁（线程调用对象的wait方法之后会让出对象的监视器锁），才可以继续执行。

一个线程在调用一个对象的notify方法之前必须获取到该对象的监视器(synchronized)，否则将抛出IllegalMonitorStateException异常。同样一个线程在调用一个对象的wait方法之前也必须获取到该对象的监视器。

wait和notify使用的例子：
```
Object lock = new Object();
Thread t1 = new Thread(() -> {
    synchronized (lock) {
        System.out.println(Thread.currentThread().getName() + " is going to wait on lock's monitor");
        try {
            lock.wait();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(Thread.currentThread().getName() + " relinquishes the lock's monitor");
    }
});
Thread t2 = new Thread(() -> {
    synchronized (lock) {
        System.out.println(Thread.currentThread().getName() + " is going to notify a thread that waits on lock's monitor");
        lock.notify();
    }
});
t1.start();
t2.start();
```

## notifyAll方法
notifyAll方法用于唤醒所有等待对象监视器锁的线程，notify只唤醒所有等待线程中的一个。

同样，如果当前线程不是对象监视器的所有者，那么调用notifyAll同样会发生IllegalMonitorStateException异常。

## wait方法
```
public final native void wait(long timeout) throws InterruptedException;
```
wait方法一般和上面说的notify方法搭配使用。一个线程调用一个对象的wait方法后，线程将进入`WAITING`状态或者`TIMED_WAITING`状态。直到其他线程唤醒这个线程。

线程在调用对象的wait方法之前必须获取到这个对象的monitor锁，否则将抛出IllegalMonitorStateException异常。线程的等待是支持中断的，如果线程在等待过程中，被其他线程中断，则抛出InterruptedException异常。

如果wait方法的参数timeout为0，代表等待过程是不会超时的，直到其他线程notify或者被中断。如果timeout大于0，则代表等待支持超时，超时之后线程自动被唤醒。

## finalize方法
```
protected void finalize() throws Throwable { }
```
垃圾回收器在回收一个无用的对象的时候，会调用对象的finalize方法，我们可以覆写对象的finalize方法来做一些清除工作。下面是一个finalize的例子：
```
public class FinalizeExample {

    public static void main(String[] args) {
        WeakReference<FinalizeExample> weakReference = new WeakReference<>(new FinalizeExample());
        weakReference.get();
        System.gc();
        System.out.println(weakReference.get() == null);
    }

    @Override
    public void finalize() {
        System.out.println("I'm finalized!");
    }
}
```
输出结果：
```
I'm finalized!
true
```
finalize方法中对象其实还可以"自救"，避免垃圾回收器将其回收。在finalize方法中通过建立this的一个强引用来避免GC：
```
public class FinalizeExample {

    private static FinalizeExample saveHook;

    public static void main(String[] args) {
        FinalizeExample.saveHook = new FinalizeExample();
        saveHook = null;
        System.gc();
        System.out.println(saveHook == null);
    }

    @Override
    public void finalize() {
        System.out.println("I'm finalized!");
        saveHook = this;
    }
}
```
输出结果：
```
I'm finalized!
false
```