---
title: 一文了解Netty的使用
date: 2023-02-05 00:10:00
theme: cyanosis
highlight: atom-one-dark-reasonable
tags:
 - netty
 - 网络编程
categories:
 - 网络编程
cover: https://img.fansqz.com/img/55da18c34cf24b1bafcf37d3fddcf4ab.jpg
---
# 一文了解Netty的使用ls

## helloworld

使用前需要导入netty依赖

~~~xml
<dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty-all</artifactId>
    <version>4.1.79.Final</version>
</dependency>
~~~

**hello-server**

~~~java
public static void main(String[] args) {
    // ServerBootstrap是一个启动器，负责组装netty组件
    new ServerBootstrap()
        .group(new NioEventLoopGroup())
        // 选择 服务器的serverSocketChannel实现
        .channel(NioServerSocketChannel.class)
        // 添加处理的具体逻辑
        .childHandler(
        	// channel代表了和客户端进行数据读写的通道，以下是负责添加Handler
       		new ChannelInitializer<NioSocketChannel>() {
            @Override
            protected void initChannel(NioSocketChannel ch) throws Exception {
                ch.pipeline().addLast(new StringDecoder());
                // 自定义handler
                ch.pipeline().addLast(new ChannelInboundHandlerAdapter() {
                    @Override
                    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
                        System.out.println(msg);
                    }
                });
            }
        }).bind(8080);
}
~~~

**hello-client**

~~~java
public static void main(String[] args) throws InterruptedException {
    Channel channel = new Bootstrap()
        // 添加EventLoop
        .group(new NioEventLoopGroup())
        .channel(NioSocketChannel.class)
        // 添加处理器
        .handler(new ChannelInitializer<NioSocketChannel>() {
            @Override
            protected void initChannel(NioSocketChannel ch) throws Exception {
                ch.pipeline().addLast(new StringEncoder());
            }
        })
        .connect(new InetSocketAddress("localhost", 8080))
        .sync()
        .channel();
    // 向服务端发送数据
    channel.writeAndFlush("hello world");
}
~~~

- channel可以理解为数据的通道
- 把msg时流动的数据，一开始输入的时ByteBuf，但是经过pipeline的加工，会变成其他类型对象，最后输出又变成ByteBuf
- 把handler理解为数据的处理工序
  - 工序有多道，合在一起就是pipeline，pipeline负责发布时间传递给每个handler，handler对自己感兴趣的事件进行处理
  - handler分为Inbound和Outbound两类

- evenLoop理解为处理数据的工人
  - 工人可以管理多个channel的io操作，并且一旦工人负责了某个channel，就要负责到底（绑定）
  - 工人既可以执行io操作，也可以进行任务处理，每位工人有任务队列，队列可以堆放多个channel的待处理任务，任务分为普通任务、定时任务
  - 工人按照pipeline的顺序，依次按照handler的规划（代码）处理数据，可以为每道工序指定不同的工人

## 组件

### EventLoop

**EventLoop**

所谓的EventLoop就是一个事件循环对象，是一个的单线程执行器（同时维护了一个Selector）,里面有run方法处理Channel上源源不断的io事件

它的继承关系比较复杂

- 一条线是继承值juc.ScheduledExcecutorService，因此包含了线程池中所有方法
- 另一条线是继承自netty自己实现的OrderedEventExecutor

**EventLoopGroup**

EventLoopGroup是一组EventLoop，channel一般会调用EventLoopGroup的register方法绑定一个EventLoop，后续这个Channel上的io事件都由此EventLoop来处理



