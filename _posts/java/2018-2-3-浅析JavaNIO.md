---
layout: post
title: 浅析Java NIO
categories: [Java]
description: 浅析Java NIO
keywords: java, NIO, IO
---

<h1 align="center">浅析Java NIO</h1>

## NIO概述
Java NIO全称为Non-blocking IO或者New IO，从名字我们知道NIO是非阻塞的IO，而Java IO则是阻塞的IO。在一般的情况下阻塞是低效率的，特别是在高并发的场景下面，因此Java引入了NIO。NIO相比IO来说主要有以下几个区别：
1. NIO是面向缓冲区的，IO则面向流。
* 标准的IO编程接口是面向字节流和字符流的。而NIO是面向通道和缓冲区的，数据总是从通道中读到buffer缓冲区内，或者从buffer缓冲区写入到通道中；（ NIO中的所有I/O操作都是通过一个通道开始的。）
* Java IO面向流意味着每次从流中读一个或多个字节，直至读取所有字节，它们没有被缓存在任何地方；
* Java NIO是面向缓存的I/O方法。 将数据读入缓冲器，使用通道进一步处理数据。 在NIO中，使用通道和缓冲区来处理I/O操作。

2. NIO是非阻塞的，IO是阻塞的。
* Java NIO使我们可以进行非阻塞IO操作。比如说，单线程中从通道读取数据到buffer，同时可以继续做别的事情，当数据读取到buffer中后，线程再继续处理数据。写数据也是一样的。另外，非阻塞写也是如此。一个线程请求写入一些数据到某通道，但不需要等待它完全写入，这个线程同时可以去做别的事情。
* Java IO的各种流是阻塞的。这意味着，当一个线程调用read() 或 write()时，该线程被阻塞，直到有一些数据被读取，或数据完全写入。该线程在此期间不能再干任何事情了

3. NIO有Selectors（多路复用器），而IO没有Selectors。
* 选择器用于使用单个线程处理多个通道。因此，它需要较少的线程来处理这些通道。
* 线程之间的切换对于操作系统来说是昂贵的。 因此，为了提高系统效率选择器是有用的

NIO中主要有以下三个概念：通道、缓冲区和Selectors。

## 通道
Java NIO Channel通道和流非常相似，主要有以下几点区别：
* 通道可以读也可以写，流一般来说是单向的（只能读或者写）。
* 通道可以异步读写。
* 通道总是基于缓冲区Buffer来读写。

<div align="center">
    <img src="{{ site.url }}/images/posts/java/nio-channel.png" alt="nio-channel"/>
</div> 

Java的NIO读写都是在通道中进行的，通道涵盖了网络UDP，TCP网络IO和文件IO：
* DatagramChannel
* SocketChannel
* FileChannel
* ServerSocketChannel
各个Channel的UML类图如下：
<div align="center">
    <img src="{{ site.url }}/images/posts/java/NIO-Channels.png" alt="NIO-Channels"/>
</div> 

### DatagramChannel
DatagramChannel用于处理UDP连接。

**打开一个DatagramChannel**
```
DatagramChannel channel = DatagramChannel.open();
channel.socket().bind(new InetSocketAddress(8888));
```
**读取数据**
```
ByteBuffer buf = ByteBuffer.allocate(1024);
buf.clear();
channel.receive(buf);
```
**发送数据**
```
String msg = "Current time is: " + System.currentTimeMillis();
ByteBuffer buf = ByteBuffer.allocate(1024);
buf.clear();
buf.put(msg.getBytes());
buf.flip();
int bytesSent = channel.send(buf, new InetSocketAddress("host", port));
```

### SocketChannel
**打开 SocketChannel**
```
SocketChannel socketChannel = SocketChannel.open();
socketChannel.connect(new InetSocketAddress("host", 80));
```
**读取数据**
```
ByteBuffer buf = ByteBuffer.allocate(48);
int bytesRead = socketChannel.read(buf);
```
如果 read()返回 -1, 表明连接已经中断。
    
**写入数据**
```
String msg = "Current Time is: " + System.currentTimeMillis();
ByteBuffer buf = ByteBuffer.allocate(48);
buf.clear();
buf.put(msg.getBytes());

buf.flip();

while(buf.hasRemaining()) {
    channel.write(buf);
}
```
    
