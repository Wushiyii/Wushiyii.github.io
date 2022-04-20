---
title: Namesrv源码分析
date: 2022-04-20 18:00:43
tags: [distribute, MQ]
categories: distribute
---

Namesrv源码分析

<!-- more -->
### Namesrv 启动流程
首先先从Namesrv的启动开始，这是RocketMQ中类似注册中心的存在，代码也相对简单，集群部署时节点之间无通信调用，一般只和Broker、Producer、Consumer交互，对应的启动入口为:

```java
public static void main(String[] args) {
        main0(args);
    }

    public static NamesrvController main0(String[] args) {
        //...
        NamesrvController controller = createNamesrvController(args);
        start(controller);
    }
```

1. NamesrvStartup.main方法启动，首先会调用创建Namesrv的主实例NamesrvController，这个类主要作用为保存Namesrv的配置、Netty配置
```java
public static NamesrvController createNamesrvController(String[] args) throws IOException, JoranException {
       
        //...
        //创建namesrv配置、netty配置
        final NamesrvConfig namesrvConfig = new NamesrvConfig();
        final NettyServerConfig nettyServerConfig = new NettyServerConfig();
        nettyServerConfig.setListenPort(9876);
        
        //... 省略部分为处理从命令行读取配置等
        final NamesrvController controller = new NamesrvController(namesrvConfig, nettyServerConfig);
        return controller;
    }
```

`NamesrvController`为Namesrv的主要类，里面存放大量功能服务类，这是其中一部分信息
```java
public class NamesrvController {

    //namesrv全局配置，主要为一些path路径
    private final NamesrvConfig namesrvConfig;
    //Netty 服务器配置，主要为端口、selector线程数、worker线程数等
    private final NettyServerConfig nettyServerConfig;
    //单线程定时任务线程池，主要为探活Broker、定时打印KV持久化配置
    private final ScheduledExecutorService scheduledExecutorService = Executors.newSingleThreadScheduledExecutor(new ThreadFactoryImpl(
        "NSScheduledThread"));
    //KV存储服务，内部存放一个table，类型为`HashMap<String/* Namespace */, HashMap<String/* Key */, String/* Value */>> configTable;` 使用`ReentrantReadWriteLock`读写锁和`map`实现
    private final KVConfigManager kvConfigManager;
    //路由信息服务，主要存放topic、broker、存活服务信息等
    private final RouteInfoManager routeInfoManager;
    //Netty服务
    private RemotingServer remotingServer;
    //监听Broker与Namesrv的连接，如果有关闭、异常、空闲，主动把broker从存活table中移除
    private BrokerHousekeepingService brokerHousekeepingService;
    //Netty的worker线程池
    private ExecutorService remotingExecutor;
    //namesrv的主要配置信息
    private Configuration configuration;
    //文件监视器，主要用于观察SSL文件修改，会同步修改netty的tls信息。
    private FileWatchService fileWatchService;
```


2. 启动NamesrvController,这一阶段主要分为初始化(`initialize()`)与启动(`start()`)
```java
public boolean initialize() {

        //载入kv配置
        this.kvConfigManager.load();

        //初始化Netty服务器（主要为配置Epoll/NIO，BOSS线程、Selector线程）
        this.remotingServer = new NettyRemotingServer(this.nettyServerConfig, this.brokerHousekeepingService);

        //初始化Worker线程池
        this.remotingExecutor =
            Executors.newFixedThreadPool(nettyServerConfig.getServerWorkerThreads(), new ThreadFactoryImpl("RemotingExecutorThread_"));

        //注册默认Netty消息处理器，这个是namesrv默认处理请求的地方
        this.registerProcessor();

        //启动一个扫描线程池，扫描超过2分钟无心跳的channel并关闭
        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
            @Override
            public void run() {
                NamesrvController.this.routeInfoManager.scanNotActiveBroker();
            }
        }, 5, 10, TimeUnit.SECONDS);

        //... 定时打印KV配置、处理SSL相关
    }
```
3. 执行Broker的start方法，主要是启动Netty服务器
```java

    org.apache.rocketmq.remoting.netty.NettyRemotingServer

    @Override
    public void start() {
        //work线程池
        this.defaultEventExecutorGroup = new DefaultEventExecutorGroup(nettyServerConfig.getServerWorkerThreads());


        ServerBootstrap childHandler =
            this.serverBootstrap.group(this.eventLoopGroupBoss, this.eventLoopGroupSelector) //boss、selector组合
                .channel(useEpoll() ? EpollServerSocketChannel.class : NioServerSocketChannel.class) //使用epoll或nio
                .option(ChannelOption.SO_BACKLOG, 1024)// backlog的定义是已连接但未进行accept处理的socket队列大小， 如果这个队列满了，将会发送一个ECONNREFUSED错误信息给到客户端，即“Connection refused”。
                .option(ChannelOption.SO_REUSEADDR, true)// 一般来说，一个端口释放后会等待两分钟之后才能再被使用，SO_REUSEADDR是让端口释放后立即就可以被再次使用。
                .option(ChannelOption.SO_KEEPALIVE, false)// 不使用TCP默认的keeplive心跳机制，详情见下方引用
                .childOption(ChannelOption.TCP_NODELAY, true)// 不使用Nagle算法，直接发送
                .childOption(ChannelOption.SO_SNDBUF, nettyServerConfig.getServerSocketSndBufSize())//发送缓冲区大小
                .childOption(ChannelOption.SO_RCVBUF, nettyServerConfig.getServerSocketRcvBufSize())//接收缓冲区大小
                .localAddress(new InetSocketAddress(this.nettyServerConfig.getListenPort()))
                .childHandler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    public void initChannel(SocketChannel ch) throws Exception {
                        ch.pipeline()
                            .addLast(defaultEventExecutorGroup, HANDSHAKE_HANDLER_NAME, handshakeHandler)// 
                            .addLast(defaultEventExecutorGroup,
                                encoder,
                                new NettyDecoder(),
                                new IdleStateHandler(0, 0, nettyServerConfig.getServerChannelMaxIdleTimeSeconds()),
                                connectionManageHandler,
                                serverHandler // 这个为Netty的业务消息处理器，代码太多，主要为调用NamesrvController初始化阶段时，注册的消息处理器(DefaultRequestProcessor)
                            );
                    }
                });
        //... 启动Netty等代码

        //定时扫描请求响应表，处理里面已超时的请求
        this.timer.scheduleAtFixedRate(new TimerTask() {

            @Override
            public void run() {
                try {
                    NettyRemotingServer.this.scanResponseTable();
                } catch (Throwable e) {
                    log.error("scanResponseTable exception", e);
                }
            }
        }, 1000 * 3, 1000);
    }
```
> 提一下，为什么不使用TCP的keepLive？可以参考这篇 https://www.zhihu.com/question/40602902

