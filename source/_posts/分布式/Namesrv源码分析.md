---
title: RocketMQ-Namesrv源码分析
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

`NamesrvStartup.main()`方法启动，首先会调用创建Namesrv的主实例NamesrvController，这个类主要作用为保存Namesrv的配置、Netty配置

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

启动NamesrvController,这一阶段主要分为初始化(`initialize()`)与启动(`start()`)

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
执行Broker的start方法，主要是启动Netty服务器

```java

    org.apache.rocketmq.remoting.netty.NettyRemotingServer

    @Override
    public void start() {
        //work线程池
        this.defaultEventExecutorGroup = new DefaultEventExecutorGroup(nettyServerConfig.getServerWorkerThreads());


        ServerBootstrap childHandler =
          //boss、selector组合
            this.serverBootstrap.group(this.eventLoopGroupBoss, this.eventLoopGroupSelector) 
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
                                encoder,//编码
                                new NettyDecoder(),//解码
                                new IdleStateHandler(0, 0, nettyServerConfig.getServerChannelMaxIdleTimeSeconds()),//空闲管理器
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

`DefaultRequestProcessor`处理Broker接收到的请求

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

以下为Namesrv的TCP接口列表

| 枚举 | CODE | DESC |
| :-----| ----: | :----: |
| PUT_KV_CONFIG | 100 | 设置持久化KV值 |
| GET_KV_CONFIG | 101 | 获取KV值 |
| DELETE_KV_CONFIG | 102 | 删除KV值 |
| QUERY_DATA_VERSION | 322 | Broker调用，对比本地与namesrv的数据，判断namesrv是否已更新 |
| REGISTER_BROKER | 103 | `注册Broker` |
| UNREGISTER_BROKER | 104 | `注销broker` |
| GET_ROUTEINFO_BY_TOPIC | 105 | 通过topic查询路由信息 |
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

#### Broker注册至Namesrv

broker的如何启动、运行不在本篇描述，可以查看另一篇`Broker源码分析`，总之在Broker启动后，会解析配置文件中的`NamesrvAddr`,发送103(`REGISTER_BROKER`), 104(`UNREGISTER_BROKER`)来注册或注销`Namesrv`.

##### `REGISTER_BROKER`注册Broker

入口为`registerBrokerWithFilterServer`, 其实这个filterServer不用太过关心，很少用到

```java
public RemotingCommand registerBrokerWithFilterServer(ChannelHandlerContext ctx, RemotingCommand request)
        throws RemotingCommandException {
        //包装请求、返回值
        final RemotingCommand response = RemotingCommand.createResponseCommand(RegisterBrokerResponseHeader.class);
        final RegisterBrokerResponseHeader responseHeader = (RegisterBrokerResponseHeader) response.readCustomHeader();
        final RegisterBrokerRequestHeader requestHeader =
            (RegisterBrokerRequestHeader) request.decodeCommandCustomHeader(RegisterBrokerRequestHeader.class);

        //校验broker传过来的body是否完整
        if (!checksum(ctx, request, requestHeader)) {
            response.setCode(ResponseCode.SYSTEM_ERROR);
            response.setRemark("crc32 not match");
            return response;
        }

        //解析body
        RegisterBrokerBody registerBrokerBody = new RegisterBrokerBody();
        if (request.getBody() != null) {
            registerBrokerBody = RegisterBrokerBody.decode(request.getBody(), requestHeader.isCompressed());
        } else {
            registerBrokerBody.getTopicConfigSerializeWrapper().getDataVersion().setCounter(new AtomicLong(0));
            registerBrokerBody.getTopicConfigSerializeWrapper().getDataVersion().setTimestamp(0);
        }

        //注册Broker节点，详情看后面
        RegisterBrokerResult result = this.namesrvController.getRouteInfoManager().registerBroker(
            requestHeader.getClusterName(),
            requestHeader.getBrokerAddr(),
            requestHeader.getBrokerName(),
            requestHeader.getBrokerId(),
            requestHeader.getHaServerAddr(),
            registerBrokerBody.getTopicConfigSerializeWrapper(),
            registerBrokerBody.getFilterServerList(),
            ctx.channel());

        //返回namesrv节点信息
        responseHeader.setHaServerAddr(result.getHaServerAddr());
        responseHeader.setMasterAddr(result.getMasterAddr());
        byte[] jsonValue = this.namesrvController.getKvConfigManager().getKVListByNamespace(NamesrvUtil.NAMESPACE_ORDER_TOPIC_CONFIG);
        response.setBody(jsonValue);
        response.setCode(ResponseCode.SUCCESS);
        response.setRemark(null);
        return response;
    }
