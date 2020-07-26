---
title: Docker Get start(3)--Service
date: 2018-11-20 13:20:12
tags: [编程,容器,CI/CD,Linux,Docker]
categories: Docker	
---
Docker中的服务
<!-- more -->
## Docker Get start(3)--Service


#### 引言
在教程的第3部分中，我们将扩展教程第2部分中的应用程序，并启用负载平衡。 要做到这一点，我们必须在分布式应用程序的层次结构中向上提升一个层级: 服务。
- Stack : 堆栈
- **Service** : 服务（本节内容）
-  Container :容器

#### 关于服务
在分布式应用程序中，应用程序的不同部分被称为"服务"。 例如一个视频共享网站（Bilibili），它可能包括一个用于在数据库中存储用户数据的服务、一个用户上传某个内容后在后台进行视频转码的服务、一个用于前端的服务、CDN加速服务等等。

服务实际上只是"生产中的容器" 服务只运行一个映像，但它规定了映像的运行方式——它应该使用哪些端口，容器应该运行多少个实例，以便服务确定所需的容量，等等。 扩展服务会改变运行该软件的容器实例的数量，从而为流程中的服务分配更多的计算资源。

幸运的是，使用` Docker `平台定义、运行和扩展服务非常简单——只需编写一个` Docker-compose `的yml文件即可。

#### docker-compose.yml 文件
`docker-compose.yml`是一个 `YAML `文件，它定义了 `Docker `容器在生产环境中的各种行为。

##### docker-compose.yml
```yaml
version: "3"
services:
  web:
    # replace username/repo:tag with your name and image details
    image: username/repo:tag
    deploy:
      replicas: 5
      resources:
        limits:
          cpus: "0.1"
          memory: 50M
      restart_policy:
        condition: on-failure
    ports:
      - "4000:80"
    networks:
      - webnet
networks:
  webnet:
```
`docker-compose.yml` 文件告诉 `Docker` 执行以下操作:
 - 从仓库中拉取出我们在教程2中上传的镜像。
 - 以 `web` 服务的形式运行该镜像的5个实例，限制每个实例最多只能使用10% 的 CPU (跨所有核心)和50MB 的 RAM。
 - 如果一个实例宕机，立即重新启动容器。
 - 将主机上的4000端口映射到 web 的80端口。
 - 控制 web 容器通过称为 `webnet` 的负载均衡网络共享80端口。 (在内部，容器自身发布到 web 的80端口，该端口位于一个短暂的端口。)
 - 用默认设置定义`webnet `网络(这是一个负载均衡的重叠网络)。
 
 #### 运行新的负载均衡应用
 在使用`docker stack deploy` 命令之前，我们首先运行:
 
```bash
docker swarm init
```
 
 **注意**: 我们将在第4部分介绍这个命令的含义。 如果不运行`docker swarm init`,会得到一个错误`this node is not a swarm manager`
 
 现在让我们来运行它。 你需要给你的应用一个名字。
```bash
docker stack deploy -c docker-compose.yml hellolab
```
现在，我们的单个服务堆栈在一台主机上，运行镜像的5个实例。 让我们来观察一下：

使用命令`docker service ls`,获取应用中一个服务的服务 ID:
```
docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE                 PORTS
nymusuaqwkgd        hellolab_web        replicated          5/5                 wushiyi/hello:part1   *:4000->80/tcp
```
如果你将服务命名为与本例中所示相同的名称，则该服务的名称为 `hellolab_web`。 还列出了服务 ID，以及实例的数量、映像名和暴露端口。

在服务中运行的单个容器称为`任务(task)`。 任务的 id 从数值上递增，直到您在 `docker-compose` 中定义的副本的数量。 Yml列出你服务的任务:
```
docker service ps hellolab_web 
```
如果只列出系统上的所有容器，也会显示任务，但不会对服务进行过滤:
```
dockser container ls -q
```
此时，可以在GitBash连续运行 curl-4 http: / / localhost: 4000，或者在浏览器中转到该 URL 并刷新几次。
![balanced](https://docs.docker.com/get-started/images/app80-in-browser.png)
无论采用哪种方式，容器 ID 都会发生变化，演示了负载均衡; 对于每个请求，都会以循环方式选择5个任务中的一个进行响应。 容器 id 与前面命令(docker container ls -q)的输出匹配。

#### 扩展应用
您可以通过更改 `docker-compose` 中的 `replicas` 值来扩展应用的实例数量。 保存更改Yml 后，并重新运行命令:
```
docker stack deploy -c docker-compose.yml hellolab
```
`Docker` 会执行`in-place`更新，不需要先删除堆栈或者kill任何容器.
现在，重新运行` docker container ls -q `来查看已部署的实例的重新配置。 如果扩大实例的规模，就会启动更多的任务(task)，从而启动更多的容器。

#### 关闭应用程序和集群
 - 用 docker 栈关闭应用程序: `docker stack rm hellolab`
 - 关闭集群: `docker swarm leave --force`

通过 `Docker` 可以很容易地支持和扩展你的应用, 接下来将学习如何在 `Docker `机器集群上运行这个应用。

> 全文教程翻译自Docker官方文档。    —— [Docker](https://docs.docker.com/get-started/part3/)