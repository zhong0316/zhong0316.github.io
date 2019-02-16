---
layout: post
title: Reactor线程模型及其在Netty中的应用
categories: [Java, 线程]
description: Reactor线程模型及其在Netty中的应用
keywords: 多线程, Netty, Reactor 
---

<h1 align="center">Reactor线程模型及其在Netty中的应用</h1>

## 什么是Reactor线程模型
Java中线程模型大致可以分为：
1. 单线程模型
2. 多线程模型
3. 线程池模型(executor)
4. Reactor线程模型

单线程模型中，server端使用一个线程来处理所有的请求，所有的请求必须串行化处理，效率低下。
多线程模型中，server端会为每个请求分配一个线程去处理请求，相对单线程模型而言多线程模型效率更高，但是多线程模型的缺点也很明显：server端为每个请求都开辟一个线程来处理请求，如果请求数量很大，则会造成大量线程被创建，造成内存溢出。Java中线程是比较昂贵的对象。线程的数量不应该无限量的增大，当线程数超过一定数目后，增加线程不仅不能提高效率，反而会降低效率。
借助"对象复用"的思想，线程池应运而生，线程池中一般有一定数目的线程，当请求数目超过线程数之后需要排队等待。这样就避免了线程会不停地增长。这里不打算对Java的线程池做过多介绍，有兴趣的可以去看我之前的文章：[java线程池]({{ site.url }}/2018/01/25/java线程池)。

Reactor是一种处理模式。Reactor模式是处理并发I/O比较常见的一种模式，用于同步I/O，中心思想是将所有要处理的IO事件注册到一个中心I/O多路复用器上，同时主线程/进程阻塞在多路复用器上；一旦有I/O事件到来或是准备就绪(文件描述符或socket可读、写)，多路复用器返回并将事先注册的相应I/O事件分发到对应的处理器中。

Reactor也是一种实现机制。Reactor利用事件驱动机制实现，和普通函数调用的不同之处在于：应用程序不是主动的调用某个API完成处理，而是恰恰相反，Reactor逆置了事件处理流程，应用程序需要提供相应的接口并注册到Reactor上，如果相应的事件发生，Reactor将主动调用应用程序注册的接口，这些接口又称为“回调函数”。用“好莱坞原则”来形容Reactor再合适不过了：不要打电话给我们，我们会打电话通知你。

## 为什么需要Reactor模型
Reactor模型实质上是对I/O多路复用的一层包装，理论上来说I/O多路复用的效率已经够高了，为什么还需要Reactor模型呢？答案是I/O多路复用虽然性能已经够高了，但是编码复杂，在工程效率上还是太低。因此出现了Reactor模型。

一个个网络请求可能涉及到多个I/O请求，相比传统的单线程完整处理请求生命期的方法，I/O复用在人的大脑思维中并不自然，因为，程序员编程中，处理请求A的时候，假定A请求必须经过多个I/O操作A1-An（两次IO间可能间隔很长时间），每经过一次I/O操作，再调用I/O复用时，I/O复用的调用返回里，非常可能不再有A，而是返回了请求B。即请求A会经常被请求B打断，处理请求B时，又被C打断。这种思维下，编程容易出错。

## Reactor模型
一般来说处理一个网络请求需要经过以下五个步骤：
1. 读取请求数据(read request)
2. 解码数据(decode request)
3. 计算，生成响应(compute)
4. 编码响应(encode response)
5. 发送响应数据(send response)
如下图所示：
<div align="center">
    <img src="{{ site.url }}/images/posts/java/ClassicServiceDesign.png" alt="ClassicServiceDesign"/>
</div>
不难看出，上面的模型中读取请求数据和发送响应和业务处理逻辑耦合在一起，都是用一个handler线程处理，当发生读写事件时线程将被阻塞，无法处理其他的事情。既然不能耦合在一起，那自然解决方案是将读写操作和其他三个步骤进行分离：有专门的线程或者线程池负责连接请求，然后使用多路复用器来监测Socket上的读写事件，其他的处理委派给业务线程池进行处理。读写操作不阻塞主线程。

