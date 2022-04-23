---
title: RocketMQ-Broker启动流程源码分析
date: 2022-04-23 23:15:12
tags: [distribute, MQ]
categories: distribute
---

Broker启动流程源码分析

<!-- more -->
`Broker`的启动流程，和`Namesrv`类似，也是分`Contoller`的**创建**和**启动**。
> 代码较多，会适当删除或者修改无需特意关注的代码（例如解析command之类的）


### 创建Controller

```java
public static BrokerController createBrokerController(String[] args) {
    //broker基本配置，主要为namesrv地址、brokerId、brokerName等
    final BrokerConfig brokerConfig = new BrokerConfig();

    //同namesrv，存放netty的默认配置
    final NettyServerConfig nettyServerConfig = new NettyServerConfig();
    //Netty客户端配置
    final NettyClientConfig nettyClientConfig = new NettyClientConfig();

    //是否使用TLS
    nettyClientConfig.setUseTLS(Boolean.parseBoolean(System.getProperty(TLS_ENABLE,
        String.valueOf(TlsSystemConfig.tlsMode == TlsMode.ENFORCING))));
    nettyServerConfig.setListenPort(10911);

    //存放消息的配置，主要为存放地址、mmap配置、flush间隔时间等
    final MessageStoreConfig messageStoreConfig = new MessageStoreConfig();

    //Slave节点，消息占用内存比率为30%
    if (BrokerRole.SLAVE == messageStoreConfig.getBrokerRole()) {
        int ratio = messageStoreConfig.getAccessMessageInMemoryMaxRatio() - 10;
        messageStoreConfig.setAccessMessageInMemoryMaxRatio(ratio);
    }

    //如果开启Dleger同步功能，则每个brokerId都为-1
    if (messageStoreConfig.isEnableDLegerCommitLog()) {
        brokerConfig.setBrokerId(-1);
    }

    //构造BrokerController，里面有大量Manager、Config等代码，用到在看
    final BrokerController controller = new BrokerController(
        brokerConfig, nettyServerConfig, nettyClientConfig, messageStoreConfig);

    //初始化controller，主要为初始化Netty、设置Dleger、初始化各个线程池等
    boolean initResult = controller.initialize();

    return controller;
}
```



初始化消息存储服务、Netty、消息处理线程池、RPC钩子等

