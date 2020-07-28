---
title: 分布式一致性理论、CAP、BASE
date: 2020-06-23 16:29:12
tags: [distribute]
categories: distribute
---
分布式一致性理论是由于分布式难以避免的节点故障产生的理论，以此来介绍下CAP与BASE理论

<!-- more -->

### 分布式常见故障

- 网络异常

分布式系统需要在各个节点间通信，无论是http、rpc还是MQ，网络异常的情况无法避免。造成网络异常的原因有很多，比如节点宕机、网络超时等；

- 网络分区（脑裂）

![splits network](http://img.lessisbetter.site/2019-03-Network_Partition_for_Optimization-2.png)

> 图片来自WIKI

WIKI对于网络分区给出的定义是：A **network partition** refers to network decomposition into relatively independent [subnets](https://en.wikipedia.org/wiki/Subnetwork) for their separate optimization as well as network split due to the failure of network devices

节点与节点之间由于各种异常原因造成分区，最终造成整个分布式系统，只有部分节点有效，并且会产生多个小集群。

**分区原因**：

1. 外部：网络交换设备故障等
2. 内部：节点异常等

- 节点故障

在分布式系统中，每个节点都有可能发生故障现象，原因也有很多：宕机、失去响应等

### CAP理论

![cap](https://static.wixstatic.com/media/933668_c8bd69bd93a642db824b4ebaedd45b6f~mv2.png/v1/fill/w_406,h_350,al_c,usm_0.66_1.00_0.01/933668_c8bd69bd93a642db824b4ebaedd45b6f~mv2.png)

CAP即为：`Consistency`、`Availability`、`Partition-tolerance`

- **一致性**（**C**onsistency）：等同于所有节点访问同一份最新的数据副本(`all nodes see the same data at the same time`)
- **可用性**（**A**vailability）：每次请求都能获取到非错的响应——但是不保证获取的数据为最新数据(`Reads and writes always succeed`)
- **分区容错性**（**P**artition tolerance）：以实际效果而言，分区相当于对通信的时限要求。系统如果不能在时限内达成数据一致性，就意味着发生了分区的情况，必须就当前操作在C和A之间做出选择(`the system continues to operate despite arbitrary message loss or failure of part of the system`)

> CAP理论中的CA和数据库事务中ACID的CA并完全是同一回事儿。两者之中的A都是C都是一致性(Consistency)。CAP中的A指的是可用性（Availability），而ACID中的A指的是原子性（Atomicity)

对于本地事务（相对分布式事务），只需要保证ACID模型即可。而在分布式系统中，针对一次业务的事务，如果要求最终数据的一致性，则很有可能需要放弃A（可用性）。

>  CAP三者只能取其二，不能全都保证*

#### CP without A

在这种情况下，分布式系统可以不要求强可用性，即允许系统暂时停机或无响应。这种情况下，就会牺牲用户体验，等待所有数据都一致后才能继续使用系统。

以CP为核心的系统最典型的为一些分布式数据库，比如`Redis`、`HBase`等，在发生异常时，优先保证数据一致性；还有常用的分布式组件`Zookeeper`也是CP的选择优先保证CP的。

#### AP without C

AP即为保证高可用，且允许分区，但是不保证数据一致性。为了保证高可用，用户请求节点的时候需要马上得到返回内容，则每个节点都只能使用本地数据，从而导致全局的数据不一致。

很多系统为了保证服务高可用，而选择放弃数据强一致性，这些系统只需保证最终一致性即可。比如12306、淘宝等系统，在买票的时候显示有票，下单的时候就报错了，这就是暂时的数据不一致。

#### CA without P

CA即为：只保证高可用与强一致性。而分布式系统天生就是需要分区的，所以CA在分布式中几乎不存在。

一些传统RDBMS数据库比如Mysql、Oracle在单机情况就保证了CA，但是在主从之类的集群部署时，也会存在CAP选择的问题。



### BASE理论

> eBay的架构师Dan Pritchett源于对大规模分布式系统的实践总结，在ACM上发表文章提出BASE理论，BASE理论是对CAP理论的延伸，核心思想是即使无法做到强一致性（Strong Consistency，[CAP](http://47.103.216.138/archives/666)的一致性就是强一致性），但应用可以采用适合的方式达到最终一致性（Eventual Consitency）。

BASE并不是B、A、S、E，而是BA（Basic Available 基本可用）、S（Soft State 软状态）、E（Eventually Consistency 最终一致性）三个。

- **基本可用**（Basic Available）：分布式系统出现故障时，允许损失部分可用性，但系统核心是可用的。比如电商系统的下单接口，在峰值请求时可能服务器压力太大，为了服务核心可用，部分用户会被引导到服务降级的页面，避免全部流量打入引起服务雪崩。
- **软状态**（Soft State）：指允许系统中的数据存在中间状态，并且该状态不会影响到系统的整体可用性，即允许分布式系统中各个节点的数据同步存在延时，比如Mysql的主从复制，在生产环境通常会延迟0.1～1s。
- **最终一致性**（Eventually Consistency）：分布式系统中所有节点在经过一段时间后，最终能达到数据的一致性

> **最终一致性**是一种特殊的弱一致性：系统能够保证在没有其他新的更新操作的情况下，数据最终一定能够达到一致的状态，因此所有客户端对系统的数据访问都能够获取到最新的值。**同时，在没有发生故障的前提下，数据达到一致状态的时间延迟，取决于网络延迟、系统负载和数据复制方案设计等因素**。

