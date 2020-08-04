---
title: Paxos and Raft
date: 2020-07-19 21:10:00
tags: [distribute、Algorithm]
categories: distribute
---
Google Chubby的作者Mike Burrows说过， `there is only one consensus protocol, and that’s Paxos” – all other approaches are just broken versions of Paxos.` 意即**世上只有一种一致性算法，那就是Paxos，所有其他一致性算法都是Paxos算法的不完整版。**

<!-- more -->

>  Paxos算法即是用于解决分布式系统中，如何就某个值（也可能是某个决议、某个指令）达成共识的算法。

### 拜占庭将军

拜占庭军队的将军们打算就是否进攻敌军进行决议，但是将军们又分散在不同的阵地，阵地之间距离遥远只能通过通讯兵来传达消息。但是通讯兵中存在叛徒，有着篡改消息的能力，通过消息就能欺骗将军是否进行行动。

Paxos算法的前提是不存在拜占庭将军问题，节点之间通信的信道是安全的，发出的信号不会被篡改，所以Paxos是**基于消息传递**的。

###  Paxos算法

在Paxos算法中，一共有三个角色

- Proposers：提议者
- Acceptors：议员
- Learners：学习者

Proposers提出提案，Acceptor对提案作出决定（accept or not），Leaner负责学习提案结果。**一个节点可能同时作为3个角色**。Proposers 提出提案，提案信息包括提案编号和提议的 value；acceptor 收到提案后可以接受（accept）提案，若提案获得多数派（majority）的 acceptors 的接受，则称该提案被批准（chosen）；learners 只能“学习”被批准的提案

> 1. 决议（value）只有在被 proposers 提出后才能被批准（未经批准的决议称为“提案（proposal）”）；
> 2. 在一次 Paxos 算法的执行实例中，只批准（chosen）一个 value；
> 3. learners 只能获得被批准（chosen）的 value。

#### 算法过程

Paxos算法类似于2PC，也是分为两个阶段

- 第一阶段（prepare）

1. proposer选择一个提案编号M并将prepare请求发送给acceptors中的一个多数派；
2. acceptor收到prepare消息后，如果提案的编号大于它已经回复的所有prepare消息(回复消息表示接受accept)，则acceptor将自己上次接受的提案回复给proposer，并承诺不再回复小于n的提案；

- 第二阶段（accept）

1. 当一个proposer收到了多数acceptors对prepare的回复后，就进入批准阶段。它要向回复prepare请求的acceptors发送accept请求，这个时候如果响应中过半是无任何提案信息的，则代表当前的提案的取值可以是任意值；如果响应中过半是其他的提案信息，那么则从中找到最大编号的提案编号V，组成【M、V】的请求发给acceptors
2. acceptor接收到请求，如果accpetor没有通过比V还大的编号，则会自动同意此提案。如果半数以上都同意，则通过当前提案。



### Raft算法

Raft算法中同样也有三个角色

- Leader：处理所有客户端交互，日志复制等，一般一次只有一个Leader
- Follower：类似选民，完全被动
- Candidate：类似Proposer律师，可以被选为一个新的领导人 

![raft](https://miro.medium.com/max/580/1*TO9R_SS5Tfn07b0sObfhgg.png)



