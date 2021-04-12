---
typora-root-url: ../../images
---

## bytebuffer

#### bytebuffer的api优点

- 它可以被用户自定义的缓冲区类型扩展;
- 通过内置的复合缓冲区类型实现了透明的零拷贝; 
- 容量可以按需增长(类似于 JDK 的 StringBuilder); 
- 在读和写这两种模式之间切换不需要调用 ByteBuffer 的 flip()方法;
- 读和写使用了不同的索引;
- 支持方法的链式调用;
- 支持引用计数;
- 支持池化。

#### ByteBuf 的使用模式

**1.堆缓冲区**

最常用的 ByteBuf 模式是将数据存储在 JVM 的堆空间中。这种模式被称为支撑数组 (backing array)，它能在没有使用池化的情况下提供快速的分配和释放

**2.直接缓冲区**

优点：这主要是为了避免在每次调用本地 I/O 操作之前(或者之后)将缓冲区的内容复 制到一个中间缓冲区(或者从中间缓冲区把内容复制到缓冲区)

缺点：相对于基于堆的缓冲区，它们的分配和释放都较为昂贵

**3.复合缓冲区**

Netty 通过一个 ByteBuf 子类——CompositeByteBuf——实现了这个模式，它提供了一 个将多个缓冲区表示为单个合并缓冲区的虚拟表示。



#### **字节操作**

```java
ByteBuf buffer = ...;
for (int i = 0; i < buffer.capacity(); i++) {
byte b = buffer.getByte(i);
System.out.println((char)b); }
```

**读取操作**：read 或者 skip 开头的操作都将检索或者跳过位于当前 readerIndex 的数据，并且将它**增加已读字节数**

```java
ByteBuf buffer = ...;
   while (buffer.isReadable()) {
   System.out.println(buffer.readByte());
}
```

**写入操作**：任何名称以 write 开头的操作都将从当前的 writerIndex 处 开始写数据，并将它增加已经写入的字节数

```java
ByteBuf buffer = ...;
   while (buffer.writableBytes() >= 4) {
buffer.writeInt(random.nextInt()); }
```



**discardReadBytes()**：可丢弃字节变为可写--虽然你可能会倾向于频繁地调用 discardReadBytes()方法以确保可写分段的最大化，但是 请注意，这将极有可能会导致内存复制

可以通过调用 **markReaderIndex()**、**markWriterIndex()**、**resetWriterIndex()** 和 **resetReaderIndex()**标记索引位置

**clear()**方法来将 readerIndex 和 writerIndex 都设置为 0--重置索引

**查找**：process(value)、forEachByte(ByteBufProcessor)--常见的值

**派生缓冲区：**duplicate()、slice()、slice(int, int)、Unpooled.unmodifiableBuffer(...)、order(ByteOrder)、readSlice(int)每个这些方法都将返回一个新的 ByteBuf 实例，它具有自己的读索引、写索引和标记索引。都共享索引

**get()**和**set()**操作，从给定的索引开始，并且保持索引不变;
**read()**和**write()**操作，从给定的索引开始，并且会根据已经访问过的字节数对索引进行调整。



## ChannelHandler

#### **ChannelHandler api**

