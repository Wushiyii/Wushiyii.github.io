---
title: RocketMQ-Producer发送消息源码分析
date: 2022-05-06 23:03:01
tags: [distribute, MQ]
categories: distribute
---

Producer发送消息源码分析

<!-- more -->

### 发送消息过程

发送的入口主要是`DefaultMqProducer.send()`,这个方法经过一层层包装，会调用到`DefaultMQProducerImpl.sendDefaultImpl()`,下面简单分析下这些代码做了什么事。

```java
private SendResult sendDefaultImpl(Message msg, 
        final CommunicationMode communicationMode,
        final SendCallback sendCallback, 
        final long timeout) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {

        //确认服务状态正常
        this.makeSureStateOK();

        //校验消息(topic是否合法，是否缺失body等)
        Validators.checkMessage(msg, this.defaultMQProducer);

        //随机生产一个请求ID
        final long invokeID = random.nextLong();

        //开始、结束时间等
        long beginTimestampFirst = System.currentTimeMillis();
        long beginTimestampPrev = beginTimestampFirst;
        long endTimestamp = beginTimestampFirst;

        //根据topic查询namesrv上的路由信息，返回结果为某个Topic在某个broker下，broker下又有哪些brokerId和地址，以及broker下的queue分布、配置
        TopicPublishInfo topicPublishInfo = this.tryToFindTopicPublishInfo(msg.getTopic());
        if (topicPublishInfo != null && topicPublishInfo.ok()) {
            boolean callTimeout = false;//是否超时
            MessageQueue mq = null;//broker中的队列
            Exception exception = null;//是否异常
            SendResult sendResult = null;//发送结果

            //同步请求默认请求次数为3，要有重试功能首先得开启
            int timesTotal = communicationMode == CommunicationMode.SYNC ? 1 + this.defaultMQProducer.getRetryTimesWhenSendFailed() : 1;
            int times = 0;//请求次数记录
            String[] brokersSent = new String[timesTotal];
            for (; times < timesTotal; times++) {

                //上一次发送的broker名称（只会在3次重试内出现）
                String lastBrokerName = null == mq ? null : mq.getBrokerName();

                //随机取一条队列（随机数与队列长度取模作为index，当然也要考虑排除上一次路由的broker名称）
                MessageQueue mqSelected = this.selectOneMessageQueue(topicPublishInfo, lastBrokerName);
                if (mqSelected != null) {
                    mq = mqSelected;
                    brokersSent[times] = mq.getBrokerName();//记录当前第n次发送时对应的broker名称
                    try {
                        beginTimestampPrev = System.currentTimeMillis();
                        if (times > 0) {
                            msg.setTopic(this.defaultMQProducer.withNamespace(msg.getTopic()));
                        }
                        //判断超时
                        long costTime = beginTimestampPrev - beginTimestampFirst;
                        if (timeout < costTime) {
                            callTimeout = true;
                            break;
                        }

                        //执行发送消息
                        sendResult = this.sendKernelImpl(msg, mq, communicationMode, sendCallback, topicPublishInfo, timeout - costTime);

                        //如果为同步模式，会直接返回结果
                        switch (communicationMode) {
                            case ASYNC:
                                return null;
                            case ONEWAY:
                                return null;
                            case SYNC:
                                return sendResult;
                            default:
                                break;
                        }
                    }
                }
            }
    }
```

