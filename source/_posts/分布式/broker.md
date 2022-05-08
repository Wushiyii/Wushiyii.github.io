---
title: RocketMQ-Broker存储消息源码分析
date: 2022-05-08 15:05:22
tags: [distribute, MQ]
categories: distribute
---

Broker存储消息源码分析

<!-- more -->

# broker 保存消息流程

首先是接收请求消息的入口，与`producer`发送消息接收返回的代码一致，不过请求类型为服务端主动请求
```java
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

这段代码大概意思是调用业务线程池处理请求
```java
public void processRequestCommand(final ChannelHandlerContext ctx, final RemotingCommand cmd) {
    //从处理器table中根据请求码获取到处理器， pair为默认请求处理器
    final Pair<NettyRequestProcessor, ExecutorService> pair = null == this.processorTable.get(cmd.getCode()) ? this.defaultRequestProcessor : matched;
    final int opaque = cmd.getOpaque();

    Runnable run = new Runnable() {
        @Override
        public void run() {
            try {

                //消息执行完的回调（其实就是调用Netty把返回值发送回去）
                final RemotingResponseCallback callback = new RemotingResponseCallback() {
                    @Override
                    public void callback(RemotingCommand response) {
                        doAfterRpcHooks(RemotingHelper.parseChannelRemoteAddr(ctx.channel()), cmd, response);
                        //非oneway类型请求，会有响应
                        if (!cmd.isOnewayRPC()) {
                            response.setOpaque(opaque);
                            response.markResponseType();
                            //发送响应消息
                            ctx.writeAndFlush(response);
                        }
                    }
                };
                //调用执行器处理请求
                RemotingCommand response = pair.getObject1().processRequest(ctx, cmd);

                //执行回调
                callback.callback(response);
            }
            //。。。
        }
    };
        //这里才是调用线程池执行处理请求的地方
        final RequestTask requestTask = new RequestTask(run, ctx.channel(), cmd);
        pair.getObject2().submit(requestTask);
}
```
由于`producer`在发送消息的时候，指明了消息code是`10`（SEND_MESSAGE），在`BrokerController.registerProcessor()`这个方法里面就有`10`对应的处理器
```java
    public void registerProcessor() {
        /**
         * SendMessageProcessor
         */
        SendMessageProcessor sendProcessor = new SendMessageProcessor(this);
        sendProcessor.registerSendMessageHook(sendMessageHookList);
        sendProcessor.registerConsumeMessageHook(consumeMessageHookList);

        this.remotingServer.registerProcessor(RequestCode.SEND_MESSAGE, sendProcessor, this.sendMessageExecutor);
        this.remotingServer.registerProcessor(RequestCode.SEND_MESSAGE_V2, sendProcessor, this.sendMessageExecutor);
        this.remotingServer.registerProcessor(RequestCode.SEND_BATCH_MESSAGE, sendProcessor, this.sendMessageExecutor);
        this.remotingServer.registerProcessor(RequestCode.CONSUMER_SEND_MSG_BACK, sendProcessor, this.sendMessageExecutor);
        //...
    }