4. `DefaultRequestProcessor`处理Broker接收到的请求
```java
@Override
    public RemotingCommand processRequest(ChannelHandlerContext ctx,
        RemotingCommand request) throws RemotingCommandException {

        switch (request.getCode()) {
            case RequestCode.PUT_KV_CONFIG:
                return this.putKVConfig(ctx, request);
            case RequestCode.GET_KV_CONFIG:
                return this.getKVConfig(ctx, request);
            case RequestCode.DELETE_KV_CONFIG:
                return this.deleteKVConfig(ctx, request);
            case RequestCode.QUERY_DATA_VERSION:
                return queryBrokerTopicConfig(ctx, request);
            case RequestCode.REGISTER_BROKER: //注册Broker节点
                Version brokerVersion = MQVersion.value2Version(request.getVersion());
                if (brokerVersion.ordinal() >= MQVersion.Version.V3_0_11.ordinal()) {
                    return this.registerBrokerWithFilterServer(ctx, request);
                } else {
                    return this.registerBroker(ctx, request);
                }
            case RequestCode.UNREGISTER_BROKER: //移除broker节点
                return this.unregisterBroker(ctx, request);
            case RequestCode.GET_ROUTEINFO_BY_TOPIC: // 通过topic获取路由信息（topic分部在哪些broker，以及queneID）
                return this.getRouteInfoByTopic(ctx, request);
            case RequestCode.GET_BROKER_CLUSTER_INFO:
                return this.getBrokerClusterInfo(ctx, request);

             // ... 接口很多，我在下方列了一个表格
        }
    }
```

5. Broker的消息处理列表

| 枚举 | CODE | DESC |
| :-----| ----: | :----: |
| PUT_KV_CONFIG | 100 | 设置持久化KV值 |
| GET_KV_CONFIG | 101 | 获取KV值 |
| DELETE_KV_CONFIG | 102 | 删除KV值 |
| QUERY_DATA_VERSION | 322 | Broker调用，对比本地与namesrv的数据，判断namesrv是否已更新 |
| REGISTER_BROKER | 103 | 注册Broker |
| UNREGISTER_BROKER | 104 | 注销broker |
| GET_ROUTEINFO_BY_TOPIC | 105 | 通过topic查询 |
| GET_BROKER_CLUSTER_INFO | 106 | 获取Broker集群信息 |
| WIPE_WRITE_PERM_OF_BROKER | 205 | 清理Broker |
| GET_ALL_TOPIC_LIST_FROM_NAMESERVER | 206 | 获取所有topic列表 |
| DELETE_TOPIC_IN_NAMESRV | 216 | 删除topic |
| GET_KVLIST_BY_NAMESPACE | 219 | 通过namespace查询kv配置 |
| GET_TOPICS_BY_CLUSTER | 224 | 通过集群名字查询topic |
| GET_SYSTEM_TOPIC_LIST_FROM_NS | 304 |获取所有的clustername，所有的brokername，brokername的master brokerAddr  |
| GET_UNIT_TOPIC_LIST | 311 | 获取topic的topicSynFlag属性是`FLAG_UNIT`的所有topicname |
| GET_HAS_UNIT_SUB_TOPIC_LIST | 312 | 获取topic的topicSynFlag属性是`FLAG_UNIT_SUB`的所有topicname |
| UPDATE_NAMESRV_CONFIG | 318 | 更新configuration这个类维护的property, nameServer中，这个类维护的是namesrvConfig，nettyServerConfig这两份配置 |
| GET_NAMESRV_CONFIG | 319 | 查询configuration这个类维护的property |


### Namesrv的主要交互代码分析
