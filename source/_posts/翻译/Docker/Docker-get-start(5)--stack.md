---
title: Docker Get start(5)--Stack
date: 2018-11-22 14:42:11
tags: [编程,容器,CI/CD,Linux,Docker]
categories: Docker	
---

<!-- more -->

### 引言
在教程第4部分中，我们学习了如何建立一个运行` Docker `的主机集群，并将一个应用部署到它上面，在多台机器上一起运行容器。
在第5部分中，您将到达分布式应用程序层次结构的顶端: `Stack`(堆栈)。 堆栈是一组共享依赖项的相关服务，可以对它们进行编排和扩展。 单个堆栈能够定义和协调整个应用程序的功能。
从第3部分开始，创建 `Compose `文件并使用` docker stack deplou`时，我们已经开始使用栈了。 但这是在单个主机上运行的单个服务堆栈，这通常不会在生产环境中发生。 在这一节，我们可以利用所学到的知识，使多个服务彼此关联，并在多台机器上运行它们。

### 添加新服务并重新部署
将服务添加到` docker-compose.yml` 中很容易。 首先，让我们添加一个免费的可视化`docker`服务，让我们看看集群是如何调度容器的。
1. 打开 `docker-compose.yml `文件， 在编辑器中将其内容替换为以下内容。 确保将 `username / repo: tag` 替换为你在`Docker Hub`上传的镜像。
```
version: "3"
services:
  web:
    # replace username/repo:tag with your name and image details
    image: username/repo:tag
    deploy:
      replicas: 5
      restart_policy:
        condition: on-failure
      resources:
        limits:
          cpus: "0.1"
          memory: 50M
    ports:
      - "80:80"
    networks:
      - webnet
  visualizer:
    image: dockersamples/visualizer:stable
    ports:
      - "8080:8080"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
    deploy:
      placement:
        constraints: [node.role == manager]
    networks:
      - webnet
networks:
  webnet:
```
这里唯一的新知识是对等网络服务，名为`visualizer`。 注意这里有两个新的东西: 一个`volumes`关键字，让` visualizer `访问主机上`Docker` 的Socket文件，和一个`placement`关键字，确保这个服务只在一个群管理器上运行---- 绝不是一个`worker`。
2. 确保当前 shell 被配置为与 myvm1联通(见教程第四部分)
3. 在管理节点上重新运行`docker stack deploy` 命令:
4. 打开可视化工具
我们在 Compose 文件中配置了可视化工具在端口8080上运行。 通过运行`docker-machine ls` 获得某个节点的 IP 地址。 转到8080端口的 IP 地址，你可以看到运行的可视化程序:
![visualize](https://docs.docker.com/get-started/images/get-started-visualizer1.png)

单实例的可视化工具正在管理器上运行，并且 web 的5个实例分布在整个群上。 你可以通过运行 `docker stack ps <stack> `来证实这个可视化:

`Visualizer `是一个独立的服务，可以在任何包含它的应用程序堆栈中运行。 它不依赖于任何其他东西。 现在让我们创建一个具有依赖性的服务: 提供访问者计数器的`Redis`服务。

### 持久化数据
让我们再次进行相同的工作流程，添加一个用于存储应用数据的` Redis `数据库。
1. 保存下方的 `docker-compose.yml` 文件，它添加了一个` Redis` 服务。 
```bash
version: "3"
services:
  web:
    # replace username/repo:tag with your name and image details
    image: username/repo:tag
    deploy:
      replicas: 5
      restart_policy:
        condition: on-failure
      resources:
        limits:
          cpus: "0.1"
          memory: 50M
    ports:
      - "80:80"
    networks:
      - webnet
  visualizer:
    image: dockersamples/visualizer:stable
    ports:
      - "8080:8080"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
    deploy:
      placement:
        constraints: [node.role == manager]
    networks:
      - webnet
  redis:
    image: redis
    ports:
      - "6379:6379"
    volumes:
      - "/home/docker/data:/data"
    deploy:
      placement:
        constraints: [node.role == manager]
    command: redis-server --appendonly yes
    networks:
      - webnet
networks:
  webnet:
```
`Redis `在 `Docker `库中有一个正式的镜像，并且镜像名称很短----`redis`，所以这里没有`username/ repo`标记。 `Redis` 端口，6379，已经被 `Redis` 预先配置为从容器暴露到主机，在我们的 Compose 文件中，我们公开它从主机到外网，所以你可以实际输入任何节点的 IP 到 `Redis` 桌面管理器，并管理这` Redis` 实例。
最重要的是，`redis` 规范中有一些东西可以让数据在这个堆栈的部署之间保持持久性:
- `redis` 总是运行在管理器上，所以总是使用同一个文件系统
- `redis`可以像访问容器一样访问主机文件系统中的任意目录

并且，这将在主机的物理文件系统中为 Redis 数据创建一个"真实源"。 如果没有它，Redis 就会将它的数据存储在容器文件系统的` / data`目录中，如果容器被重新部署，这些数据就会被清除。
2. 创建一个节点管理器的` / data` 目录:
```
docker-machine ssh myvm1 "mkdir ./data"
```
3. 确保当前 `shell `被配置为与 `myvm1`联通
4. 再次运行` docker stack deploy`。
```
$ docker stack deploy -c docker-compose.yml hellolab
```
5. 运行 `docker service ls`来验证这三个服务是否按预期的方式运行。
6. 查看一个节点上的 `web `页面，比如 http: / / 192.168.99.101，然后查看访问者计数器的结果，该计数器现在存储` Redis `上的信息。
![redis](https://docs.docker.com/get-started/images/app-in-browser-redis.png)

此外，检查端口`8080`的可视化工具在任何一个节点的 IP 地址，并注意到 `redis `、`web `和`visualize`服务一起运行。
![three-service](https://docs.docker.com/get-started/images/visualizer-with-redis.png)

> 全文教程翻译自Docker官方文档。    —— [Docker](https://docs.docker.com/get-started/part5/)