~~~java
// 1. 可以指定线程数，默认是cpu核数*2
EventLoopGroup group = new NioEventLoopGroup(2);
// 2. 通过next可以获取下一个事件循环对象
System.out.println(group.next());
// 3. 执行普通任务
group.next().submit(() -> {
    System.out.println("执行任务");
});
// 4. 执行定时任务，任务，初始延时事件，间隔事件，时间单位
group.next().scheduleAtFixedRate(() -> {
    System.out.println("执行");
},0, 1, TimeUnit.SECONDS);
// 5. 执行io事件
EventLoopGroup group2 = new DefaultEventLoop();
new ServerBootstrap()
    // 可以用连个事件循环组 boss和worker
    // boss用于处理 accept事件
    // worker用于吃醋里socketChannel上的读写事件
    .group(new NioEventLoopGroup(), new NioEventLoopGroup())
    .channel(NioServerSocketChannel.class)
    .childHandler(new ChannelInitializer<NioSocketChannel>() {
        @Override
        protected void initChannel(NioSocketChannel ch) {
            ch.pipeline().addLast(new ChannelInboundHandlerAdapter(){
                @Override
                public void channelRead(ChannelHandlerContext ctx, Object msg) {
                    ByteBuf buf = (ByteBuf) msg;
                    System.out.println(buf.toString(Charset.defaultCharset()));
                    ctx.fireChannelRead(msg);
                }
            });
            ch.pipeline().addLast(group2, new ChannelInboundHandlerAdapter(){
                @Override
                public void channelRead(ChannelHandlerContext ctx, Object msg) {
                    System.out.println("给group2执行的handler");
                }
            });
        }
    }).bind(8080);
~~~



### Channel

channel主要作用

~~~java
close() 		//关闭channel
closeFutrue() 	//用来处理channel的关闭
pipeline() 		//方法添加处理器
write() 		//方法将数据添加到缓冲区，并不会立即发送数
writeAndFlush() //将数据写入并刷新，会立即发送给服务端
~~~

**关于ChannelFutrue**

连接是非阻塞的，会返回一个ChannelFutrue，通过ChannelFuture来获取channel。

~~~java
ChannelFuture channelFuture = (ChannelFuture) new Bootstrap()
    .group(new NioEventLoopGroup())
    .channel(NioSocketChannel.class)
    .handler(new ChannelInitializer<NioSocketChannel>() {
        @Override
        protected void initChannel(NioSocketChannel ch) throws Exception {
            ch.pipeline().addLast(new StringEncoder());
        }
    })
    // 连接到服务器，异步非阻塞
    .connect(new InetSocketAddress("localhost", 8080));

// 1. 阻塞等待连接建立完毕
/**
channelFuture.sync();
// 获取连接
Channel channel = channelFuture.channel();
channel.writeAndFlush("hello world");
*/

// 2. 通过回调方法处理结果
channelFuture.addListener(new ChannelFutureListener() {
    @Override
    public void operationComplete(ChannelFuture future) throws Exception {
        Channel channel = future.channel();
        channel.writeAndFlush("hello world");
		//3. 可以获取CloseFutrue对象，可以在连接关闭的时候做一些事情
        ChannelFuture closeFuture = channel.closeFuture();
        closeFuture.addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                System.out.println("连接关闭");
            }
        });
    }
});
~~~



### handler & pipeline

ChannelHanderl用于处理Channel上各种事件，分为入站、出站。所有ChannelHandler被连成一串就是Pipeline

- 入站ChannelInboundHanderAdapter，用于读取客户端数据，写回结果
- 出站ChannelOutboundHandlerAdapter，用于写回结果进行加工

~~~java
new ServerBootstrap()
    .group(new NioEventLoopGroup())
    .channel(NioServerSocketChannel.class)
    .childHandler(new ChannelInitializer<NioSocketChannel>() {
        @Override
        protected void initChannel(NioSocketChannel ch) {
            ch.pipeline().addLast(new ChannelInboundHandlerAdapter() {
                @Override
                public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
                    System.out.println("1");
                    super.channelRead(ctx, msg);
                }
            });
            ch.pipeline().addLast(new ChannelInboundHandlerAdapter() {
                @Override
                public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
                    System.out.println("2");
                    ch.writeAndFlush(ctx.alloc().buffer().writeBytes("hello".getBytes()));
                }
            });
            ch.pipeline().addLast(new ChannelOutboundHandlerAdapter() {
                @Override
                public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
                    System.out.println("3");
                    super.write(ctx, msg, promise);
                }
            });
            ch.pipeline().addLast(new ChannelOutboundHandlerAdapter(){
                @Override
                public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
                    System.out.println("4");
                    super.write(ctx, msg, promise);
                }
            });
        }
    }).bind(8080);
~~~

- head -> h1 -> h2 -> h3 -> h4 -> tail，当调用channerl.write会从tail向前去找channelOutboundHandlerAdapter，如果调用context.write则会从当前handler向前找。