此处主要逻辑是组装消息（Header、事务消息tag等）
```java
private SendResult sendKernelImpl(final Message msg, final MessageQueue mq, final CommunicationMode communicationMode,
    final SendCallback sendCallback, final TopicPublishInfo topicPublishInfo,
    final long timeout) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {

    long beginStartTime = System.currentTimeMillis();

    //mq为随机选的一个队列
    String brokerAddr = this.mQClientFactory.findBrokerAddressInPublish(mq.getBrokerName());
    SendMessageContext context = null;

    //消息原数据
    byte[] prevBody = msg.getBody();
    try {
        //非批量消息，需要生成一个唯一ID
        if (!(msg instanceof MessageBatch)) {
            MessageClientIDSetter.setUniqID(msg);
        }

        int sysFlag = 0;
        boolean msgBodyCompressed = false;
        
        //如果消息大小大于4k，会尝试进行压缩
        if (this.tryToCompressMessage(msg)) {
            sysFlag |= MessageSysFlag.COMPRESSED_FLAG;
            msgBodyCompressed = true;
        }

        //事务消息标识
        final String tranMsg = msg.getProperty(MessageConst.PROPERTY_TRANSACTION_PREPARED);
        if (tranMsg != null && Boolean.parseBoolean(tranMsg)) {
            sysFlag |= MessageSysFlag.TRANSACTION_PREPARED_TYPE;
        }

        //消息发送hook(发送前、后执行的回调)
        if (this.hasSendMessageHook()) {
            //。。。 组装context等
            this.executeSendMessageHookBefore(context);
        }

        //组装消息header头信息
        SendMessageRequestHeader requestHeader = new SendMessageRequestHeader();
        requestHeader.setProducerGroup(this.defaultMQProducer.getProducerGroup());
        requestHeader.setTopic(msg.getTopic());
        requestHeader.setDefaultTopic(this.defaultMQProducer.getCreateTopicKey());
        requestHeader.setDefaultTopicQueueNums(this.defaultMQProducer.getDefaultTopicQueueNums());
        requestHeader.setQueueId(mq.getQueueId());
        requestHeader.setSysFlag(sysFlag);
        requestHeader.setBornTimestamp(System.currentTimeMillis());
        requestHeader.setFlag(msg.getFlag());
        requestHeader.setProperties(MessageDecoder.messageProperties2String(msg.getProperties()));
        requestHeader.setReconsumeTimes(0);
        requestHeader.setUnitMode(this.isUnitMode());
        requestHeader.setBatch(msg instanceof MessageBatch);
        //如果为重试的消息，则发送重试次数、最大重试次数
        if (requestHeader.getTopic().startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
            String reconsumeTimes = MessageAccessor.getReconsumeTime(msg);
            if (reconsumeTimes != null) {
                requestHeader.setReconsumeTimes(Integer.valueOf(reconsumeTimes));
                MessageAccessor.clearProperty(msg, MessageConst.PROPERTY_RECONSUME_TIME);
            }

            String maxReconsumeTimes = MessageAccessor.getMaxReconsumeTimes(msg);
            if (maxReconsumeTimes != null) {
                requestHeader.setMaxReconsumeTimes(Integer.valueOf(maxReconsumeTimes));
                MessageAccessor.clearProperty(msg, MessageConst.PROPERTY_MAX_RECONSUME_TIMES);
            }
        }

        SendResult sendResult = null;
        switch (communicationMode) {
            case ASYNC:
                Message tmpMessage = msg;
                //如果消息压缩过，需要clone到临时对象，把原值还原
                if (msgBodyCompressed) {
                    tmpMessage = MessageAccessor.cloneMessage(msg);
                    msg.setBody(prevBody);
                }

                long costTimeAsync = System.currentTimeMillis() - beginStartTime;
                //调用Netty客户端接口，发送消息
                sendResult = this.mQClientFactory.getMQClientAPIImpl().sendMessage(
                    brokerAddr,
                    mq.getBrokerName(),
                    tmpMessage,
                    requestHeader,
                    timeout - costTimeAsync,
                    communicationMode,
                    sendCallback,
                    topicPublishInfo,
                    this.mQClientFactory,
                    this.defaultMQProducer.getRetryTimesWhenSendAsyncFailed(),
                    context,
                    this);
                break;
            case ONEWAY:
            case SYNC:
                //同步消息相比异步，不用传callback进去
                long costTimeSync = System.currentTimeMillis() - beginStartTime;
                sendResult = this.mQClientFactory.getMQClientAPIImpl().sendMessage(
                    brokerAddr,
                    mq.getBrokerName(),
                    msg,
                    requestHeader,
                    timeout - costTimeSync,
                    communicationMode,
                    context,
                    this);
                break;
            default:
                assert false;
                break;
        }

        //执行发送消息后的回调
        if (this.hasSendMessageHook()) {
            context.setSendResult(sendResult);
            this.executeSendMessageHookAfter(context);
        }

        return sendResult;
```
下面看一看`getMQClientAPIImpl().sendMessage()`这一块的实现，主要就是区分三种请求方式，执行不同的发送方法（但是后面的发送方法又耦合在一起，迷惑）
```java
    long beginStartTime = System.currentTimeMillis();
    RemotingCommand request = null;//其实就是请求体
    
    //普通消息和批量消息走不同的RequestCode
    if (sendSmartMsg || msg instanceof MessageBatch) {
        SendMessageRequestHeaderV2 requestHeaderV2 = SendMessageRequestHeaderV2.createSendMessageRequestHeaderV2(requestHeader);
        request = RemotingCommand.createRequestCommand(msg instanceof MessageBatch ? RequestCode.SEND_BATCH_MESSAGE : RequestCode.SEND_MESSAGE_V2, requestHeaderV2);
    } else {
        //又组装一遍报文到RequestCommand对象中
        request = RemotingCommand.createRequestCommand(RequestCode.SEND_MESSAGE, requestHeader);
    }
    request.setBody(msg.getBody());

    //三种执行方式
    switch (communicationMode) {
        case ONEWAY:
            //直接发送，不关心返回
            this.remotingClient.invokeOneway(addr, request, timeoutMillis);
            return null;
        case ASYNC:
            //异步请求、会在请求完成后处理callback
            final AtomicInteger times = new AtomicInteger();
            long costTimeAsync = System.currentTimeMillis() - beginStartTime;
            this.sendMessageAsync(addr, brokerName, msg, timeoutMillis - costTimeAsync, request, sendCallback, topicPublishInfo, instance,
                retryTimesWhenSendFailed, times, context, producer);
            return null;
        case SYNC:
            //同步请求，利用CountDownLatch阻塞请求，直到对方返回或超时
            long costTimeSync = System.currentTimeMillis() - beginStartTime;
            return this.sendMessageSync(addr, brokerName, msg, timeoutMillis - costTimeSync, request);
        default:
            assert false;
            break;
}
```

