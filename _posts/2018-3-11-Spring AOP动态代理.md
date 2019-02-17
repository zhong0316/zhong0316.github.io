---
layout: post
title: Spring AOP动态代理
categories: [Java, Spring]
description: Spring AOP动态代理
keywords: Spring, AOP, 动态代理
---

<h1 align="center">Spring AOP动态代理</h1>
IOC和AOP是Spring中最重要的两个概念，而AOP最核心的部分在于动态代理。Spring AOP中的拦截功能都是通过动态代理来生成的。
那么什么是动态代理呢？所谓动态代理是指代理类是在JVM运行时动态生成的，与之相对的是静态代理。静态代理中代理类是在编译期生成的，静态代理相对动态代理来说效率会更高，但是会生成大量的代理类，不利于开发。而动态代理虽然效率会低一些，但是其大大提高了代码的简洁度和开发工作。有关二者的具体比较可以参照我前面的额博文：[Java中的静态代理和动态代理]({{ site.url }}/2018/03/10/Java中的静态代理和动态代理)。

Spring AOP的动态代理主要有两种实现方式：JDK动态代理（JDK Dynamic Proxy）和CGlib动态代理。JDK动态代理是基于Java反射功能来实现的，而CGlib动态代理借助于ASM字节码生成工具来生成代理类。两者各有优劣：JDK动态代理相对来说效率更高，而CGlib动态代理相对更加灵活。下面我们通过一个例子来说明如何使用两者进行动态代理。

## 使用JDK动态代理
### 定义接口和实现类
```
interface HelloService {
    void sayHello();
}

class HelloServiceImpl implements HelloService {
    @Override
    public void sayHello() {
        System.out.println("Hello World!");
    }
}
```

### 实现动态代理类
```
class HelloServiceDynamicProxy {

    private HelloService helloService;
    public HelloServiceDynamicProxy(HelloService helloService) {
        this.helloService = helloService;
    }

    public Object getProxyInstance() {
        return Proxy.newProxyInstance(helloService.getClass().getClassLoader(), helloService.getClass().getInterfaces(), new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                System.out.println("JDK Dynamic Proxy, Before say hello...");
                Object ret = method.invoke(helloService, args);
                System.out.println("JDK Dynamic Proxy, After say hello...");
                return ret;
            }
        });
    }
}
```

### 测试类
```
public class HelloServieDynamicProxyTest {
    public static void main(String[] args){
        HelloService helloService = new HelloServiceImpl();
        HelloService dynamicProxy = (HelloService) new HelloServiceDynamicProxy(helloService).getProxyInstance();
        dynamicProxy.sayHello();
    }
}
```

### 输出结果
```
JDK Dynamic Proxy, Before say hello...
Hello World!
JDK Dynamic Proxy, After say hello...
```

## 使用CGlib实现动态代理
### 定义接口和实现类
```
interface HelloService {
    void sayHello();
}

class HelloServiceImpl implements HelloService {
    @Override
    public void sayHello() {
        System.out.println("Hello World!");
    }
}
```

### 实现动态代理类
```
class HelloServiceCgLibProxy implements MethodInterceptor {

    private HelloService helloService;

    public HelloServiceCgLibProxy(HelloService helloService) {
        this.helloService = helloService;
    }

    public Object createProxy() {
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(helloService.getClass());
        enhancer.setCallback(this);
        return enhancer.create();
    }

    @Override
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        System.out.println("CGlib Dynamic Proxy, Before say hello...");
        Object ret = method.invoke(helloService, objects);
        System.out.println("CGlib Dynamic Proxy, After say hello...");
        return ret;
    }
}
```

### 测试类
```
public class HelloServieCGlibDynamicProxyTest {
    public static void main(String[] args){
        HelloService helloService = new HelloServiceImpl();
        HelloServiceCgLibProxy helloServiceCgLibProxy = new HelloServiceCgLibProxy(helloService);
        HelloService helloServiceProxy = (HelloService) helloServiceCgLibProxy.createProxy();
        helloServiceProxy.sayHello();
    }
}


```

### 输出结果
```
CGlib Dynamic Proxy, Before say hello...
Hello World!
CGlib Dynamic Proxy, After say hello...
```

## 指定Spring AOP的动态代理方式
```
<aop:config proxy-target-class="true">
</aop:config>
```
通过配置Spring的中`<aop:config>`标签来显示的指定使用动态代理机制`proxy-target-class=true`表示使用CGLib代理，如果为false就是默认使用JDK动态代理。

## 总结
1. JDK动态代理基于接口，要求委托了必须实现接口，如果没有实现接口的话JDK动态带就无能为力了。
2. CGlib动态代理不要求委托类实现接口，CGlib使用的是继承机制，代理类继承委托类。CGlib要求委托类不能是final的，否则会创建代理失败，因为final类是不允许继承的。

