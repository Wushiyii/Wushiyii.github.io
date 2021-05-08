---
title: Netty（一） 
date: 2021-05-08 16:02:07
tags: [JAVA,NETTY]
categories: NETTY
---
Netty基础概念与实现一个Echo服务

<!-- more -->

# Netty（一）

### 基础概念

- Channel -> Socket； 一个Channel在它的生命周期内只注册在一个EventLoop上
- EventLoop -> 控制流、多线程处理、并发；一个EventLoop在它的生命周期内只和一个Thread绑定，一个EventLoop可能会绑定多个Channel。
- ChannelFutre -> 异步通知
- EventLoopGroup ->处理连接的生命周期，包含一个或多个EventLoop
- ChannelFuture -> Netty中所有操作都是异步的，通过ChannelFuture的addListener方法注册channelFutureListenner，可以在操作完成时进行获得通知。

### 实现一个Echo服务

> 客户端将消息发给服务端，服务端把同消息发给客户端

`EchoServer`负责服务端的启动、绑定等，`EchoServerHandler`负责真正的业务逻辑

```java
public class EchoServer {

    private final int port;

    public EchoServer(int port) {
        this.port = port;
    }

    public void start() throws Exception {
        EventLoopGroup group = new NioEventLoopGroup();
        try {
            ServerBootstrap bootstrap = new ServerBootstrap();
            bootstrap.group(group)
                    .channel(NioServerSocketChannel.class) //指定使用NIO传输channel
                    .localAddress(new InetSocketAddress(port)) //使用本地地址
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            ch.pipeline().addLast(new EchoServerHandler());//echoServerHandler被标记了@Sharable 对于所有的客户端，都会使用同一个EchoServerHandler
                        }
                    });

            ChannelFuture channelFuture = bootstrap.bind().sync(); //加上.sync() 的future都会阻塞获取，直到完成
            channelFuture.channel().closeFuture().sync(); //阻塞获取当前bootstrap的channel，直到获取到对应的closeFuture
        } finally {
            group.shutdownGracefully().sync(); //关闭eventLoopGroup
        }
    }

    public static void main(String[] args) throws Exception {
        new EchoServer(7070).start();
    }
}

@ChannelHandler.Sharable
@Slf4j
public class EchoServerHandler extends ChannelInboundHandlerAdapter {

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        ByteBuf in = (ByteBuf) msg;
        log.info("收到消息: {}", in.toString(CharsetUtil.UTF_8));
        ctx.write(in);
    }

    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
        ctx.writeAndFlush(Unpooled.EMPTY_BUFFER)
                .addListener(ChannelFutureListener.CLOSE);
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        cause.printStackTrace();
        ctx.close();
    }
}
```

`EchoClient`负责客户端的启动，`EchoClientHandler`负责服务端的业务逻辑

```java
public class EchoClient {
    private final int port;

    public EchoClient(int port) {
        this.port = port;
    }

    public void start() throws Exception {
        EventLoopGroup eventLoopGroup = new NioEventLoopGroup();
        try {
            Bootstrap bootstrap = new Bootstrap();
            bootstrap.group(eventLoopGroup)
                    .channel(NioSocketChannel.class) //NIO传输，客户端类型
                    .remoteAddress(new InetSocketAddress(port))
                    .handler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            ch.pipeline().addLast(new EchoClientHandler());
                        }
                    });
            ChannelFuture channelFuture = bootstrap.connect().sync();// 连接远程节点，阻塞到完成
            channelFuture.channel().closeFuture().sync(); //阻塞获取closeFuture 直至关闭channel
        } finally {
            eventLoopGroup.shutdownGracefully().sync();
        }
    }

    public static void main(String[] args) throws Exception {
        new EchoClient(7070).start();
    }

}

@ChannelHandler.Sharable
@Slf4j
public class EchoClientHandler extends SimpleChannelInboundHandler<ByteBuf> {

    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        ctx.writeAndFlush(Unpooled.copiedBuffer("Hello~", CharsetUtil.UTF_8));
    }

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, ByteBuf msg) throws Exception {
        log.info("客户端收到消息: {}", msg.toString(CharsetUtil.UTF_8));
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        cause.printStackTrace();
        ctx.close();
    }
}
```

