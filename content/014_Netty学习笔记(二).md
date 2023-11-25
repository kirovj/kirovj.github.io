+++
title = "Netty学习笔记(二)"
date = "2021-07-17"
+++

> Thanks to [https://waylau.gitbooks.io/netty-4-user-guide/content/](https://waylau.gitbooks.io/netty-4-user-guide/content/)
> 
> code at [https://github.com/kirovj/winter](https://github.com/kirovj/winter)

## Make a Time Server & Client

---

### Server

_TimeServer.java_

```java
public class TimeServer {
    // port to listen
    private final int port;

    public TimeServer(int port) {
        this.port = port;
    }

    public void run() throws Exception {
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup)
                    .channel(NioServerSocketChannel.class)
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) {
                            ch.pipeline().addLast(new TimeEncoder(), new TimeServerHandler());
                        }
                    })
                    .option(ChannelOption.SO_BACKLOG, 128)
                    .childOption(ChannelOption.SO_KEEPALIVE, true);

            // 绑定端口，开始接收进来的连接
            ChannelFuture f = b.bind(this.port). sync();

            // 等待服务器  socket 关闭 。
            f.channel().closeFuture().sync();
        } finally {
            workerGroup.shutdownGracefully();
            bossGroup.shutdownGracefully();
        }
    }

    public static void main(String[] args) throws Exception {
        new TimeServer(8080).run();
    }
}
```

> 不同于之前，将UnixTime对象作为数据传输

_UnixTime.java_

```java
public class UnixTime {
    private final long value;

    public UnixTime() {
        this(System.currentTimeMillis() / 1000L + 2208988800L);
    }

    public UnixTime(long value) {
        this.value = value;
    }

    public long value() {
        return value;
    }

    @Override
    public String toString() {
        return new Date((value() - 2208988800L) * 1000L).toString();
    }
}
```

_TimeServerHandler.java_

```java
public class TimeServerHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelActive(ChannelHandlerContext ctx) {
        ChannelFuture f = ctx.writeAndFlush(new UnixTime());
        f.addListener(ChannelFutureListener.CLOSE);
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        cause.printStackTrace();
        ctx.close();
    }
}
```

> 服务端直接返回 new UnixTime()

> 但虽然用对象进行传输，但是底层传输的还是bytes，所以也要对对象进行编码解码

_TimeEncoder.java_

```java
// 编码器,是ChannelOutboundHandler的实现,用来将 UnixTime 对象重新转化为一个 ByteBuf
public class TimeEncoder extends MessageToByteEncoder<UnixTime> {
    @Override
    protected void encode(ChannelHandlerContext ctx, UnixTime msg, ByteBuf out) {
        out.writeInt((int) msg.value());
    }
}
```

---

### Client

_TimeClient.java_

```java
public class TimeClient {
    public static void main(String[] args) throws InterruptedException {
        String host = "127.0.0.1";
        int port = 8080;
        EventLoopGroup workerGroup = new NioEventLoopGroup();

        try {
            // BootStrap 和 ServerBootstrap 类似,不过他是对非服务端的 channel 而言，比如客户端或者无连接传输模式的 channel。
            Bootstrap b = new Bootstrap();
            // 如果你只指定了一个 EventLoopGroup，那他就会即作为一个 boss group ，也会作为一个 workder group，尽管客户端不需要使用到 boss worker 。
            b.group(workerGroup);
            // 代替NioServerSocketChannel的是NioSocketChannel,这个类在客户端channel 被创建时使用。
            b.channel(NioSocketChannel.class);
            // 不像在使用 ServerBootstrap 时需要用 childOption() 方法，因为客户端的 SocketChannel 没有父亲。
            b.option(ChannelOption.SO_KEEPALIVE, true);
            b.handler(new ChannelInitializer<SocketChannel>() {
                @Override
                protected void initChannel(SocketChannel ch) throws Exception {
                    ch.pipeline().addLast(new TimeDecoder(), new TimeClientHandler());
                }
            });

            // 启动客户端 我们用 connect() 方法代替了 bind() 方法。
            ChannelFuture f = b.connect(host, port).sync();

            // 等待连接关闭
            f.channel().closeFuture().sync();
        } finally {
            workerGroup.shutdownGracefully();
        }
    }
}
```

> 注意channel的pipeline中加了一个TimeDecoder

> 服务端编码，客户端解码

_TimeDecoder.java_

```java
public class TimeDecoder extends ByteToMessageDecoder {
    // ByteToMessageDecoder 是 ChannelInboundHandler 的一个实现类，他可以在处理数据拆分的问题上变得很简单
    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
        // 每当有新数据接收的时候，ByteToMessageDecoder 都会调用 decode() 方法来处理内部的那个累积缓冲。
        if (in.readableBytes() < 4) {
            // Decode() 方法可以决定当累积缓冲里没有足够数据时可以往 out 对象里放任意数据
            // 当有更多的数据被接收了 ByteToMessageDecoder 会再一次调用 decode() 方法。
            return;
        }

        // 如果在 decode() 方法里增加了一个对象到 out 对象里，这意味着解码器解码消息成功
        // ByteToMessageDecoder 将会丢弃在累积缓冲里已经被读过的数据
        // 请记得你不需要对多条消息调用 decode()，ByteToMessageDecoder 会持续调用 decode() 直到不放任何数据到 out 里。
        out.add(new UnixTime(in.readUnsignedInt()));
    }
}
```

_TimeClientHandler.java_

```java
public class TimeClientHandler extends ChannelInboundHandlerAdapter {

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        UnixTime time = (UnixTime) msg;
        System.out.println(time);
        ctx.close();
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        cause.printStackTrace();
        ctx.close();
    }
}
```

> 将msg转为UnixTime对象