Reactor模型有三种线程模型：
1. 单线程模型
2. 多线程模型（单Reactor）
3. 多线程模型（多Reactor)

## 单线程Reactor模型
单线程模型中Reactor既负责Accept新的连接请求，又负责分派请求到具体的handler中进行处理，一般不使用这种模型，因为单线程效率比较低下。
<div align="center">
    <img src="{{ site.url }}/images/posts/java/BasicReactorPattern.png" alt="BasicReactorPattern"/>
</div>

下面是基于Java NIO单线程Reactor模型的实现：

```
class Reactor implements Runnable {
    final Selector selector;
    final ServerSocketChannel serverSocket;

    Reactor(int port) throws IOException { // Reactor设置
        selector = Selector.open();
        serverSocket = ServerSocketChannel.open();
        serverSocket.socket().bind(
                new InetSocketAddress(port));
        serverSocket.configureBlocking(false);
        SelectionKey sk =
                serverSocket.register(selector,
                        SelectionKey.OP_ACCEPT); // 监听Socket连接事件
        sk.attach(new Acceptor());
    }

    public void run() { // normally in a new Thread
        try {
            while (!Thread.interrupted()) {
                selector.select();
                Set selected = selector.selectedKeys();
                Iterator it = selected.iterator();
                while (it.hasNext())
                    dispatch((SelectionKey) (it.next());
                selected.clear();
            }
        } catch (IOException ex) { /* ... */ }
    }

    void dispatch(SelectionKey k) {
        Runnable r = (Runnable) (k.attachment());
        if (r != null)
            r.run();
    }

    class Acceptor implements Runnable { // inner
        public void run() {
            try {
                SocketChannel c = serverSocket.accept();
                if (c != null)
                    new Handler(selector, c);
            } catch (IOException ex) { /* ... */ }
        }
    }
}


final class Handler implements Runnable {
    static final int READING = 0, SENDING = 1;
    final SocketChannel socket;
    final SelectionKey sk;
    ByteBuffer input = ByteBuffer.allocate(1024);
    ByteBuffer output = ByteBuffer.allocate(1024);
    int state = READING;

    Handler(Selector sel, SocketChannel c)
            throws IOException {
        socket = c;
        c.configureBlocking(false);
        // Optionally try first read now
        sk = socket.register(sel, 0);
        sk.attach(this);
        sk.interestOps(SelectionKey.OP_READ);
        sel.wakeup();
    }

    boolean inputIsComplete() { /* ... */ }

    boolean outputIsComplete() { /* ... */ }

    void process() { /* ... */ }

    public void run() {
        try {
            if (state == READING) read();
            else if (state == SENDING) send();
        } catch (IOException ex) { /* ... */ }
    }

    void read() throws IOException {
        socket.read(input);
        if (inputIsComplete()) {
            process();
            state = SENDING;
            // Normally also do first write now
            sk.interestOps(SelectionKey.OP_WRITE);
        }
    }

    void send() throws IOException {
        socket.write(output);
        if (outputIsComplete()) sk.cancel();
    }
}
```

## 多线程模型（单Reactor）
该模型在事件处理器（Handler）链部分采用了多线程（线程池），也是后端程序常用的模型。模型图如下：
<div align="center">
    <img src="{{ site.url }}/images/posts/java/WorkerThreadPools-Reactor.png" alt="WorkerThreadPools"/>
</div>
其实现大致代码如下：
```
class Handler implements Runnable {
    // 使用线程池
    static PooledExecutor pool = new PooledExecutor(...);
    static final int PROCESSING = 3;
    // ...
    synchronized void read() { // ...
        socket.read(input);
        if (inputIsComplete()) {
            state = PROCESSING;
            pool.execute(new Processer());
        }
    }
    synchronized void processAndHandOff() {
        process();
        state = SENDING; // or rebind attachment
        sk.interest(SelectionKey.OP_WRITE);
    }
    class Processer implements Runnable {
        public void run() { processAndHandOff(); }
    }
}
```