```
下面开始是`broker`中处理消息的核心逻辑了，代码较多，会分多个小块解析。

### DefaultMessageStore保存消息入口
调用入口是`DefaultMessageStore.putMessage()`，这个在`SendMessageProcessor.processRequest()`里面处理消息的地方可以找到。
这段代码较简单，主要是校验功能
```java
@Override
public CompletableFuture<PutMessageResult> asyncPutMessage(MessageExtBrokerInner msg) {
    ////检擦当前broker状态（是否启动、是否为Slave、是否可写、是否繁忙）
    PutMessageStatus checkStoreStatus = this.checkStoreStatus();

    //检查消息是否合法
    PutMessageStatus msgCheckStatus = this.checkMessage(msg);

    //调用commitLog服务存储消息
    CompletableFuture<PutMessageResult> putResultFuture = this.commitLog.asyncPutMessage(msg);
    
    //...
}
```

### commitLog执行保存消息
主要为调用mmap服务保存消息，以及处理刷盘、主从同步等
```java
public CompletableFuture<PutMessageResult> asyncPutMessage(final MessageExtBrokerInner msg) {

    //自旋锁
    putMessageLock.lock();
    try {
        //取最新的mmap文件，若不存在则创建一个
        MappedFile mappedFile = this.mappedFileQueue.getLastMappedFile(0); 
        //append消息到mmap对象
        AppendMessageResult result = mappedFile.appendMessage(msg, this.appendMessageCallback);
        switch (result.getStatus()) {
            //。。。
        }
    } finally {
        putMessageLock.unlock();
    }

    //flush刚刚append到mmap文件上到消息
    CompletableFuture<PutMessageStatus> flushResultFuture = submitFlushRequest(result, msg);
    
    //复制到其他机器（DLedger或者简单主从同步）
    CompletableFuture<PutMessageStatus> replicaResultFuture = submitReplicaRequest(result, msg);
    
    //。。。
}
```
> 代码较多，mmap等地方会比较复杂，下面会进行拆分
### 获取/创建 Mmap文件
上文中`getLastMappedFile(0)`这个函数，会尝试获取当前mmap文件列表的最后一个文件，若获取不到则会创建文件并生产mmap映射
```java
public MappedFile getLastMappedFile(final long startOffset, boolean needCreate) {

    //文件大小偏移量，初识时为0
    long createOffset = -1;
    
    //尝试获取最后一个mmap文件
    MappedFile mappedFileLast = getLastMappedFile();

    //若文件不存在，则偏移量为: 0 +1073741824 = 1073741824; (1G = 1024 * 1024 * 1024)
    if (mappedFileLast != null && mappedFileLast.isFull()) {
        createOffset = mappedFileLast.getFileFromOffset() + this.mappedFileSize;
    }

    if (createOffset != -1 && needCreate) {
        //创建的文件路径，如 /home/data/store/00000000001073741824
        String nextFilePath = this.storePath + File.separator + UtilAll.offset2FileName(createOffset);

        //下一个待创建的文件路径 如 /home/data/store/00000000002147483648
        String nextNextFilePath = this.storePath + File.separator
                + UtilAll.offset2FileName(createOffset + this.mappedFileSize);
        MappedFile mappedFile = null;

        //如果有指定分配mmap文件的服务，则使用指定的服务创建（这个服务可以异步处理分配内存）
        if (this.allocateMappedFileService != null) {
            mappedFile = this.allocateMappedFileService.putRequestAndReturnMappedFile(nextFilePath,
                    nextNextFilePath, this.mappedFileSize);
        } else {
            //默认创建方式
            mappedFile = new MappedFile(nextFilePath, this.mappedFileSize);
        }
        return mappedFile;
    }
    return mappedFileLast;
}

public MappedFile putRequestAndReturnMappedFile(String nextFilePath, String nextNextFilePath, int fileSize) {
        
    //可以允许请求分配的请求数（其实就是bufferPool数量限制），默认为2原因是要分配当前文件和下一个文件
    int canSubmitRequests = 2;
    if (this.messageStore.getMessageStoreConfig().isTransientStorePoolEnable()) {
        if (this.messageStore.getMessageStoreConfig().isFastFailIfNoBufferInStorePool()
            && BrokerRole.SLAVE != this.messageStore.getMessageStoreConfig().getBrokerRole()) {
            //最大同时创建buffer数 - 等待处理分配请求数 = 可分配数
            canSubmitRequests = this.messageStore.getTransientStorePool().availableBufferNums() - this.requestQueue.size();
        }
    }

    //允许分配数小于等于0，则直接返回不让分配
    if (canSubmitRequests <= 0) {
        this.requestTable.remove(nextFilePath);
        return null;
    }
    //允许分配，把分配请求放入优先队列中（文件名小的优先）
    AllocateRequest nextReq = new AllocateRequest(nextFilePath, fileSize);
    boolean offerOK = this.requestQueue.offer(nextReq);
    canSubmitRequests--;

    //取出当前文件的分配请求
    AllocateRequest result = this.requestTable.get(nextFilePath);

    //CountDownLatch超时5秒，另一边有监听优先队列的实现
    boolean waitOK = result.getCountDownLatch().await(waitTimeOut, TimeUnit.MILLISECONDS);
    if (waitOK) {
        this.requestTable.remove(nextFilePath);
        return result.getMappedFile();
    }
}

