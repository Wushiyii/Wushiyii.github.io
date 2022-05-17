---
title: RocketMQ-Consumer拉取消费源码分析
date: 2022-05-17 22:49:10
tags: [distribute, MQ]
categories: distribute
---

Consumer拉取消费源码分析

<!-- more -->

### 创建并启动`DefaultMQPushConsumer`
这是一段`RocketMq`自带的消费者启动测试代码，我们可以跟着这个流程一步步分析消费者做了什么
```java
//创建消费者
DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("please_rename_unique_group_name");

//设置订阅关系
consumer.subscribe("TagFilterTest", "TagA || TagC");

//注册消费者实现
consumer.registerMessageListener(new MessageListenerConcurrently() {
    @Override
    public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs,
        ConsumeConcurrentlyContext context) {
        System.out.printf("%s Receive New Messages: %s %n", Thread.currentThread().getName(), msgs);
        return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
    }
});

//启动消费者
consumer.start();
```

#### 创建消费者
首先从创建开始，`new DefaultMQPushConsumer("testConsumerGroup")`，一路往下回到这里，其中namespace默认为空，`AllocateMessageQueueStrategy`为消费策略
```java
public DefaultMQPushConsumer(final String namespace, final String consumerGroup, RPCHook rpcHook,
    AllocateMessageQueueStrategy allocateMessageQueueStrategy) {
    this.consumerGroup = consumerGroup;
    this.namespace = namespace;
    this.allocateMessageQueueStrategy = allocateMessageQueueStrategy;
    defaultMQPushConsumerImpl = new DefaultMQPushConsumerImpl(this, rpcHook);
}

//简单构造，配置注入
public DefaultMQPushConsumerImpl(DefaultMQPushConsumer defaultMQPushConsumer, RPCHook rpcHook) {
    this.defaultMQPushConsumer = defaultMQPushConsumer;
    this.rpcHook = rpcHook;
    this.pullTimeDelayMillsWhenException = defaultMQPushConsumer.getPullTimeDelayMillsWhenException();
}
```
`AllocateMessageQueueStrategy`里一共有6个实现,一般情况使用默认实现即可
- **AllocateMachineRoomNearby**: 同机房分配策略
- **AllocateMessageQueueAveragely**: 平均分配（默认）
- **AllocateMessageQueueAveragelyByCircle**: 轮询分配
- **AllocateMessageQueueByConfig**: 指定配置消费
- **AllocateMessageQueueByMachineRoom**: 机房分配
- **AllocateMessageQueueConsistentHash**: 一致性哈希


> 设置订阅关系与注册消费者较简单，此处省略
#### 启动消费者


```java
public synchronized void start() throws MQClientException {
    //检查基本配置（消费组名称、订阅关系、消费线程数等）
    this.checkConfig();

    //重新赋值一遍订阅关系（重试消息的topic为:%Retry%+consumerGroup，订阅所有的tag）
    this.copySubscription();

    //构造MQ客户端实例
    this.mQClientFactory = MQClientManager.getInstance().getOrCreateMQClientInstance(this.defaultMQPushConsumer, this.rpcHook);

    //设置消费进度服务
    switch (this.defaultMQPushConsumer.getMessageModel()) {
        //广播模式进度存在本地
        case BROADCASTING:
            this.offsetStore = new LocalFileOffsetStore(this.mQClientFactory, this.defaultMQPushConsumer.getConsumerGroup());
            break;
        //集群模式需要通知到broker
        case CLUSTERING:
            this.offsetStore = new RemoteBrokerOffsetStore(this.mQClientFactory, this.defaultMQPushConsumer.getConsumerGroup());
            break;
    }
    this.defaultMQPushConsumer.setOffsetStore(this.offsetStore);

    //并发消费服务
    this.consumeMessageService = new ConsumeMessageConcurrentlyService(this, (MessageListenerConcurrently) this.getMessageListenerInner());
    this.consumeMessageService.start();

    //启动实例
    mQClientFactory.start();

    //更新订阅关系（会通知到broker）
    this.updateTopicSubscribeInfoWhenSubscriptionChanged();
    
    //心跳通知broker
    this.mQClientFactory.sendHeartbeatToAllBrokerWithLock();
    
    //负载关系均衡
    this.mQClientFactory.rebalanceImmediately();
}

// mQClientFactory.start() 启动实例
public void start() throws MQClientException {

        // 启动netty客户端
    this.mQClientAPIImpl.start();
    // 启动所有定时任务（更新topic路由信息、发送心跳、调整消费线程大小等）
    this.startScheduledTask();
    // 拉取消息服务
    this.pullMessageService.start();
    // 定时重负载均衡服务
    this.rebalanceService.start();

    //...
}
```

#### 拉取消息
上面在`consumer`启动的时候，也启动了`pullMessageService`，而这个service就是拉取消息的核心方法。
```java
@Override
public void run() {
    while (!this.isStopped()) {
        //拉取请求阻塞队列，如果队列为空，则阻塞
        PullRequest pullRequest = this.pullRequestQueue.take();
        this.pullMessage(pullRequest);
    }

    log.info(this.getServiceName() + " service end");
}
```
`pullMessage`会调用多个方法，最后调用netty请求到`broker`，默认每次拉32个消息.
```java
//构造拉取请求
PullMessageRequestHeader requestHeader = new PullMessageRequestHeader();
requestHeader.setConsumerGroup(this.consumerGroup);
requestHeader.setTopic(mq.getTopic());
requestHeader.setQueueId(mq.getQueueId());
//。。。

PullResult pullResult = this.mQClientFactory.getMQClientAPIImpl().pullMessage(findBrokerResult.getBrokerAddr(),
    requestHeader,
    timeoutMillis,
    communicationMode,
    pullCallback);

return pullResult;
```