```java
public boolean initialize() throws CloneNotSupportedException {

    //初始化各个manager
    //topicConfigManager : 主要为保存各个topic的信息，当前的配置文件版本等.文件位置:{user.home}/store/config/topics.json
    //consumerOffsetManager : 消费进度管理类，可以查询或提交某个topic的消费进度. 文件位置:{user.home}/store/config/consumerOffset.json
    //subscriptionGroupManager : 订阅关系管理类。文件位置:{user.home}/store/config/subscriptionGroup.json
    //consumerFilterManager : 消费者过滤信息管理类  topic分组。文件位置：{user.home}/store/config/consumerFilter.json
    boolean result = this.topicConfigManager.load();
    result = result && this.consumerOffsetManager.load();
    result = result && this.subscriptionGroupManager.load();
    result = result && this.consumerFilterManager.load();

    if (result) {
        try {
            //构造默认消息存储服务
            this.messageStore =
                new DefaultMessageStore(this.messageStoreConfig, this.brokerStatsManager, this.messageArrivingListener,
                    this.brokerConfig);
            //如果开启DLeger一致性功能，会默认创建一个
            if (messageStoreConfig.isEnableDLegerCommitLog()) {
                DLedgerRoleChangeHandler roleChangeHandler = new DLedgerRoleChangeHandler(this, (DefaultMessageStore) messageStore);
                ((DLedgerCommitLog)((DefaultMessageStore) messageStore).getCommitLog()).getdLedgerServer().getdLedgerLeaderElector().addRoleChangeHandler(roleChangeHandler);
            }
            //Broker统计工具类
            this.brokerStats = new BrokerStats((DefaultMessageStore) this.messageStore);
            //加载消息存储插件
            MessageStorePluginContext context = new MessageStorePluginContext(messageStoreConfig, brokerStatsManager, messageArrivingListener, brokerConfig);
            this.messageStore = MessageStoreFactory.build(context, this.messageStore);
            this.messageStore.getDispatcherList().addFirst(new CommitLogDispatcherCalcBitMap(this.brokerConfig, this.consumerFilterManager));
        } catch (IOException e) {
            result = false;
            log.error("Failed to initialize", e);
        }
    }

    //初始化消息存储服务，创建store路径的文件对于的mmap引用
    result = result && this.messageStore.load();

    if (result) {
        //创建netty服务器
        this.remotingServer = new NettyRemotingServer(this.nettyServerConfig, this.clientHousekeepingService);
        NettyServerConfig fastConfig = (NettyServerConfig) this.nettyServerConfig.clone();
        fastConfig.setListenPort(nettyServerConfig.getListenPort() - 2);
        this.fastRemotingServer = new NettyRemotingServer(fastConfig, this.clientHousekeepingService);

        // 创建 各个apiCode对应的处理器线程池，比如发送消息、回复消息、查询消息、心跳服务等
        // ... 省略一堆线程池创建代码

        //注册对应apiCode对应等处理器
        this.registerProcessor();


        final long initialDelay = UtilAll.computeNextMorningTimeMillis() - System.currentTimeMillis();
        final long period = 1000 * 60 * 60 * 24;
        //启动定时统计服务
        this.scheduledExecutorService.scheduleAtFixedRate(() -> {
            try {
                BrokerController.this.getBrokerStats().record();
            } catch (Throwable e) {
                log.error("schedule record error.", e);
            }
        }, initialDelay, period, TimeUnit.MILLISECONDS);

        //启动定义保存消费进度服务 文件位置:{user.home}/store/config/consumerOffset.json
        this.scheduledExecutorService.scheduleAtFixedRate(() -> {
            try {
                BrokerController.this.consumerOffsetManager.persist();
            } catch (Throwable e) {
                log.error("schedule persist consumerOffset error.", e);
            }
        }, 1000 * 10, this.brokerConfig.getFlushConsumerOffsetInterval(), TimeUnit.MILLISECONDS);

        //启动过滤消息服务 文件位置:{user.home}/store/config/consumerFilter.json
        this.scheduledExecutorService.scheduleAtFixedRate(() -> {
            try {
                BrokerController.this.consumerFilterManager.persist();
            } catch (Throwable e) {
                log.error("schedule persist consumer filter error.", e);
            }
        }, 1000 * 10, 1000 * 10, TimeUnit.MILLISECONDS);

        //SPI方式初始化事务消息服务
        initialTransaction();

        //SPI方式加载验证服务
        initialAcl();
        
        //SPI方式初始化RPC 接口调用钩子
        initialRpcHooks();
    }
    return result;
}
```



### 启动Broker

要启动的服务很多，每个服务的功能集成度也特别高，下面截取一些相对重要的做分析

```java
public void start() throws Exception {

    //消息存储服务启动
    this.messageStore.start();

    //Netty服务启动
    this.remotingServer.start();

    //TLS使用的文件观察服务
    this.fileWatchService.start();

    //broker对外API 启动
    this.brokerOuterAPI.start();

    //长轮询服务启动
    this.pullRequestHoldService.start();

    //心跳连接处理服务
    this.clientHousekeepingService.start();

    //如果未开启DLeger
    if (!messageStoreConfig.isEnableDLegerCommitLog()) {
        //如果是master节点，则启动事务消息检查服务
        startProcessorByHa(messageStoreConfig.getBrokerRole());
        //同步slave服务启动，调用的是SlaveSynchronize.syncAll()
        handleSlaveSynchronize(messageStoreConfig.getBrokerRole());
        //注册broker节点到namesrv上
        this.registerBrokerAll(true, false, true);
    }

    //启动定时注册broker到namesrv服务
    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
        @Override
        public void run() {
            try {
                BrokerController.this.registerBrokerAll(true, false, brokerConfig.isForceRegister());
            } catch (Throwable e) {
                log.error("registerBrokerAll Exception", e);
            }
        }
    }, 1000 * 10, Math.max(10000, Math.min(brokerConfig.getRegisterNameServerPeriod(), 60000)), TimeUnit.MILLISECONDS);

    //broker状态管理器
    if (this.brokerStatsManager != null) {
        this.brokerStatsManager.start();
    }

    //定时清理过期请求服务
    if (this.brokerFastFailure != null) {
        this.brokerFastFailure.start();
    }
}
```

