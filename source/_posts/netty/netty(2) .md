---
title: Netty（二） 
date: 2021-05-08 18:53:07
tags: [JAVA,NETTY]
categories: NETTY
---
Netty组件：ChannelHandlerChannelPipeline的关系

<!-- more -->

# Netty（二）

#### ChannelHandler

`ChannelHander`充当了所有处理入站和出战的逻辑处理容器，比如`ChannelInboundHandler`是一个处理接收入站信息的容器，`ChannelOutboundHandler`是处理出战信息的容器。一般而言，程序处理的业务逻辑通常会在多个`ChannelInboundHandler`。



#### ChannelPipeline

`ChannelPipeline`即是一个链式处理各个`ChannelHandler`的流水管道。入站和出站的`ChannelHandler`

可以被加到同一个`ChannelPipeline`中，当一个消息的入站事件被读取，`ChannelPipeline`既会开始各个handler的流动，最终达到`ChannePipeline`的尾端，届时，所有的处理就都结束了。

出站流程和入站几乎无差别，不过数据的流动是从`ChannelPipeline`的尾端开始流动，直到到达头部为止。

> 入站与出战的`ChannelHandler`的类型是不同的，所以即使把这些处理器都加入到同一个pipeline，Netty也能区分开的，确保数据只会在相同类型的`ChannelHandler`之间流转。



#### 编码器与解码器

当使用Netty接收与发送一个消息时，就会发生一次数据转换。接收到入站消息，需要从字节类型转换成特定的报文格式（一般是Java对象）；接收到出战消息，则需要将当前的格式转换成字节类型。

这两种类型的转换，对应的抽象类为：入站(`ByteToMessageDecoder`)，出站(`MessageToByteEncoder`)。



#### SimpleChannelInboundHandler

`SimpleChannelInboundHandler`可以处理对应泛型<I>的消息，要注如果在pipeline链上，前一个`ChannelHandler`传过来的消息，必须和<I>指定的类型一致，否则不会解析消息。