---
title: Lamport timestamps and Paxos
date: 2020-07-19 21:10:00
tags: [distribute、Algorithm]
categories: distribute
---
在分布式系统中，不同节点的物理时钟难以同步，无法区分处理事件的顺序。**Leslie Lamport**在1978年提出逻辑时钟的概念来解决分布式系统中区分事件发生的时序问题。

<!-- more -->

###  Lamport 逻辑时钟



//TODO 