```
上面这一段，解析了broker如何拿到mmap文件和构造创建mmapd请求队列，下面开始讲执行监听mmap请求队列到实现
`AllocateMappedFileService`实现了`Runnable`，而其中的`run()`方法则会在这个服务启动的时候执行
```java
public void run() {
    //只要服务不停止，则一直进行mmap操作
    while (!this.isStopped() && this.mmapOperation()) {}
}

private boolean mmapOperation() {

    //获取队列里的消息
    AllocateRequest req = this.requestQueue.take();
    long beginTime = System.currentTimeMillis();

    MappedFile mappedFile;
    if (messageStore.getMessageStoreConfig().isTransientStorePoolEnable()) {
        //使用SPI方式加载MappedFile服务
        mappedFile = ServiceLoader.load(MappedFile.class).iterator().next();

        //执行mmap文件初始化与映射
        mappedFile.init(req.getFilePath(), req.getFileSize(), messageStore.getTransientStorePool());
    } else {
        //与上面类似，都是调用mapperFile里面的init方法
        mappedFile = new MappedFile(req.getFilePath(), req.getFileSize());
    }

    // 预热mmap文件，一般情况不开
    if (mappedFile.getFileSize() >= this.messageStore.getMessageStoreConfig()
            .getMappedFileSizeCommitLog()
            &&
            this.messageStore.getMessageStoreConfig().isWarmMapedFileEnable()) {
        mappedFile.warmMappedFile(this.messageStore.getMessageStoreConfig().getFlushDiskType(),
                this.messageStore.getMessageStoreConfig().getFlushLeastPagesWhenWarmMapedFile());
    }

    req.setMappedFile(mappedFile);
    return true;
}

//执行mmap，代码不多，主要是NIO干活
private void init(final String fileName, final int fileSize) throws IOException {
    this.fileName = fileName;
    this.fileSize = fileSize;
    this.file = new File(fileName);
    this.fileFromOffset = Long.parseLong(this.file.getName());

    //检查文件上级目录，无目录则创建
    ensureDirOK(this.file.getParent());

    //构造随机读写对象
    this.fileChannel = new RandomAccessFile(this.file, "rw").getChannel();
    //使用fileChannel执行map映射
    this.mappedByteBuffer = this.fileChannel.map(MapMode.READ_WRITE, 0, fileSize);
}
```

### mmap存储执行append消息
上面讲了一大堆铺垫，其实最终干活的还是mmap映射对象。`mappedFile.appendMessage()`这段是发送消息的入口
```java
public AppendMessageResult doAppend(final long fileFromOffset, final ByteBuffer byteBuffer, final int maxBlank,
        final MessageExtBrokerInner msgInner) {

        //查询topic+queueID组合下的offset
        Long queueOffset = CommitLog.this.topicQueueTable.get(key);

        //消息类型
        final int tranType = MessageSysFlag.getTransactionValue(msgInner.getSysFlag());

        //序列化消息
        final byte[] propertiesData =
            msgInner.getPropertiesString() == null ? null : msgInner.getPropertiesString().getBytes(MessageDecoder.CHARSET_UTF8);

        //各种字段长度
        final int propertiesLength = propertiesData == null ? 0 : propertiesData.length;
        final byte[] topicData = msgInner.getTopic().getBytes(MessageDecoder.CHARSET_UTF8);
        final int topicLength = topicData.length;
        final int bodyLength = msgInner.getBody() == null ? 0 : msgInner.getBody().length;
        final int msgLen = calMsgLength(msgInner.getSysFlag(), bodyLength, topicLength, propertiesLength);

        // 记录消息的各个字段与长度到一个buffer中
        this.resetByteBuffer(msgStoreItemMemory, msgLen);
        // 1 TOTALSIZE
        this.msgStoreItemMemory.putInt(msgLen);
        // 2 MAGICCODE
        this.msgStoreItemMemory.putInt(CommitLog.MESSAGE_MAGIC_CODE);
        // xxx 10几个字段设置

        //写入消息到buffer中
        byteBuffer.put(this.msgStoreItemMemory.array(), 0, msgLen);

        //返回结果
        AppendMessageResult result = new AppendMessageResult(AppendMessageStatus.PUT_OK, wroteOffset, msgLen, msgId,
            msgInner.getStoreTimestamp(), queueOffset, CommitLog.this.defaultMessageStore.now() - beginTimeMills);

        switch (tranType) {
            case:
                //...
            case MessageSysFlag.TRANSACTION_NOT_TYPE:
            case MessageSysFlag.TRANSACTION_COMMIT_TYPE:
                // 普通消息和事务消息到提交消息、记录存储的偏移量
                CommitLog.this.topicQueueTable.put(key, ++queueOffset);
                break;
        }
        return result;
    }