**非阻塞模式**
```
socketChannel.configureBlocking(false);
socketChannel.connect(new InetSocketAddress("host", 80));

while (!socketChannel.finishConnect()) {
      
}
```  
我们可以设置 SocketChannel 为异步模式, 这样 connect, read, write 都是异步的了。在异步模式中, 或许连接还没有建立, connect 方法就返回了, 因此我们需要检查当前是否是连接到了主机，因此通过一个 while 循环来判断。
    
### FileChannel
**打开**
```
RandomAccessFile aFile = new RandomAccessFile("test.txt", "rw");
FileChannel inChannel = aFile.getChannel();
```
**读取**
```
ByteBuffer buf = ByteBuffer.allocate(48);
int bytesRead = inChannel.read(buf);
```
**写入**
```
String newData = "Current time is: " + System.currentTimeMillis();

ByteBuffer buf = ByteBuffer.allocate(1024);
buf.clear();
buf.put(newData.getBytes());
buf.flip();
while (buf.hasRemaining()) {
    channel.write(buf);
}
```
**关闭**
```
channel.close();
```
**设置 position**
```
long pos = channel.position();
channel.position(pos + 123);
```

**文件大小**

我们可以通过 channel.size()获取关联到这个 Channel 中的文件的大小。注意, 这里返回的是文件的大小, 而不是 Channel 中剩余的元素个数。

**截断文件**
```
channel.truncate(1024);
```
将文件的大小截断为1024字节。

**强制写入**
```
channel.force(true);
```
强制将缓存中的数据写入文件中：

### ServerSocketChannel
ServerSocketChannel顾名思义，它是用来监听server端的socket连接。
**打开和关闭**
```
ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
serverSocketChannel.close();
```
**监听连接**

使用ServerSocketChannel的accept()方法来监听客户端的TCP连接请求，accept()方法是阻塞的，直到有连接进来。
```
while(true){
    SocketChannel socketChannel = serverSocketChannel.accept();
    //do something with socketChannel...
}
```
**非阻塞模式**

如果设定ServerSocketChannel是非阻塞的，则accept()方法不会阻塞。如果返回的是null证明没有新的连接，如果不是null，则有新的连接请求。
```
ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
serverSocketChannel.socket().bind(new InetSocketAddress(9999));
serverSocketChannel.configureBlocking(false);
while (true) {
    SocketChannel socketChannel = serverSocketChannel.accept();
    if(socketChannel != null) {
        // do something with socketChannel...
    }
}
```

## Buffer缓冲区

Java NIO Buffers用于和NIO Channel交互。正如你已经知道的，我们从channel中读取数据到buffers里，从buffer把数据写入到channels。

buffer本质上就是一块内存区，可以用来写入数据，并在稍后读取出来。这块内存被NIO Buffer包裹起来，对外提供一系列的读写方便开发的接口。

**Buffer基本用法**

利用Buffer读写数据，通常遵循四个步骤：

* 把数据写入buffer；
* 调用flip；
* 从Buffer中读取数据；
* 调用buffer.clear()或者buffer.compact()

当写入数据到buffer中时，buffer会记录已经写入的数据大小。当需要读数据时，通过flip()方法把buffer从写模式调整为读模式；在读模式下，可以读取所有已经写入的数据。

当读取完数据后，需要清空buffer，以满足后续写入操作。清空buffer有两种方式：调用clear()或compact()方法。clear会清空整个buffer，compact则只清空已读取的数据，未被读取的数据会被移动到buffer的开始位置，写入位置则近跟着未读数据之后。

这里有一个简单的buffer案例，包括了write，flip和clear操作：

```
RandomAccessFile aFile = new RandomAccessFile("data/nio-data.txt", "rw");
FileChannel inChannel = aFile.getChannel();

// create buffer with capacity of 48 bytes
ByteBuffer buf = ByteBuffer.allocate(48);

int bytesRead = inChannel.read(buf); //read into buffer.
while (bytesRead != -1) {

  buf.flip();  // make buffer ready for read

  while(buf.hasRemaining()){
      System.out.print((char) buf.get()); // read 1 byte at a time
  }

  buf.clear(); //make buffer ready for writing
  bytesRead = inChannel.read(buf);
}
aFile.close();
```
Buffer的容量，位置，上限（Buffer Capacity, Position and Limit）
buffer缓冲区实质上就是一块内存，用于写入数据，也供后续再次读取数据。这块内存被NIO Buffer管理，并提供一系列的方法用于更简单的操作这块内存。