## 多线程模型（多Reactor）
比起多线程单Rector模型，它是将Reactor分成两部分，mainReactor负责监听并Accept新连接，然后将建立的socket通过多路复用器（Acceptor）分派给subReactor。subReactor负责多路分离已连接的socket，读写网络数据；业务处理功能，其交给worker线程池完成。通常，subReactor个数上可与CPU个数等同。其模型图如下：
<div align="center">
    <img src="{{ site.url }}/images/posts/java/UsingMultiplyReactors-Reactor.png" alt="UsingMultiplyReactors"/>
</div>

## Netty线程模型
Netty的线程模型类似于Reactor模型。Netty中ServerBootstrap用于创建服务端，下图是它的结构：
<div align="center">
    <img src="{{ site.url }}/images/posts/java/Netty-ServerBootstrap.png" alt="Netty-ServerBootstrap"/>
</div>
ServerBootstrap继承自AbstractBootstrap，AbstractBootstrap中的group属性就是Netty中的Acceptor，用于接受请求。而ServerBootstrap中的childGroup对应于Reactor模型中的worker线程池。
请求过来后Netty从group线程池中选出一个线程来建立连接，连接建立后委派给childGroup中的worker线程处理。
服务端线程模型工作原理如下图：
<div align="center">
    <img src="{{ site.url }}/images/posts/java/Netty服务端线程工作流程.png" alt="Netty服务端线程工作流程"/>
</div>

下面是一个完整的Netty服务端的例子：

```
public class TimeServer {
    public void bind(int port) {
        // Netty的多Reactor线程模型，bossGroup是Acceptor线程池，用于接受连接。workGroup是Worker线程池，处理业务。
        // bossGroup是Acceptor线程池
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        // workGroup是Worker线程池
        EventLoopGroup workGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workGroup).channel(NioServerSocketChannel.class)
                    .option(ChannelOption.SO_BACKLOG, 1024)
                    .childHandler(new ChildChannelHandler());
            // 绑定端口，同步等待成功
            ChannelFuture f = b.bind(port).sync();
            // 等待服务端监听端口关闭
            f.channel().closeFuture().sync();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            bossGroup.shutdownGracefully();
            workGroup.shutdownGracefully();
        }
    }
    private class ChildChannelHandler extends ChannelInitializer<SocketChannel> {
        @Override
        protected void initChannel(SocketChannel ch) throws Exception {
            ch.pipeline().addLast(new TimeServerHandler());
        }
    }
}

public class TimeServerHandler extends ChannelHandlerAdapter {

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause)
            throws Exception {
        ctx.close();
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg)
            throws Exception {
        // msg转Buf
        ByteBuf buf = (ByteBuf) msg;
        // 创建缓冲中字节数的字节数组
        byte[] req = new byte[buf.readableBytes()];
        // 写入数组
        buf.readBytes(req);
        String body = new String(req, "UTF-8");
        String currenTime = "QUERY TIME ORDER".equalsIgnoreCase(body) ? new Date(
                System.currentTimeMillis()).toString() : "BAD ORDER";
        // 将要返回的信息写入Buffer
        ByteBuf resp = Unpooled.copiedBuffer(currenTime.getBytes());
        // buffer写入通道
        ctx.write(resp);
    }

    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
        // write读入缓冲数组后通过invoke flush写入通道
        ctx.flush();
    }
}
```

## 参考资料
[Scalable IO in Java](http://gee.cs.oswego.edu/dl/cpjslides/nio.pdf)
[Netty 系列之 Netty 线程模型](https://www.infoq.cn/article/netty-threading-model)