```
### 同步刷盘 && 异步刷盘
至此，消息终于存放到了buffer中，但是消息是如何落盘的？开启了同步刷屏的实现在哪？
在上面的`asyncPutMessage`中，最下面执行了`submitFlushRequest`，而如果配置了同步刷盘，这个逻辑就会被执行
```java
public CompletableFuture<PutMessageStatus> submitFlushRequest(AppendMessageResult result, MessageExt messageExt) {
    // 同步刷盘
    if (FlushDiskType.SYNC_FLUSH == this.defaultMessageStore.getMessageStoreConfig().getFlushDiskType()) {
        final GroupCommitService service = (GroupCommitService) this.flushCommitLogService;
        if (messageExt.isWaitStoreMsgOK()) {
            GroupCommitRequest request = new GroupCommitRequest(result.getWroteOffset() + result.getWroteBytes(),
                    this.defaultMessageStore.getMessageStoreConfig().getSyncFlushTimeout());
            //执行
            service.putRequest(request);
        } 
        //...
    }
    // 异步刷盘
    else {
        if (!this.defaultMessageStore.getMessageStoreConfig().isTransientStorePoolEnable()) {
            flushCommitLogService.wakeup();
        } else  {
            commitLogService.wakeup();
        }
    }
}

class GroupCommitService extends FlushCommitLogService {

    public void run() {

        while (!this.isStopped()) {
            //这边使用CountDownLatch阻塞获取请求，获取到了会调用onWaitEnd()切换请求模式
            this.waitForRunning(10);

            //提交请求
            this.doCommit();
        }

        Thread.sleep(10);
        synchronized (this) {
            //切换读写请求
            this.swapRequests();
        }
        //再提交一次，这一次模式换了一个
        this.doCommit();
    }

}

private void doCommit() {
    synchronized (this.requestsRead) {
        if (!this.requestsRead.isEmpty()) {
            for (GroupCommitRequest req : this.requestsRead) {
                //...

                //执行flush
                CommitLog.this.mappedFileQueue.flush(0);
            }
        }
    }
}


public int flush(final int flushLeastPages) {
    //如果当前mmap文件可以执行flush操作， hold方法调用意味着占用这个flush操作
    if (this.isAbleToFlush(flushLeastPages) && this.hold()) {
        
        //直接调用mmap的force方法刷到磁盘
        if (writeBuffer != null || this.fileChannel.position() != 0) {
            this.fileChannel.force(false);
        } else {
            this.mappedByteBuffer.force();
        }

        //释放占用
        this.release();
    }
    return this.getFlushedPosition();
}
```
上面是同步刷盘的逻辑，异步刷盘又有何不同呢？异步刷盘调用的服务是`FlushCommitLogService`，调用方式与上面大同小异
```java
class FlushRealTimeService extends FlushCommitLogService {

    public void run() {
        while (!this.isStopped()) {
            //其实就是每隔xxx秒，刷一次
            if (flushCommitLogTimed) {
                Thread.sleep(500);
            } else {
                this.waitForRunning(interval);
            }

            CommitLog.this.mappedFileQueue.flush(flushPhysicQueueLeastPages);

        }
}

```

### 最后
至此，`broker`的消息存储代码已完结，里面删除了很多分散注意力的代码，详情可以参考源代码再看一遍，有很多并发的手段（通知、信号量、异步线程等）