一个Buffer有三个属性是必须掌握的，分别是：

* capacity容量
* position位置
* limit限制

position和limit的具体含义取决于当前buffer的模式。capacity在两种模式下都表示容量。

下面有张示例图，描诉了不同模式下position和limit的含义：
<div align="center">
    <img src="{{ site.url }}/images/posts/java/nio-buffers-modes.png" alt="nio-buffers-modes"/>
</div>

**容量（Capacity）**

作为一块内存，buffer有一个固定的大小，叫做capacity容量。也就是最多只能写入容量值得字节，整形等数据。一旦buffer写满了就需要清空已读数据以便下次继续写入新的数据。

**位置（Position）**

当写入数据到Buffer的时候需要中一个确定的位置开始，默认初始化时这个位置position为0，一旦写入了数据比如一个字节，整形数据，那么position的值就会指向数据之后的一个单元，position最大可以到capacity - 1。

当从Buffer读取数据时，也需要从一个确定的位置开始。buffer从写入模式变为读取模式时，position会归零，每次读取后，position向后移动。

**上限（Limit）**

在写模式，limit的含义是我们所能写入的最大数据量。它等同于buffer的容量。

一旦切换到读模式，limit则代表我们所能读取的最大数据量，他的值等同于写模式下position的位置。

数据读取的上限时buffer中已有的数据，也就是limit的位置（原position所指的位置）。

**Buffer Types**

Java NIO有如下具体的Buffer类型：

ByteBuffer
MappedByteBuffer
CharBuffer
DoubleBuffer
FloatBuffer
IntBuffer
LongBuffer
ShortBuffer
正如你看到的，Buffer的类型代表了不同数据类型，换句话说，Buffer中的数据可以是上述的基本类型；

**分配一个Buffer（Allocating a Buffer）**

为了获取一个Buffer对象，你必须先分配。每个Buffer实现类都有一个allocate()方法用于分配内存。下面看一个实例,开辟一个48字节大小的buffer：

`ByteBuffer buf = ByteBuffer.allocate(48);`
开辟一个1024个字符的CharBuffer：

`CharBuffer buf = CharBuffer.allocate(1024);`

**写入数据到Buffer（Writing Data to a Buffer）**

写数据到Buffer有两种方法：

* 从Channel中写数据到Buffer。
* 手动写数据到Buffer，调用put方法。
下面是一个实例，演示从Channel写数据到Buffer：
`int bytesRead = inChannel.read(buf); // read into buffer`
通过put写数据：

`buf.put(127);`  
put方法有很多不同版本，对应不同的写数据方法。例如把数据写到特定的位置，或者把一个字节数据写入buffer。看考JavaDoc文档可以查阅的更多数据。

**翻转（flip()）**

flip()方法可以吧Buffer从写模式切换到读模式。调用flip方法会把position归零，并设置limit为之前的position的值。也就是说，现在position代表的是读取位置，limit表示的是已写入的数据位置。

从Buffer读取数据（Reading Data from a Buffer）
冲Buffer读数据也有两种方式。

* 从buffer读数据到channel
* 从buffer直接读取数据，调用get方法
读取数据到channel的例子：
```
// read from buffer into channel.
int bytesWritten = inChannel.write(buf);
```
调用get读取数据的例子：

`byte aByte = buf.get();`  
get也有诸多版本，对应了不同的读取方式。

**rewind()**

Buffer.rewind()方法将position置为0，这样我们可以重复读取buffer中的数据。limit保持不变。

**clear() and compact()**

一旦我们从buffer中读取完数据，需要复用buffer为下次写数据做准备。只需要调用clear或compact方法。

clear方法会重置position为0，limit为capacity，也就是整个Buffer清空。实际上Buffer中数据并没有清空，我们只是把标记为修改了。

如果Buffer还有一些数据没有读取完，调用clear就会导致这部分数据被“遗忘”，因为我们没有标记这部分数据未读。

针对这种情况，如果需要保留未读数据，那么可以使用compact。 因此compact和clear的区别就在于对未读数据的处理，是保留这部分数据还是一起清空。

**mark() and reset()**