```

详细注册节点实现，主要为更新`RouteInfoManager`的几个节点Map、更新节点信息等，有关filterServer都移除了

```java
public RegisterBrokerResult registerBroker(
            final String clusterName,
            final String brokerAddr,
            final String brokerName,
            final long brokerId,
            final String haServerAddr,
            final TopicConfigSerializeWrapper topicConfigWrapper,
            final List<String> filterServerList,
            final Channel channel) {
        RegisterBrokerResult result = new RegisterBrokerResult();
        try {
            try {
                //RouteInfoManager全局使用的读写锁
                this.lock.writeLock().lockInterruptibly();

                //根据集群名获取对于的broker名集合，并加入到集合
                Set<String> brokerNames = this.clusterAddrTable.get(clusterName);
                brokerNames.add(brokerName);

                //是否首次注册
                boolean registerFirst = false;

                //根据BrokerName获取Broker信息（Broker主节点、Slave节点等）
                BrokerData brokerData = this.brokerAddrTable.get(brokerName);
                if (null == brokerData) {
                    registerFirst = true;
                    brokerData = new BrokerData(clusterName, brokerName, new HashMap<Long, String>());
                    this.brokerAddrTable.put(brokerName, brokerData);
                }
                Map<Long, String> brokerAddrsMap = brokerData.getBrokerAddrs();
                //Map<BrokerId, BrokerAddr>， 如果broker的地址一样，但是brokerId不一样，则直接从map中移除这个节点，后续做处理
                Iterator<Entry<Long, String>> it = brokerAddrsMap.entrySet().iterator();
                while (it.hasNext()) {
                    Entry<Long, String> item = it.next();
                    if (null != brokerAddr && brokerAddr.equals(item.getValue()) && brokerId != item.getKey()) {
                        log.debug("remove entry {} from brokerData", item);
                        it.remove();
                    }
                }

                //直接覆盖brokerId对应的地址
                String oldAddr = brokerData.getBrokerAddrs().put(brokerId, brokerAddr);

                registerFirst = registerFirst || (null == oldAddr);

                //如果为主节点，且broker携带的Topic的配置有变更过，则更新broker的topic配置信息
                if (null != topicConfigWrapper
                        && MixAll.MASTER_ID == brokerId) {
                    if (this.isBrokerTopicConfigChanged(brokerAddr, topicConfigWrapper.getDataVersion())
                            || registerFirst) {
                        if (topicConfigWrapper.getTopicConfigTable() != null) {
                            for (Map.Entry<String, TopicConfig> entry : topicConfigWrapper.getTopicConfigTable().entrySet()) {
                                this.createAndUpdateQueueData(brokerName, entry.getValue());
                            }
                        }
                    }
                }

                //更新节点存活时间戳
                BrokerLiveInfo prevBrokerLiveInfo = this.brokerLiveTable.put(brokerAddr,
                        new BrokerLiveInfo(
                                System.currentTimeMillis(),
                                topicConfigWrapper.getDataVersion(),
                                channel,
                                haServerAddr));

                if (MixAll.MASTER_ID != brokerId) {
                    String masterAddr = brokerData.getBrokerAddrs().get(MixAll.MASTER_ID);
                    if (masterAddr != null) {
                        BrokerLiveInfo brokerLiveInfo = this.brokerLiveTable.get(masterAddr);
                        //更新本地存的master节点地址
                        if (brokerLiveInfo != null) {
                            result.setHaServerAddr(brokerLiveInfo.getHaServerAddr());
                            result.setMasterAddr(masterAddr);
                        }
                    }
                }
            } finally {
                this.lock.writeLock().unlock();
            }
        } catch (Exception e) {
            log.error("registerBroker Exception", e);
        }
        return result;
    }
