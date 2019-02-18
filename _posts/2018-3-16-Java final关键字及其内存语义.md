---
layout: post
title: Java final关键字及其内存语义
categories: [Java]
description: Java final关键字及其内存语义
keywords: final
---

<h1 align="center">Java final关键字及其内存语义</h1>
final是Java中的一个关键字，final可用于修饰类、方法、参数和变量（包括实例变量和类变量）。

## final修饰类
final修饰的类具有不可继承性，也就是如果一个类是final类型的，则这个类不允许有子类。首先我们顶一个final类:
```
public final class FinalClass {
    private int field;
}
```
然后如果我们尝试去继承这个类的话编译器会报错：
```
public class FalseExtension extends FinalClass {

}
```

<div align="center">
    <img src="{{ site.url }}/images/posts/java/FalseExtension.png" alt="FalseExtension"/>
</div> 
编译器提示我们：cannot extend final class，也就是说final类型的类不允许继承。

## final修饰方法
final修饰的方法具有不可变性，也就是说final的方法不允许在子类中被覆写(@Override)。下面我们来看一个反例：
<div align="center">
    <img src="{{ site.url }}/images/posts/java/finalMethod.png" alt="finalMethod"/>
</div> 
在Base子类Extension中我们覆写（Override）了子类中的final方法，编译器提示错误：final方法不能被覆写。

## final修饰参数和变量
如果参数用final修饰，那么在方法中我们不能对这个final参数进行修改：
```
public void test(final int x) {
    // x++; // 这句是非法的，因为x是final的
}
```

final修饰的变量（包括实例变量和类变量）具有引用不可变性，例如：`private final int x = 1`，这里变量x是final类型的，如果我们尝试修改x：`x = 2`，编译器就会提示出错。
同样的对于包装类，如果我们尝试修改变量的指针，一样会提示错误：
```
class Test {
    
    private int x;

    public int getX() {
        return x;
    }

    public void setX(int x) {
        this.x = x;
    }
}

final Test test = new Test();
// test = new Test(); // 这个表达式是非法的，因为test是final的
test.setX(2); // 这个表达式是合法的因为这个表达式没有修改test的引用
```
<font color="#FF0000">注意这里包装类的不可变性是指引用的不可变，如果我们不修改变量的引用，而是通过访问变量所指向的包装类的方法去修改包装类的属性，这个是合法的。</font>

## 编译器对final变量的优化
请看下面的例子：
```

public class Test {
    public static void main(String[] args)  {
        String a = "helloworld1"; 
        final String b = "helloworld";
        String c = "helloworld";
        String d = b + 2; 
        String e = d + 2;
        System.out.println((a == d));
        System.out.println((a == e));
    }
}
```
输出结果：
```
true
false
```
可以看到a和d是指向同一个地址，而a和e则是指向不同的地址。这里就可以看出final变量和普通变量的区别了。变量b是final类型的，编译器知道b的引用不会改变，因此直接可以“算出来”d就是“helloworld2”，那么a和d都是指向"helloworld2"字面量的变量，自然`a == d`成立了。编译器知道一个字符串变量是final不可变的变量后，就可以直接进行替换了。对于编译期间不能确定的final变量，编译器则不会进行替换，请看下面这个例子：

```
public class Test {
    public static void main(String[] args)  {
        String a = "helloworld2"; 
        final String b = getHelloWorld();
        String c = b + 2; 
        System.out.println((a == c));
 
    }
     
    public static String getHelloWorld() {
        return "helloworld";
    }
}
```
这段代码的输出结果为false。

因为编译器无法在编译期就能确定b为"helloworld"，因此编译器无法对b进行替换，所以a和c是不等的。

## final的内存语义
### final域的重排序规则
1. 在构造函数内对一个final域的写入，与随后把这个被构造对象的引用赋值给一个引用变量，这两个操作之间不能重排序。
2. 初次读一个包含final域的对象的引用，与随后初次读这个final域，这两个操作之间不能重排序。
### 写final域的重排序规则
写final域的重排序规则禁止把final域的写重排序到构造函数之外。这个规则的实现包含下面2个方面：
1. JMM禁止编译器把final域的写重排序到构造函数之外。
2. 编译器会在final域的写之后，构造函数return之前，插入一个StoreStore屏障。这个屏障禁止处理器把final域的写重排序到构造函数之外。写final域的重排序规则可以确保：在对象引用为任意线程可见之前，对象的final域已经被正确初始化过了，而普通域不具有这个保障。
### 读final域的重排序规则
读final域的重排序规则是，在一个线程中，初次读对象引用与初次读该对象包含的final域，JMM禁止处理器重排序这两个操作（注意，这个规则仅仅针对处理器）。编译器会在读final域操作的前面插入一个LoadLoad屏障。初次读对象引用与初次读该对象包含的final域，这两个操作之间存在间接依赖关系。由于编译器遵守间接依赖关系，因此编译器不会重排序这两个操作。大多数处理器也会遵守间接依赖，也不会重排序这两个操作。但有少数处理器允许对存在间接依赖关系的操作做重排序（比如alpha处理器），这个规则就是专门用来针对这种处理器的。读final域的重排序规则可以确保：在读一个对象的final域之前，一定会先读包含这个final域的对象的引用。
### final域为引用类型
对于引用类型，写final域的重排序规则对编译器和处理器增加了如下约束：在构造函数内对一个final引用的对象的成员域的写入，与随后在构造函数外把这个被构造对象的引用赋值给一个引用变量，这两个操作之间不能重排序。这一规则确保了其他线程能读到被正确初始化的final引用对象的成员域。