通过mark方法可以标记当前的position，通过reset来恢复mark的位置，这个非常像canvas的save和restore：
```
buffer.mark();

// call buffer.get() a couple of times, e.g. during parsing.

buffer.reset();  // set position back to mark.    
```
## Selector
Selector是Java NIO中的一个组件，用于检查一个或多个NIO Channel的状态是否处于可读、可写。如此可以实现单线程管理多个channels,也就是可以管理多个网络链接。

### 为什么使用Selector（Why Use a Selector?）
用单线程处理多个channels的好处是我需要更少的线程来处理channel。实际上，你甚至可以用一个线程来处理所有的channels。从操作系统的角度来看，切换线程开销是比较昂贵的，并且每个线程都需要占用系统资源，因此暂用线程越少越好。

需要留意的是，现代操作系统和CPU在多任务处理上已经变得越来越好，所以多线程带来的影响也越来越小。如果一个CPU是多核的，如果不执行多任务反而是浪费了机器的性能。不过这些设计讨论是另外的话题了。简而言之，通过Selector我们可以实现单线程操作多个channel。

这有一幅示意图，描述了单线程处理三个channel的情况：

<div align="center">
    <img src="{{ site.url }}/images/posts/java/nio-selector.png" alt="nio-selector"/>
</div>

### 创建Selector(Creating a Selector)
创建一个Selector可以通过Selector.open()方法：
`Selector selector = Selector.open();`

### 注册Channel到Selector上(Registering Channels with the Selector)
为了同Selector挂了Channel，我们必须先把Channel注册到Selector上，这个操作使用SelectableChannel.register()：
```
channel.configureBlocking(false);
SelectionKey key = channel.register(selector, SelectionKey.OP_READ);
```
Channel必须是非阻塞的。所以FileChannel不适用Selector，因为FileChannel不能切换为非阻塞模式。Socket channel可以正常使用。

注意register的第二个参数，这个参数是一个“关注集合”，代表我们关注的channel状态，有四种基础类型可供监听：

1. Connect
2. Accept
3. Read
4. Write

一个channel触发了一个事件也可视作该事件处于就绪状态。因此当channel与server连接成功后，那么就是“连接就绪”状态。server channel接收请求连接时处于“可连接就绪”状态。channel有数据可读时处于“读就绪”状态。channel可以进行数据写入时处于“写就绪”状态。

上述的四种就绪状态用SelectionKey中的常量表示如下：
1. SelectionKey.OP_CONNECT
2. SelectionKey.OP_ACCEPT
3. SelectionKey.OP_READ
4. SelectionKey.OP_WRITE

如果对多个事件感兴趣可利用位的或运算结合多个常量，比如：

`int interestSet = SelectionKey.OP_READ | SelectionKey.OP_WRITE;` 
### SelectionKey's
在上一小节中，我们利用register方法把Channel注册到了Selectors上，这个方法的返回值是SelectionKeys，这个返回的对象包含了一些比较有价值的属性：

* The interest set
* The ready set
* The Channel
* The Selector
* An attached object (optional)

**Interest Set**

这个“关注集合”实际上就是我们希望处理的事件的集合，它的值就是注册时传入的参数，我们可以用按为与运算把每个事件取出来：
```
int interestSet = selectionKey.interestOps();

boolean isInterestedInAccept  = interestSet & SelectionKey.OP_ACCEPT;
boolean isInterestedInConnect = interestSet & SelectionKey.OP_CONNECT;
boolean isInterestedInRead    = interestSet & SelectionKey.OP_READ;
boolean isInterestedInWrite   = interestSet & SelectionKey.OP_WRITE; 
```
**Ready Set**

"就绪集合"中的值是当前channel处于就绪的值，一般来说在调用了select方法后都会需要用到就绪状态，select的介绍在胡须文章中继续展开。

`int readySet = selectionKey.readyOps();`

从“就绪集合”中取值的操作类似于“关注集合”的操作，当然还有更简单的方法，SelectionKey提供了一系列返回值为boolean的的方法：
```
selectionKey.isAcceptable();
selectionKey.isConnectable();
selectionKey.isReadable();
selectionKey.isWritable();
```
**Channel + Selector**

从SelectionKey操作Channel和Selector非常简单：
```
Channel  channel  = selectionKey.channel();
Selector selector = selectionKey.selector();   
```
**Attaching Objects**