| 类型                                                         | 描述                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| channelRegistered                                            | 当 Channel 已经注册到它的 EventLoop 并且能够处理 I/O 时被调用 |
| channelUnregistered                                          | 当 Channel 从它的 EventLoop 注销并且无法处理任何 I/O 时被调用 |
| channelActive                                                | 当 Channel 处于活动状态时被调用;Channel 已经连接/绑定并且已经就绪 |
| channelInactive                                              | 当 Channel 离开活动状态并且不再连接它的远程节点时被调用      |
| channelReadComplete                                          | 当Channel上的一个读操作完成时被调用                          |
| channelRead                                                  | 当从 Channel 读取数据时被调用                                |
| ChannelWritability- Changed                                  | 当Channel的可写状态发生改变时被调用。用户可以确保写操作不会完成 得太快(以避免发生 OutOfMemoryError)或者可以在 Channel 变为再次可写时恢复写入。可以通过调用Channel的isWritable()方法来检测 Channel 的可写性。与可写性相关的阈值可以通过 Channel.config(). setWriteHighWaterMark()和 Channel.config().setWriteLowWater- Mark()方法来设置 |
| userEventTriggered                                           | 当 ChannelnboundHandler.fireUserEventTriggered()方法被调 用时被调用，因为一个 POJO 被传经了 ChannelPipeline |
| handlerAdded                                                 | 当把 ChannelHandler 添加到 ChannelPipeline 中时被调用        |
| handlerRemoved                                               | 当从 ChannelPipeline 中移除 ChannelHandler 时被调用          |
| exceptionCaught                                              | 当处理过程中在 ChannelPipeline 中有错误产生时被调用          |
| bind(ChannelHandlerContext, SocketAddress,ChannelPromise)    | 当请求将 Channel 绑定到本地地址时被调用                      |
| connect(ChannelHandlerContext, SocketAddress,SocketAddress,ChannelPromise) | 当请求将 Channel 连接到远程节点时被调用                      |
| disconnect(ChannelHandlerContext, ChannelPromise)            | 当请求将 Channel 从远程节点断开时被调用                      |
| close(ChannelHandlerContext,ChannelPromise)                  | 当请求关闭 Channel 时被调用                                  |
| deregister(ChannelHandlerContext, ChannelPromise)            | 当请求将 Channel 从它的 EventLoop 注销 时被调用              |
| read(ChannelHandlerContext)                                  | 当请求从 Channel 读取更多的数据时被调用                      |
| flush(ChannelHandlerContext)                                 | 当请求通过 Channel 将入队数据冲刷到远程                      |
| write(ChannelHandlerContext,Object, ChannelPromise)          | 节点时被调用当请求通过 Channel 将数据写到远程节点时 被调用   |

#### **ChannelPipeline**