三种请求方式里重点看一下同步请求,代码照样删了些不重要的
> 下面代码只写了发送以及等待，哪里触发`CountDownLatch`释放呢？
```java
    //构造TCP返回对象
    final ResponseFuture responseFuture = new ResponseFuture(channel, opaque, timeoutMillis, null, null);
    //以唯一请求码为key放入一个临时map中
    this.responseTable.put(opaque, responseFuture);
    final SocketAddress addr = channel.remoteAddress();
    //直接调用netty的发送方法
    channel.writeAndFlush(request).addListener(new ChannelFutureListener() {
        @Override
        public void operationComplete(ChannelFuture f) throws Exception {
            if (f.isSuccess()) {
                responseFuture.setSendRequestOK(true);
                return;
            } else {
                responseFuture.setSendRequestOK(false);
            }
        }
    });

    //通过CountDownLatch阻塞等待请求结果
    RemotingCommand responseCommand = responseFuture.waitResponse(timeoutMillis);
    return responseCommand;
```

上面提了什么时候触发`CountDownLatch`的释放，其实熟悉Netty的话就会在client的pipeline中找到答案
在`NettyRemotingClient.java`中，pipeline中加了几个处理器，大概为编码、解码、空闲关联、连接关联以及最重要的请求处理器(`NettyClientHandler`)
```java
    pipeline.addLast(
                    defaultEventExecutorGroup,
                    new NettyEncoder(),
                    new NettyDecoder(),
                    new IdleStateHandler(0, 0, nettyClientConfig.getClientChannelMaxIdleTimeSeconds()),
                    new NettyConnectManageHandler(),
                    new NettyClientHandler());
```
其中`NettyClientHandler`继承了`SimpleChannelInboundHandler`，意为只处理入站请求。里面的`channelRead0`就是处理请求的入口
```java
class NettyClientHandler extends SimpleChannelInboundHandler<RemotingCommand> {

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, RemotingCommand msg) throws Exception {
        processMessageReceived(ctx, msg);
    }
}


public void processMessageReceived(ChannelHandlerContext ctx, RemotingCommand msg) throws Exception {
        final RemotingCommand cmd = msg;
    //请求分为两类： 一类是服务端主动发送的请求，一类是服务端发送响应的请求
    switch (cmd.getType()) {
        //服务端主动发送
        case REQUEST_COMMAND:
            processRequestCommand(ctx, cmd);
            break;
        //服务端响应的请求
        case RESPONSE_COMMAND:
            processResponseCommand(ctx, cmd);
            break;
        default:
            break;
    }
}
```
- 服务端主动发送的请求：这类看client处于哪个服务中，例如在`namesrv`中`producer`过来拉取topic路由。
- 服务端返回响应的请求：代表是响应客户端的请求，这也是我们在现在流程要关注的。

```java
public void processResponseCommand(ChannelHandlerContext ctx, RemotingCommand cmd) {
    final int opaque = cmd.getOpaque();
    //根据唯一请求ID，获取到请求table中待处理的future对象
    final ResponseFuture responseFuture = responseTable.get(opaque);
    if (responseFuture != null) {
        //设置返回报文
        responseFuture.setResponseCommand(cmd);
        responseTable.remove(opaque);

        //如果是异步请求，执行回调
        if (responseFuture.getInvokeCallback() != null) {
            executeInvokeCallback(responseFuture);
        } else {
            //其他清空，释放CountDownLatch、释放信号量
            responseFuture.putResponse(cmd);
            responseFuture.release();
        }
    } else {
        log.warn("receive response, but not matched any request, " + RemotingHelper.parseChannelRemoteAddr(ctx.channel()));
        log.warn(cmd.toString());
    }
}
```

至此，`producer`的发送流程结束，没有特别多逻辑，主要是组装报文，通过Netty请求到服务端，还有一些并发与锁控制流量与响应等。