### 消费消息
在上面拉取消息的实现里，在拉取消息执行完成时，会调用一个callback执行处理，这里会将代码放入到一个`TreeMap`中，另有线程处理这些消息
```java
//未拉取到消息会马上再拉一次
if (pullResult.getMsgFoundList() == null || pullResult.getMsgFoundList().isEmpty()) {
    DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
} else {

    //把请求到入到消息处理队列中
    boolean dispatchToConsume = processQueue.putMessage(pullResult.getMsgFoundList());

    //根据返回情况是否直接处理消息
    DefaultMQPushConsumerImpl.this.consumeMessageService.submitConsumeRequest(
        pullResult.getMsgFoundList(),
        processQueue,
        pullRequest.getMessageQueue(),
        dispatchToConsume);

    //拉取消息后，立即再次拉取消息
    if (DefaultMQPushConsumerImpl.this.defaultMQPushConsumer.getPullInterval() > 0) {
        DefaultMQPushConsumerImpl.this.executePullRequestLater(pullRequest,
            DefaultMQPushConsumerImpl.this.defaultMQPushConsumer.getPullInterval());
    } else {
        //马上拉取
        DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
    }
}

public void submitConsumeRequest() {
    final int consumeBatchSize = this.defaultMQPushConsumer.getConsumeMessageBatchMaxSize();
    for (int total = 0; total < msgs.size(); ) {
            
        //一次组装多个msg，可以配置批量消费消息数
        List<MessageExt> msgThis = new ArrayList<MessageExt>(consumeBatchSize);
        for (int i = 0; i < consumeBatchSize; i++, total++) {
            if (total < msgs.size()) {
                msgThis.add(msgs.get(total));
            }
        }
        
        //异步处理,提交到线程池
        ConsumeRequest consumeRequest = new ConsumeRequest(msgThis, processQueue, messageQueue);
        this.consumeExecutor.submit(consumeRequest);

    }
}
```
上面终于把拉取的消息缓存到了本地，下面开始执行消费。逻辑很简答，只是执行启动`consumer`时注册的listener。
这段就是业务代码执行消费的逻辑了，
```java
class ConsumeRequest implements Runnable {

    @Override
    public void run() {
        //构造消息context
        ConsumeConcurrentlyContext context = new ConsumeConcurrentlyContext(messageQueue);
        
        //使用启动消费者时注册的listener执行消费逻辑
        MessageListenerConcurrently listener = ConsumeMessageConcurrentlyService.this.messageListener;
        ConsumeConcurrentlyStatus status = listener.consumeMessage(Collections.unmodifiableList(msgs), context);
        //判断status消费状态，执行消费成功、重消费等逻辑 
        // 。。。

        //执行消费消息后的操作
        if (!processQueue.isDropped()) {
            ConsumeMessageConcurrentlyService.this.processConsumeResult(status, context, this);
        }
    }

}
```
#### 消费完成后的处理
在上面我们看到，消息已经交给业务代码执行，执行完成后调用了`processConsumeResult`，这个函数主要是把本地缓存的临时消息清除，以及更新`broker`端的消息完成`offset`
```java
public void processConsumeResult() {

    //处理消费失败的消息，会将消费失败的消息发给broker
    for (int i = ackIndex + 1; i < consumeRequest.getMsgs().size(); i++) {
        MessageExt msg = consumeRequest.getMsgs().get(i);
        //通知broker消费失败消息
        this.sendMessageBack(msg, context);
    }

    //从消息树中移除消息
    long offset = consumeRequest.getProcessQueue().removeMessage(consumeRequest.getMsgs());

    //更新消费成功便宜量offset
    if (offset >= 0 && !consumeRequest.getProcessQueue().isDropped()) {
        this.defaultMQPushConsumerImpl.getOffsetStore().updateOffset(consumeRequest.getMessageQueue(), offset, true);
    }
}

//removeMessage代码比较简单，就是根据每一个msg的offset，从消息树中remove
public long removeMessage(final List<MessageExt> msgs) {
    //开启读写锁
    //把消息从消息树中去除，无论成功失败
    for (MessageExt msg : msgs) {
        MessageExt prev = msgTreeMap.remove(msg.getQueueOffset());
    }
}

//更新本地的消费进度，有另一个线程会执行，而这个线程在consumer启动的时候就启动了
public void updateOffset(MessageQueue mq, long offset, boolean increaseOnly) {
    //更新本地的消费进度偏移量
    AtomicLong offsetOld = this.offsetTable.get(mq);
    MixAll.compareAndIncreaseOnly(offsetOld, offset);
}
```

#### 通知broker消费进度
在启动消费者时，有一个`startScheduledTask()`,这个方法会开启多个线程执行任务，其中有一个定时保存消费进度的任务
```java

this.scheduledExecutorService.scheduleAtFixedRate(() ->
    MQClientInstance.this.persistAllConsumerOffset(),
    1000 * 10, 
    this.clientConfig.getPersistConsumerOffsetInterval(), 
    TimeUnit.MILLISECONDS);
```
而这个线程最终会调用`offsetStore.persistAll()`
```java
public void persistAll(Set<MessageQueue> mqs) {
        
    //消费者监听的每一个队列
    for (Map.Entry<MessageQueue, AtomicLong> entry : this.offsetTable.entrySet()) {
        MessageQueue mq = entry.getKey();
        AtomicLong offset = entry.getValue();
        
        //通知broker更新消费者的消费进度
        this.updateConsumeOffsetToBroker(mq, offset.get());
    }
}
```

至此结束了消息消费流程，里面删了些部分主流程不用关心的代码，总体而言`consumer`的代码较为简单，比较复杂的tcp交互、拉取等待的逻辑都在`broker`完成了。