| 名称                                                  | 描述                                                         |
| ----------------------------------------------------- | ------------------------------------------------------------ |
| **ChannelPipeline**的用于访问**ChannelHandler**的操作 |                                                              |
| AddFirst、addBefore、<br />addAfter、addLast          | 将一个ChannelHandler添加到ChannelPipeline中                  |
| remove                                                | 将一个 ChannelHandler 从 ChannelPipeline 中移除              |
| replace                                               | 将 ChannelPipeline 中的一个 ChannelHandler 替换为另一个 Channel-Handler |
| get                                                   | 通过类型或者名称返回 ChannelHandler                          |
| context                                               | 返回和 ChannelHandler 绑定的 ChannelHandlerContext           |
| names                                                 | 返回 ChannelPipeline 中所有 ChannelHandler 的名称            |
| **ChannelInboundHandler**                             |                                                              |
| fireChannelRegistered                                 | 调用 ChannelPipeline 中下一个 ChannelInboundHandler 的 channelRegistered(ChannelHandlerContext)方法 |
| fireChannelUnregistered                               | 调用 ChannelPipeline 中下一个 ChannelInboundHandler 的 channelUnregistered(ChannelHandlerContext)方法 |
| fireChannelActive                                     | 调用 ChannelPipeline 中下一个 ChannelInboundHandler 的 channelActive(ChannelHandlerContext)方法 |
| fireChannelInactive                                   | 调用 ChannelPipeline 中下一个 ChannelInboundHandler 的 channelInactive(ChannelHandlerContext)方法 |
| fireExceptionCaught                                   | 调用 ChannelPipeline 中下一个 ChannelInboundHandler 的 exceptionCaught(ChannelHandlerContext, Throwable)方法 |
| fireUserEventTriggered                                | 调用 ChannelPipeline 中下一个 ChannelInboundHandler 的 userEventTriggered(ChannelHandlerContext, Object)方法 |
| fireChannelRead                                       | 调用 ChannelPipeline 中下一个 ChannelInboundHandler 的 channelRead(ChannelHandlerContext, Object msg)方法 |
| fireChannelReadComplete                               | 调用 ChannelPipeline 中下一个 ChannelInboundHandler 的 channelReadComplete(ChannelHandlerContext)方法 |
| fireChannelWritability- Changed                       | 调用 ChannelPipeline 中下一个 ChannelInboundHandler 的 channelWritabilityChanged(ChannelHandlerContext)方法 |
| **ChannelOutboundHandler**                            |                                                              |
| bind                                                  | 将 Channel 绑定到一个本地地址，这将调用 ChannelPipeline 中的下一个 ChannelOutboundHandler 的 bind(ChannelHandlerContext, Socket- Address, ChannelPromise)方法 |
| connect                                               | 将 Channel 连接到一个远程地址，这将调用 ChannelPipeline 中的下一个 ChannelOutboundHandler 的 connect(ChannelHandlerContext, Socket- Address, ChannelPromise)方法 |
| disconnect                                            | 将 Channel 断开连接。这将调用 ChannelPipeline 中的下一个 ChannelOutbound- Handler 的 disconnect(ChannelHandlerContext, Channel Promise)方法 |
| close                                                 | 将 Channel 关闭。这将调用 ChannelPipeline 中的下一个 ChannelOutbound- Handler 的 close(ChannelHandlerContext, ChannelPromise)方法 |
| deregister                                            | 将 Channel 从它先前所分配的 EventExecutor(即 EventLoop)中注销。这将调 用 ChannelPipeline 中的下一个 ChannelOutboundHandler 的 deregister (ChannelHandlerContext, ChannelPromise)方法 |
| flush                                                 | 冲刷 Channel 所有挂起的写入。这将调用 ChannelPipeline 中的下一个 Channel-OutboundHandler 的 flush(ChannelHandlerContext)方法 |
| write                                                 | 将消息写入 Channel。这将调用 ChannelPipeline 中的下一个 Channel-OutboundHandler 的 write(ChannelHandlerContext, Object msg, Channel-Promise)方法。注意:这并不会将消息写入底层的 Socket，而只会将它放入队列中。 要将它写入 Socket，需要调用 flush()或者 writeAndFlush()方法 |
| writeAndFlush                                         | OutboundHandler 的 write(ChannelHandlerContext, Object msg, Channel-这是一个先调用write()方法再接着调用flush()方法的便利方法 |
| read                                                  | 请求从 Channel 中读取更多的数据。这将调用 ChannelPipeline 中的下一个 ChannelOutboundHandler 的 read(ChannelHandlerContext)方法 |
| write                                                 | 通过这个实例写入消息并经过 ChannelPipeline                   |
| writeAndFlush                                         | 通过这个实例写入并冲刷消息并经过 ChannelPipeline             |



**ChannelPipeline模型**

```java
*  +---------------------------------------------------+---------------+
*  |                           ChannelPipeline         |               |
*  |                                                  \|/              |
*  |    +----------------------------------------------+----------+    |
*  |    |                   ChannelHandler  N                     |    |
*  |    +----------+-----------------------------------+----------+    |
*  |              /|\                                  |               |
*  |               |                                  \|/              |
*  |    +----------+-----------------------------------+----------+    |
*  |    |                   ChannelHandler N-1                    |    |
*  |    +----------+-----------------------------------+----------+    |
*  |              /|\                                  .               |
*  |               .                                   .               |
*  | ChannelHandlerContext.fireIN_EVT() ChannelHandlerContext.OUT_EVT()|
*  |          [method call]                      [method call]         |
*  |               .                                   .               |
*  |               .                                  \|/              |
*  |    +----------+-----------------------------------+----------+    |
*  |    |                   ChannelHandler  2                     |    |
*  |    +----------+-----------------------------------+----------+    |
*  |              /|\                                  |               |
*  |               |                                  \|/              |
*  |    +----------+-----------------------------------+----------+    |
*  |    |                   ChannelHandler  1                     |    |
*  |    +----------+-----------------------------------+----------+    |
*  |              /|\                                  |               |
*  +---------------+-----------------------------------+---------------+
*                  |                                  \|/
*  +---------------+-----------------------------------+---------------+
*  |               |                                   |               |
*  |       [ Socket.read() ]                    [ Socket.write() ]     |
*  |                                                                   |
*  |  Netty Internal I/O Threads (Transport Implementation)            |
*  +-------------------------------------------------------------------+
```



#### **ChannelHandlerContext**

要想调用从某个特定的 ChannelHandler 开始的处理过程，必须获取到在(Channel-Pipeline)该 ChannelHandler 之前的 ChannelHandler 所关联的 ChannelHandler-Context。这个 ChannelHandlerContext 将调用和它所关联的 ChannelHandler 之后的 ChannelHandler。

```java
ChannelHandlerContext ctx = ..;
//  write()方法将把缓冲区数据发送 到下一个ChannelHandler
ctx.write(Unpooled.copiedBuffer("Netty in Action", CharsetUtil.UTF_8));
```

**ChannelHandlerContext关系图**

![ChannelHandlerContext](/ChannelHandlerContext.png)

多个 ChannelPipeline 中共享同一 个 ChannelHandler，对应的 ChannelHandler 必须要使用@Sharable 注解标注多个 ChannelPipeline 中共享同一 个 ChannelHandler，对应的 ChannelHandler 必须要使用@Sharable 注解标注



 **ChannelHandlerContext API**

| 方法名称                      | 描述                                                         |
| ----------------------------- | ------------------------------------------------------------ |
| alloc                         | 返回和这个实例相关联的 Channel 所配置的 ByteBufAllocator     |
| bind                          | 绑定到给定的 SocketAddress，并返回 ChannelFuture             |
| channel                       | 返回绑定到这个实例的 Channel                                 |
| close                         | 关闭 Channel，并返回 ChannelFuture                           |
| connect                       | 连接给定的 SocketAddress，并返回 ChannelFuture               |
| deregister                    | 从之前分配的 EventExecutor 注销，并返回 ChannelFuture        |
| disconnect                    | 从远程节点断开，并返回 ChannelFuture                         |
| executor                      | 返回调度事件的 EventExecutor                                 |
| fireChannelActive             | 触发对下一个 ChannelInboundHandler 上的 channelActive()方法(已连接)的调用 |
| fireChannelInactive           | 触发对下一个 ChannelInboundHandler 上的 channelInactive()方法(已关闭)的调用 |
| fireChannelRead               | 触发对下一个 ChannelInboundHandler 上的 channelRead()方法(已接收的消息)的调用 |
| fireChannelReadComplete       | 触发对下一个 ChannelInboundHandler 上的 channelReadComplete()方法的调用 |
| fireChannelRegistered         | 触发对下一个 ChannelInboundHandler 上的 fireChannelRegistered()方法的调用 |
| fireChannelUnregistered       | 触发对下一个 ChannelInboundHandler 上的 fireChannelUnregistered()方法的调用 |
| fireChannelWritabilityChanged | 触发对下一个ChannelInboundHandler上的fireChannelWritabilityChanged()方法的调用 |
| fireExceptionCaught           | 触发对下一个 ChannelInboundHandler 上的 fireExceptionCaught(Throwable)方法的调用 |
| fireUserEventTriggered        | 触发对下一个 ChannelInboundHandler 上的 fireUserEventTriggered(Object evt)方法的调用 |
| handler                       | 返回绑定到这个实例的 ChannelHandler                          |
| isRemoved                     | 如果所关联的 ChannelHandler 已经被从 ChannelPipeline 中移除则返回 true |
| name                          | 返回这个实例的唯一名称                                       |
| pipeline                      | 返回这个实例所关联的 ChannelPipeline                         |
| read                          | 将数据从Channel读取到第一个入站缓冲区;如果读取成功则触发一个channelRead事件，并(在最后一个消息被读取完成后) 通 知 ChannelInboundHandler 的 channelReadComplete (ChannelHandlerContext)方法 |
| write                         | 通过这个实例写入消息并经过 ChannelPipeline                   |
| writeAndFlush                 | 通过这个实例写入并冲刷消息并经过 ChannelPipeline             |



#### 异常处理

ChannelHandler.exceptionCaught()的默认实现是简单地将当前异常转发给

ChannelPipeline 中的下一个 ChannelHandler;
 如果异常到达了 ChannelPipeline 的尾端，它将会被记录为未被处理; 要想定义自定义的处理逻辑，你需要重写 exceptionCaught()方法。然后你需要决定是否需要将该异常传播出去。







## Netty 线程模型

**EventLoop的执行逻辑**如下：

![eventLoop线程模式](/eventLoop线程模式.png)

每个 EventLoop 都有它自已的任务队列。



#### EventLoop/线程的分配

**1.非阻塞传输**

异步传输实现只使用了少量的 EventLoop(以及和它们相关联的 Thread)，而且在当前的线程模型中，它们可能会被多个 Channel 所共享，每一个EventLoop由一个线程处理，用于非阻塞传输(如 NIO 和 AIO)的 EventLoop 分配方式模型图如下。

![eventLoop非阻塞分配方式](/eventLoop非阻塞分配方式.png)

**2.阻塞传输**

这里每一个 Channel 都将被分配给一个 EventLoop(以及它的 Thread)。

![eventLoop阻塞分配方式](/eventLoop阻塞分配方式.png)





## 引导类

**Bootstrap类的API**

| 名称                                                         | 描述                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| Bootstrap group(EventLoopGroup)                              | 设置用于处理 Channel 所有事件的 EventLoopGroup               |
| Bootstrap channel( Class<? extends C>)<br />Bootstrap channelFactory( ChannelFactory<? extends C>) | channel()方法指定了Channel的实现类。如果该实现类 没提供默认的构造函数 1 ，可以通过调用channel- Factory()方法来指定一个工厂类，它将会被bind()方 法调用 |
| Bootstrap localAddress(     SocketAddress)                   | 指定 Channel 应该绑定到的本地地址。如果没有指定， 则将由操作系统创建一个随机的地址。或者，也可以通过 bind()或者 connect()方法指定 localAddress |
| <T> Bootstrap option( ChannelOption<T> option, T value)      | 设置 ChannelOption，其将被应用到每个新创建的 Channel 的 ChannelConfig。这些选项将会通过 bind()或者 connect()方法设置到 Channel，不管哪 个先被调用。这个方法在 Channel 已经被创建后再调用 将不会有任何的效果。支持的 ChannelOption 取决于 使用的 Channel 类型。 |
| <T> Bootstrap attr( Attribute<T> key, T value)               | 指定新创建的 Channel 的属性值。这些属性值是通过 bind()或者 connect()方法设置到 Channel 的，具体 取决于谁最先被调用。这个方法在 Channel 被创建后将 不会有任何的效果 |
| Bootstrap handler(ChannelHandler)                            | 设置将被添加到 ChannelPipeline 以接收事件通知的 ChannelHandler |
| Bootstrap remoteAddress(     SocketAddress)                  | 设置远程地址。或者，也可以通过 connect()方法来指 定它        |
| ChannelFuture connect()                                      | 连接到远程节点并返回一个 ChannelFuture，其将 会在 连接操作完成后接收到通知 |
| ChannelFuture bind()                                         | 绑定Channel并返回一个ChannelFuture，其将会在绑 定操作完成后接收到通知，在那之后必须调用 Channel. |

**使用示例：**

```java
public class BootstrapTest {
    public static void main(String[] args) {
        EventLoopGroup group = new NioEventLoopGroup();
        // 创建一个新 的 Bootstrap 类的实例，以 创建新的客 户端 Channel
        Bootstrap bootstrap = new Bootstrap();
        // 指定一个适用于 NIO 的 EventLoopGroup 实现
        bootstrap.group(group).channel(NioSocketChannel.class).
                // 设置一个用于处理 Channel 的 I/O 事件和数据的 ChannelInboundHandler
                handler(new SimpleChannelInboundHandler<ByteBuf>() {
                    @Override
                    protected void messageReceived(ChannelHandlerContext ctx, ByteBuf msg) throws Exception {
                        System.out.println("Received data");
                    }
                });
				// 尝试连接到 远程节点
        ChannelFuture future = bootstrap.connect(new InetSocketAddress("www.manning.com", 80));
        future.addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture channelFuture) throws Exception {
                if (channelFuture.isSuccess()) {
                    System.out.println("Connection established");
                } else {
                    System.err.println("Connection attempt failed");
                }
            }
        });
    }
}
```



**ServerBootstrap类的api**

| 名称           | 描述                                                         |
| -------------- | ------------------------------------------------------------ |
| group          | 设置 ServerBootstrap 要用的 EventLoopGroup。这个 EventLoopGroup 将用于 ServerChannel 和被接受的子 Channel 的 I/O 处理 |
| channel        | 设置将要被实例化的 ServerChannel 类                          |
| channelFactory | 如果不能通过默认的构造函数 1创建Channel，那么可以提供一个Channel-Factory |
| localAddress   | 指定 ServerChannel 应该绑定到的本地地址。如果没有指定，则将由操作系统使用一个随机地址。或者，可以通过 bind()方法来指定该 localAddress |
| option         | 指定要应用到新创建的 ServerChannel 的 ChannelConfig 的 Channel- Option。这些选项将会通过 bind()方法设置到 Channel。在 bind()方法 被调用之后，设置或者改变 ChannelOption 都不会有任何的效果。所支持 的 ChannelOption 取决于所使用的 Channel 类型。参见正在使用的 ChannelConfig 的 API 文档 |
| childOption    | 指定当子 Channel 被接受时，应用到子 Channel 的 ChannelConfig 的 ChannelOption。所支持的 ChannelOption 取决于所使用的 Channel 的类 型。参见正在使用的 ChannelConfig 的 API 文档 |
| attr           | 指定 ServerChannel 上的属性，属性将会通过 bind()方法设置给 Channel。 在调用 bind()方法之后改变它们将不会有任何的效果 |
| childAttr      | 将属性设置给已经被接受的子 Channel。接下来的调用将不会有任何的效果 |
| handler        | 设置被添加到 ServerChannel 的 ChannelPipeline 中的 ChannelHandler。更加常用的方法参见 childHandler() |
| childHandler   | 设置将被添加到已被接受的子 Channel 的 ChannelPipeline 中的 Channel-Handler。handler()和childHandler()的主要区别是，handler()是发生在**初始化的时候**，childHandler()是发生在**客户端连接之后**。 |
| clone          | 克隆一个设置和原始的 ServerBootstrap 相同的 ServerBootstrap  |
| bind           | 绑定 ServerChannel 并且返回一个 ChannelFuture，其将会在绑定操作完 |

**ServerBootstrap使用**

```java
public class ServerBootstrapTest {
    public static void main(String[] args) {
        NioEventLoopGroup group = new NioEventLoopGroup();
        ServerBootstrap bootstrap = new ServerBootstrap();
        // 设置 EventLoopGroup，其提供了用 于处理 Channel 事件的 EventLoop
        bootstrap.group(group)
                // 指定要使用的 Channel 实现
                .channel(NioServerSocketChannel.class)
           			//  设置用于处理已被接受 的子 Channel 的 I/O 及数据的 ChannelInbound- Handler
                .childHandler(new SimpleChannelInboundHandler<ByteBuf>() {
                    @Override
                    protected void messageReceived(ChannelHandlerContext ctx, ByteBuf msg) throws Exception {
                        System.out.println("Received data");
                    }
                });
        // 通过配置好的ServerBootstrap的实例绑定该Channel
        ChannelFuture future = bootstrap.bind(new InetSocketAddress(8080));
        future.addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture channelFuture)
                    throws Exception {
                if (channelFuture.isSuccess()) {
                    System.out.println("Server bound");
                } else {
                    System.err.println("Bound attempt failed");
                    channelFuture.cause().printStackTrace();
                }
            }
        });
    }
}
```

**Channel 和 EventLoopGroup 的兼容性**

必须保持这种兼容性，不能混用具有不同前缀的组件，如 NioEventLoopGroup 和 OioSocketChannel

![EventLoopGroup和Channel兼容性](/EventLoopGroup和Channel兼容性.png)



**在引导过程中添加多个 ChannelHandler**

```java
public class Example8_6 {
    public static void main(String[] args) throws InterruptedException {
        // 创建ServerBootstrap 以创建和绑定新的Channel
        ServerBootstrap bootstrap = new ServerBootstrap();
        // 设置EventLoopGroup，其将提供用以处理Channel 事件的EventLoop
        bootstrap.group(new NioEventLoopGroup(), new NioEventLoopGroup())
                .channel(NioServerSocketChannel.class)
                // 注册一个ChannelInitializerImpl的实例来设置ChannelPipeline
                .childHandler(new ChannelInitializerImpl());
        // 绑定到地址
        ChannelFuture future = bootstrap.bind(new InetSocketAddress(8080));
        future.sync();
    }

    /**
     * 用以设置ChannelPipeline 的自定义ChannelInitializerImpl 实现
     */
    static final class ChannelInitializerImpl extends ChannelInitializer<Channel> {
        /*
         * 将所需的ChannelHandler添加到ChannelPipeline
         */
        @Override
        protected void initChannel(Channel ch) throws Exception {
            ChannelPipeline pipeline = ch.pipeline();
            pipeline.addLast(new HttpClientCodec());
            pipeline.addLast(new HttpObjectAggregator(Integer.MAX_VALUE));
        }
    }
}
```

**DatagramChannel**

```java
public class DatagramChannelTest {
    public static void main(String[] args) {
        // 创建一个Bootstrap 的实例以创建和绑定新的数据报Channel
        Bootstrap bootstrap = new Bootstrap();
        // 设置EventLoopGroup，其提供了用以处理Channel 事件的EventLoop
        bootstrap.group(new OioEventLoopGroup())
                // 指定Channel的实现
                .channel(OioDatagramChannel.class)
                .handler(
                        // 设置用以处理Channel 的I/O 以及数据的Channel-InboundHandler
                        new SimpleChannelInboundHandler<DatagramPacket>() {
                            @Override
                            protected void messageReceived(ChannelHandlerContext ctx, DatagramPacket msg) {
                                // Do something with the packet
                            }
                        }
                );
        // 调用bind()方法，因为该协议是无连接的
        ChannelFuture future = bootstrap.bind(new InetSocketAddress(0));
        future.addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture channelFuture) {
                if (channelFuture.isSuccess()) {
                    System.out.println("Channel bound");
                } else {
                    System.err.println("Bind attempt failed");
                    channelFuture.cause().printStackTrace();
                }
            }
        });
    }
}
```

**EventLoopGroup关闭**

```java
// 创建处理 I/O 的 EventLoopGroup
EventLoopGroup group = new NioEventLoopGroup();
// 创建一个 Bootstrap 类的实例并配置它
Bootstrap bootstrap = new Bootstrap();
bootstrap.group(group).channel(NioSocketChannel.class);
...
// block until the group has shutdown
// shutdownGracefully()方法将释放所有的资源，并且关闭所有的当前正在使用中的Channel
Future<?> future = group.shutdownGracefully();
future.syncUninterruptibly();
```

shutdownGracefully()方法也是一个异步的操作，所以你需要阻塞等待直到它完成，或者向 所返回的 Future 注册一个监听器以在关闭完成时获得通知。或者，你也可以在调用 EventLoopGroup.shutdownGracefully()方法之前，显式地 在所有活动的 Channel 上调用 Channel.close()方法。但是在任何情况下，都请记得关闭 EventLoopGroup 本身。



## 编解码器

#### 解码器

**ByteToMessageDecoder**

使用示例

```java
public class ToIntegerDecoder extends ByteToMessageDecoder {
    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
        if (in.readableBytes() >= 4) {
            out.add(in.readInt());
        }
    }
}
```

API

| 方法                                                         | 描述                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| decode(<br/>ChannelHandlerContext ctx, ByteBuf in,<br/> List<Object> out) | 这是你必须实现的唯一抽象方法。decode()方法被调用时将会传 入一个包含了传入数据的 ByteBuf，以及一个用来添加解码消息 的 List。对这个方法的调用将会重复进行，直到确定没有新的元 素被添加到该 List，或者该 ByteBuf 中没有更多可读取的字节 时为止。然后，如果该 List 不为空，那么它的内容将会被传递给 ChannelPipeline 中的下一个 ChannelInboundHandler |
| decodeLast( ChannelHandlerContext ctx, ByteBuf in,<br/> List<Object> out) | Netty提供的这个默认实现只是简单地调用了decode()方法。 当Channel的状态变为非活动时，这个方法将会被调用一次。 可以重写该方法以提供特殊的处理 |

**ReplayingDecoder**

使用示例

```java
public class ToIntegerDecoder extends ReplayingDecoder<Void> {
    @Override
    public void decode(ChannelHandlerContext ctx, ByteBuf in,
                       List<Object> out) throws Exception {
        out.add(in.readInt());
    }
}
```

从ByteBuf中提取的int将会被添加到List中。如果没有足够的字节可用，这 个readInt()方法的实现将会抛出一个Error，其将在基类中被捕获并处理

- 并不是所有的 ByteBuf 操作都被支持，如果调用了一个不被支持的方法，将会抛出一个 UnsupportedOperationException;
- ReplayingDecoder 稍慢于 ByteToMessageDecoder

**MessageToMessageDecoder**

```java
class IntegerToStringDecoder extends MessageToMessageDecoder<Integer> {
    @Override
    protected void decode(ChannelHandlerContext ctx, Integer msg, List<Object> out) {
        out.add(String.valueOf(msg));
    }
}
```



#### 编码器

**MessageToByteEncoder**

```java
public class ShortToByteEncoder extends MessageToByteEncoder<Short> {
    @Override
    public void encode(ChannelHandlerContext ctx, Short msg, ByteBuf out) {
        // 将Short 写入ByteBuf 中
        out.writeShort(msg);
    }
}
```

**MessageToMessageEncoder**

```java
static class IntegerToStringEncoder extends MessageToMessageEncoder<Integer> {
    @Override
    public void encode(ChannelHandlerContext ctx, Integer msg, List<Object> out) {
        // 将Integer 转换为String，并将其添加到List中
        out.add(String.valueOf(msg));
    }
}
```



#### 编解码器类

**SslHandler**

通过 SslHandler 进行解密和加密的数据流

![sshHandler](/sshHandler.png)

使用 ChannelInitializer 来将 SslHandler添加到Channel-Pipeline中

```java
public class SslChannelInitializer extends ChannelInitializer<Channel> {
    private final SslContext context;
    private final boolean startTls;
    public SslChannelInitializer(SslContext context, boolean startTls) {
        this.context = context;
        // 如果设置为true，第一个写入的消息将不会被加密客户端应该设置为true
        this.startTls = startTls;
    }
    @Override
    protected void initChannel(Channel ch) throws Exception {
        // 对于每个SslHandler实例，都使用Channel的ByteBuf-Allocator从SslContext获取一个新的SSLEngine
        SSLEngine engine = context.newEngine(ch.alloc());
        // 将SslHandler作为第一个ChannelHandler添加到ChannelPipeline中
        ch.pipeline().addFirst("ssl",new SslHandler(engine, startTls)); 
    }
}
```





#### netty线程模型

**单线程reactor**

![netty单线程Reactor官方](/netty单线程Reactor官方.jpg)

![netty单线程reactor](/netty单线程reactor.png)

```java
 NioEventLoopGroup group = new NioEventLoopGroup(1);
        ServerBootstrap bootstrap = new ServerBootstrap();
        bootstrap.group(group)
                .channel(NioServerSocketChannel.class)
                .channel(NioServerSocketChannel.class)
                .option(ChannelOption.TCP_NODELAY, true)
                .option(ChannelOption.SO_BACKLOG, 1024)
                .childHandler(new ServerHandlerInitializer());

```

**多线程reactor**

模型图：

![netty多线程Reactor官方](/netty多线程Reactor官方.jpg)

![netty多线程reactor](/netty多线程reactor.png)

核心代码：

```java
class Handler implements Runnable {
    // uses util.concurrent thread pool
    static PooledExecutor pool = new PooledExecutor(...);
    static final int PROCESSING = 3;
    // ...
    synchronized void read() { // ...
        socket.read(input);
        if (inputIsComplete()) {
            state = PROCESSING;
            pool.execute(new Processer());
        } }
    synchronized void processAndHandOff() { process();
        state = SENDING; // or rebind attachment sk.interest(SelectionKey.OP_WRITE);
    }
    class Processer implements Runnable {
        public void run() { processAndHandOff(); }
    }
}
```



```java
NioEventLoopGroup eventGroup = new NioEventLoopGroup();
ServerBootstrap bootstrap = new ServerBootstrap();
bootstrap.group(eventGroup)
        .channel(NioServerSocketChannel.class)
        .option(ChannelOption.TCP_NODELAY, true)
        .option(ChannelOption.SO_BACKLOG, 1024)
        .childHandler(new ServerHandlerInitializer());

```

**多Reactor主从**

![netty多reactor主从官方](/netty多reactor主从官方.jpg)

![netty多reactor主从](/netty多reactor主从.png)主从多线程模型是有多个Reactor，也就是存在多个selector，所以我们定义一个bossGroup和一个workGroup，核心代码如下

```java
NioEventLoopGroup bossGroup = new NioEventLoopGroup();
NioEventLoopGroup workerGroup = new NioEventLoopGroup();
ServerBootstrap bootstrap = new ServerBootstrap();
bootstrap.group(bossGroup,workerGroup)
        .channel(NioServerSocketChannel.class)
        .option(ChannelOption.TCP_NODELAY, true)
        .option(ChannelOption.SO_BACKLOG, 1024)
        .childHandler(new ServerHandlerInitializer());

```