我们可以给一个SelectionKey附加一个Object，这样做一方面可以方便我们识别某个特定的channel，同时也增加了channel相关的附加信息。例如，可以把用于channel的buffer附加到SelectionKey上：
```
selectionKey.attach(theObject);

Object attachedObj = selectionKey.attachment();
```
附加对象的操作也可以在register的时候就执行：

`SelectionKey key = channel.register(selector, SelectionKey.OP_READ, theObject);`

**从Selector中选择channel(Selecting Channels via a Selector)**

一旦我们向Selector注册了一个或多个channel后，就可以调用select来获取channel。select方法会返回所有处于就绪状态的channel。 select方法具体如下：

* int select()
* int select(long timeout)
* int selectNow()
select()方法在返回channel之前处于阻塞状态。 select(long timeout)和select做的事一样，不过他的阻塞有一个超时限制。

selectNow()不会阻塞，根据当前状态立刻返回合适的channel。

select()方法的返回值是一个int整形，代表有多少channel处于就绪了。也就是自上一次select后有多少channel进入就绪。举例来说，假设第一次调用select时正好有一个channel就绪，那么返回值是1，并且对这个channel做任何处理，接着再次调用select，此时恰好又有一个新的channel就绪，那么返回值还是1，现在我们一共有两个channel处于就绪，但是在每次调用select时只有一个channel是就绪的。

**selectedKeys()**

在调用select并返回了有channel就绪之后，可以通过选中的key集合来获取channel，这个操作通过调用selectedKeys()方法：

`Set<SelectionKey> selectedKeys = selector.selectedKeys(); `   
还记得在register时的操作吧，我们register后的返回值就是SelectionKey实例，也就是我们现在通过selectedKeys()方法所返回的SelectionKey。

遍历这些SelectionKey可以通过如下方法：
```
Set<SelectionKey> selectedKeys = selector.selectedKeys();
Iterator<SelectionKey> keyIterator = selectedKeys.iterator();
while(keyIterator.hasNext()) {
    SelectionKey key = keyIterator.next();
    if(key.isAcceptable()) {
        // a connection was accepted by a ServerSocketChannel.
    } else if (key.isConnectable()) {
        // a connection was established with a remote server.
    } else if (key.isReadable()) {
        // a channel is ready for reading
    } else if (key.isWritable()) {
        // a channel is ready for writing
    }
    keyIterator.remove();
}
```
上述循环会迭代key集合，针对每个key我们单独判断他是处于何种就绪状态。

注意keyIterator.remove()方法的调用，Selector本身并不会移除SelectionKey对象，这个操作需要我们收到执行。当下次channel处于就绪是，Selector任然会吧这些key再次加入进来。

SelectionKey.channel返回的channel实例需要强转为我们实际使用的具体的channel类型，例如ServerSocketChannel或SocketChannel.

**wakeUp()**

由于调用select而被阻塞的线程，可以通过调用Selector.wakeup()来唤醒即便此时已然没有channel处于就绪状态。具体操作是，在另外一个线程调用wakeup，被阻塞与select方法的线程就会立刻返回。

**close()**

当操作Selector完毕后，需要调用close方法。close的调用会关闭Selector并使相关的SelectionKey都无效。channel本身不管被关闭。

**完整的Selector案例**

这有一个完整的案例，首先打开一个Selector,然后注册channel，最后检测Selector的状态：
```
Selector selector = Selector.open();
channel.configureBlocking(false);
SelectionKey key = channel.register(selector, SelectionKey.OP_READ);
while (true) {
    int readyChannels = selector.select();
    if (readyChannels == 0) continue;
    Set<SelectionKey> selectedKeys = selector.selectedKeys();
    Iterator<SelectionKey> keyIterator = selectedKeys.iterator();
    while (keyIterator.hasNext()) {
        SelectionKey key = keyIterator.next();
        if (key.isAcceptable()) {
            // a connection was accepted by a ServerSocketChannel.
        } else if (key.isConnectable()) {
            // a connection was established with a remote server.
        } else if (key.isReadable()) {
            // a channel is ready for reading
        } else if (key.isWritable()) {
            // a channel is ready for writing
        }
        keyIterator.remove();
    }
}
```

## 参考资料
[http://tutorials.jenkov.com/java-nio/index.html](http://tutorials.jenkov.com/java-nio/index.html)