### byteBuf

**基本使用**

~~~java
// 默认256大小，且可以动态扩容
ByteBuf buf = ByteBufAllocator.DEFAULT.buffer();
StringBuilder stringBuilder = new StringBuilder();
for (int i = 0; i < 300; i++) {
    stringBuilder.append("a");
}
buf.writeBytes(stringBuilder.toString().getBytes());
System.out.println(buf);
~~~

**直接内存和堆内存**

可以申请直接内存：

~~~java
ByteBuf buf = ByteBufAllocator.DEFAULT.directBuffer();
~~~

也可以申请堆内存

~~~java
ByteBuf buf = ByteBufAllocator.DEFAULT.heapBuffer();
~~~

- 直接内存创建和销毁的代价高，但是读写性能高，配合池化功能一起用
- 直接内存对于GC压力小，因为这部分内存不受JVM垃圾回收管理，但也要注意及时主动释放

**池化和非池化**

池化的可以重用ByteBuf

- 不需要每次都创建新ByteBuf实例，可以重用ByteBuf实例，并且采用与jemalloc类似内存分配算法提升分配效率
- 高并发时，池化功能更节约内存，减少内存溢出可能

**组成**

![image-20230204155951023](https://img.fansqz.com/img/image-20230204155951023.png)

一共有四个部分

- 读指针，写指针，容量，最大容量

- 灰色部分是废弃部分，绿色部分是可读部分，蓝色部分是可写部分，橘色部分是可控部分

**回收**

byteBuf有许多不同的实现，不同的实现需要使用不同的方法进行回收

- UnpooledHeapByteBuf 使用的是JVM内存，只需要等待GC回收即可
- UnpooledDirectByteBuf 使用的是直接内存，需要特殊的方法来回收
- PooledByteBuf 和它的子类使用了池化机制，回收更复杂

Nettt 采用了引用计数来控制回收内存，每个ByteBuf都实现了ReferenceCounted接口

- 每个ByteBuf对象初始计数为1
- 调用release方法计数减1，如果计数为0，ByteBuf内存被回收
- 调用retain方法计数加1，表示调用者没用完之前，其他handler即使调用了release也不会造成回收
- 当计数为0时，底层内存会被回收，即使ByteBuf对象还在，其各个方法均无法使用

**Slice**

零拷贝的体现之一，对于原始的ByteBuf进行切片成多个ByteBuf，切片后的ByteBuf并没有发送内存的复制，而是使用原始ByteBuf的内存，切片后的ByteBuf维护独立的read，write指针

![image-20230204164156233](https://img.fansqz.com/img/image-20230204164156233.png)

~~~java
ByteBuf buf = ByteBufAllocator.DEFAULT.directBuffer();
buf.writeBytes("abcdefg".getBytes());
ByteBuf buf1 = buf.slice(0,2);
ByteBuf buf2 = buf.slice(2, 5);
~~~

注：

- 由于使用的是同一块空间，所以切片以后的ByteBuf不能超过容量大小，为了防止切片写入超过容量大小，影响其他切片。超过容量大小会抛出异常。

- 如果释放原ByteBuf会导致切片的ByteBuf无法使用，所以创建切片的时候，最好调用retrain()方法使引用计数+1。

  ~~~java
  ByteBuf buf = ByteBufAllocator.DEFAULT.directBuffer();
  buf.writeBytes("abcdefg".getBytes());
  buf.retain();
  ByteBuf buf1 = buf.slice(0,2);
  buf.retain();
  ByteBuf buf2 = buf.slice(2, 5);
  buf.release();
  buf.release();
  ~~~

**CompositeBuffer**

可以将多个小的ByteBuf合并成一个大的ByteBuf，而不发生数据的拷贝

~~~java
ByteBuf buf1 = ByteBufAllocator.DEFAULT.buffer();
buf1.writeBytes("123456".getBytes());
ByteBuf buf2 = ByteBufAllocator.DEFAULT.buffer();
buf2.writeBytes("6789".getBytes());
CompositeByteBuf buf = ByteBufAllocator.DEFAULT.compositeBuffer();
// true意味着让写指针自动增长
buf.addComponents(true, buf1, buf2);
~~~

