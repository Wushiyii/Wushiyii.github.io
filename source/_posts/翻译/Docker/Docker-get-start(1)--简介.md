---
title: Docker Get start(1)--简介
date: 2018-11-19 11:00:59
tags: [编程,容器,CI/CD,Linux,Docker]
categories: Docker	
---
Docker的简介与基本命令
<!-- more -->
## Docker Get start(1)--简介

### 关于Docker的简介一些基本概念

#### 简介 

**Docker**是一个为程序员及运维提供开发、部署、运行应用的平台。Docker采用集装箱化应用(containerization)来管理部署在不同linux容器的应用。集装箱化有着许多优点，比如:
 
- **灵活** ：即使是最复杂的应用程序也可以集装箱化；
- **轻量级** ：容器利用并共享主机内核
- **可互换** ：您可以动态地部署更新和升级
- **便携式** ：你可以在本地构建，部署到云端，然后运行到任何地方
- **可伸缩** ：可以增加和自动分发容器副本
- **可堆栈** ：您可以垂直和动态地堆栈服务
#### 镜像与容器
一个容器是通过运行一个镜像进行启动的， 镜像是一个可执行包，包含运行应用程序所需的一切——代码、运行时、库、环境变量和配置文件。即，容器是一个镜像的运行的实例。 你可以使用linux命令看到一个运行时容器的镜像列表，各种查看命令，比如`docker ps`。

#### 容器与虚拟机
容器在 Linux 上运行，并与其他容器共享主机的内核。 容器运行的是一个离散进程，一个进程只占用很少的内存。相比之下，虚拟机(VM)运行一个全面的"客户端"操作系统，通过一个管理程序可以虚拟地访问主机资源。 一般来说，虚拟机(VM)提供了一个比大多数应用程序所需要的更多资源的环境。

[![FpPARP.md.png](https://s1.ax1x.com/2018/11/19/FpPARP.md.png)](https://imgchr.com/i/FpPARP)

#### 准备环境
安装：https://docs.docker.com/engine/installation/

测试：`docker --version`

``` powershell
docker --version
Docker version 18.06.1-ce, build e68fc7a
```
详细信息: `docker info` 或者 `docker version`(没有 -- )
```powershell
docker info

Containers: 1
 Running: 0
 Paused: 0
 Stopped: 1
Images: 11
Server Version: 18.06.1-ce
Storage Driver: overlay2
...
```
#### 测试Docker安装
- 通过运行简单的 Docker 镜像来测试安装是否正常，比如加载hello-world镜像:
```powershell
docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
ca4f61b1923c: Pull complete
Digest: sha256:ca0eeb6fb05351dfc8759c20733c91def84cb8007aa89a5bf606bc8b315b9fc7
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.
...
```
- 列出下载到本机上的镜像:
```powershell
docker image ls
```
- 列出所有容器, -all代表列出所有，不加代表列出运行中的，-aq代表列出quiet mode状态的
```powershell
docker container ls -all
```

> 全文教程翻译自Docker官方文档。    —— [Docker](https://docs.docker.com/get-started/)