```

##### `UNREGISTER_BROKER` 注销Broker
```java
public void unregisterBroker(
        final String clusterName,
        final String brokerAddr,
        final String brokerName,
        final long brokerId) {
        try {
            try {
                //全局读写锁
                this.lock.writeLock().lockInterruptibly();
                BrokerLiveInfo brokerLiveInfo = this.brokerLiveTable.remove(brokerAddr);

                boolean removeBrokerName = false;
                BrokerData brokerData = this.brokerAddrTable.get(brokerName);
                if (null != brokerData) {
                    //直接从brokerAddrTable中移除broker信息
                    brokerData.getBrokerAddrs().remove(brokerId);

                    //若移除了当前节点后，所有相同brokerName下的broker都不存在了，则认为从namesrv中移除了该brokerName
                    if (brokerData.getBrokerAddrs().isEmpty()) {
                        this.brokerAddrTable.remove(brokerName);
                        log.info("unregisterBroker, remove name from brokerAddrTable OK, {}", brokerName);

                        removeBrokerName = true;
                    }
                }
                
                // 从clusterAddrTable中移除brokerName
                if (removeBrokerName) {
                    Set<String> nameSet = this.clusterAddrTable.get(clusterName);
                    if (nameSet != null) {
                        boolean removed = nameSet.remove(brokerName);
                        log.info("unregisterBroker, remove name from clusterAddrTable {}, {}", removed ? "OK" : "Failed", brokerName);

                        if (nameSet.isEmpty()) {
                            this.clusterAddrTable.remove(clusterName);
                            log.info("unregisterBroker, remove cluster from clusterAddrTable {}", );
                        }
                    }
                    //移除topic对应的相同brokerName的queue信息
                    this.removeTopicByBrokerName(brokerName);
                }
            } finally {
                this.lock.writeLock().unlock();
            }
        } catch (Exception e) {
            log.error("unregisterBroker Exception", e);
        }
    }
```


##### `GET_ROUTEINFO_BY_TOPIC` 通过topic查询分布信息

```java
public TopicRouteData pickupTopicRouteData(final String topic) {
        TopicRouteData topicRouteData = new TopicRouteData();
        boolean foundQueueData = false;
        boolean foundBrokerData = false;
        Set<String> brokerNameSet = new HashSet<String>();
        List<BrokerData> brokerDataList = new LinkedList<BrokerData>();
        topicRouteData.setBrokerDatas(brokerDataList);

        try {
            try {
                this.lock.readLock().lockInterruptibly();

                // 查找topic对应的queue信息
                //            |  brokerName  |  readQueueNums  |  writeQueueNums  |  perm  |  topicSynFlag |
                // topic =>   |  broker-a    |  8              |  8               |  6     |  0            |
                //            |  broker-b    |  8              |  8               |  6     |  0            |
                //            |  broker-c    |  8              |  8               |  6     |  0            |
                List<QueueData> queueDataList = this.topicQueueTable.get(topic);
                if (queueDataList != null) {
                    topicRouteData.setQueueDatas(queueDataList);
                    foundQueueData = true;

                    Iterator<QueueData> it = queueDataList.iterator();
                    while (it.hasNext()) {
                        QueueData qd = it.next();
                        brokerNameSet.add(qd.getBrokerName());
                    }

                    //每个BrokerName对应一个BrokerData，BrokerData中包含该BrokerName对应的所有brokerId和brokerAddr
                    for (String brokerName : brokerNameSet) {
                        BrokerData brokerData = this.brokerAddrTable.get(brokerName);
                        if (null != brokerData) {
                            BrokerData brokerDataClone = new BrokerData(brokerData.getCluster(), brokerData.getBrokerName(), (HashMap<Long, String>) brokerData.getBrokerAddrs().clone());
                            brokerDataList.add(brokerDataClone);
                            foundBrokerData = true;
                        }
                    }
                }
            } finally {
                this.lock.readLock().unlock();
            }
        } catch (Exception e) {
            log.error("pickupTopicRouteData Exception", e);
        }

        if (foundBrokerData && foundQueueData) {
            return topicRouteData;
        }
        return null;
    }
```
> 为什么要用clone? 看目的为了避免莫名其妙的引用修改，不过直接new一个是不是也没有什么性能损耗呢？


其他的接口大同小异，无非都是根据cluster/topic/broker之类的信息查询，没有很多写操作