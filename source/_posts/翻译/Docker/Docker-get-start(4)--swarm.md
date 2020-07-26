---
title: Docker Get start(4)--Swarm
date: 2018-11-21 15:08:11
tags: [编程,容器,CI/CD,Linux,Docker]
categories: Docker	
---
 Docker get start系列第四节教程
 <!-- more -->
## Docker Get start(4)--Swarm


### 引言
在教程第3部分中，我们将一个应用转换为服务，来定义它在生产环境中应该如何运行，并且将其扩展到5倍实例的状态。
在第4部分中，我们将该应用部署到一个集群上，并在多台计算机上运行它。 多容器、多主机应用将可能通过将多台主机连接到一个称为`Dockerized`集群来实现，而这样的集群也称为`Swarm`集群。

### 理解Swarm集群
一个`Swarm`就是一组服务器运行着`Docker `组成的集群。 在`Swarm`中，你可以和往常一样继续运行以前的 Docker 命令，但是现在它们将由一个被`swarm manage`管理的集群上执行。 `swarm`集群的主机可以是物理的，也可以是虚拟的。 这些主机在加入`swarm`集群后，它们被称为节点(node)。

`swarm manage`可以使用几种策略来运行容器，比如`emtiest node`——用容器填充最少的主机; 或者`global`，它确保每台计算机获得指定容器的一个实例。 你可以设置`swarm manage`在 `Compose` 文件中使用这些策略。

`swarm manage`是集群中唯一能够执行你的命令，或者授权其他主机作为`workers`加入集群。 `workers`在那里只是提供计算，并没有权力告诉任何其他主机能做什么和不能做什么。

到目前为止，您一直在本地计算机上以单主机模式使用 `Docker`。 但是 `Docker `也可以切换到`swarm mode`(集群模式)，并且只有在此模式下才可以使用`swarm`系列的命令。 启用`swarm`集群模式可以使当前的主机成为群管理器。Docker 会在你管理的`swarm`集群上执行命令，而不仅仅是在当前机器上。

### 建立Swarm集群

一个`swarm`集群由多个节点(`node`)组成，这些节点可以是物理主机，也可以是虚拟主机。 基本的概念非常简单: 运行 `docker swarm init`初始化集群，进入集群模式，使你当前的机器成为一个集群管理器，然后在其他主机运行 `docker swarm join`，让它们以`worker`身份加入集群。 我们使用Win10下的 VMs快速创建一个双机集群，并将其转化为一个群。

#### 创建一个集群
##### 本地计算机上的虚拟机(WINDOWS10)
首先，快速创建虚拟机(vm)共享的虚拟交换机，以便它们可以相互连接。
1. 启动 `Hyper-V 管理器`
2. 单击右边菜单中的`虚拟交换机管理器`
3. 单击创建`外部`类型的虚拟网络交换机
4. 设置名称`myswitch`，并选中`外部网络`选择框

现在，使用我们的节点管理工具 `docker-machine` 创建几个虚拟主机:
```bash
docker-machine create -d hyperv --hyperv-virtual-switch "myswitch" myvm1
docker-machine create -d hyperv --hyperv-virtual-switch "myswitch" myvm2
```
如果出现本地无法使用，不能识别版本号之类的，可以使用如下命令:
```bash
docker-machine create --driver hyperv --hyperv-virtual-switch "myswitch" --hyperv-boot2docker-url file://C:/Users/User_name/.docker/machine/machines/dev/boot2docker.iso myvm1
```
其中`boot2docker`的[下载链接](https://github.com/boot2docker/boot2docker/releases/download/v18.09.0/boot2docker.iso).

##### 列出虚拟机并获取它们的 IP 地址
现在已经创建了两个虚拟机，分别命名为`myvm1`和`myvm2`。
使用此命令列出机器并获取它们的 IP 地址。
```
$ docker-machine ls
NAME    ACTIVE   DRIVER       STATE     URL                         SWARM    DOCKER        ERRORS
myvm1   -        virtualbox   Running   tcp://192.168.99.100:2376            v17.06.2-ce
myvm2   -        virtualbox   Running   tcp://192.168.99.101:2376            v17.06.2-ce
```
##### 初始化集群并添加节点
第一台主机扮演管理者的角色，执行管理命令并认证`worker`加入集群，第二台主机是`worker`。
可以使用`docker-machine ssh `将命令发送到你的虚拟机中。 命令`myvm1`成为一个带有 `docker swarm init `的 `swarm manager`，输出如下:
```
$ docker-machine ssh myvm1 "docker swarm init --advertise-addr <myvm1 ip>"
Swarm initialized: current node <node ID> is now a manager.

To add a worker to this swarm, run the following command:

  docker swarm join \
  --token <token> \
  <myvm ip>:<port>

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```
可以看见，`docker swarm init` 的响应包含一个预先配置的 `docker swarm join `命令，您可以在想要添加的任何节点上运行该命令。 复制这个命令，通过 `docker-machine ssh` 发送到 myvm2，让 myvm2作为一个`worker`加入你的 `swarm`集群:
```
$ docker-machine ssh myvm2 "docker swarm join \
--token <token> \
<ip>:2377"

This node joined a swarm as a worker.
```
**注意** ： 此处加入的端口不能与leader端口号相同，否则会报错。

在管理器上运行` docker node ls`来查看这个集群中的节点:
```
$ docker-machine ssh myvm1 "docker node ls"
            ID               HOSTNAME   STATUS  AVAILABILITY   MANAGER STATUS
brtu9urxwfd5j0zrmkubhpkbd     myvm2     Ready      Active
rihwohkh3ph38fhillhhb84sk *   myvm1     Ready      Active         Leader
```
如果你想重新开始，你可以在每个节点运行` docker swarm leave`

### 将应用部署到Swarm集群中
困难的部分已经过去了。 现在只需重复教程第3部分中使用的在新的集群上部署的过程，只要记住，只有像 `myvm1`这样的`swarm manager`群管理器才能执行 `Docker` 命令; `worker`只是为了满足运算能力。

#### 为Swarm manager(集群管理器)配置一个 `docker-machine shell`
到目前为止，我们一直在`docker-machine ssh `中加入 `Docker` 命令来与虚拟机交流。 另一种方式是运行`docker-machine env <machine>`，获取并运行一个命令，该命令将当前 shell 配置为与虚拟机上的` Docker` 守护进程联通。 这种方法在之后的实践效果更好，因为它允许使用本地的`docker-compose. yml `文件通过远程部署应用程序，而无需在任何地方复制它。
输入 `docker-machine env myvm1`，然后终端输出的最后一行复制、粘贴并运行，以配置当前`shell` 与 `myvm1(swarm manager)`联通。
##### Windows10下配置
1. 运行 `docker-machine env myvm1`。
```
PS C:\Users\sam\sandbox\get-started> docker-machine env myvm1
$Env:DOCKER_TLS_VERIFY = "1"
$Env:DOCKER_HOST = "tcp://192.168.203.207:2376"
$Env:DOCKER_CERT_PATH = "C:\Users\sam\.docker\machine\machines\myvm1"
$Env:DOCKER_MACHINE_NAME = "myvm1"
$Env:COMPOSE_CONVERT_WINDOWS_PATHS = "true"
# Run this command to configure your shell:
# & "C:\Program Files\Docker\Docker\Resources\bin\docker-machine.exe" env myvm1 | Invoke-Expression
```
2. 运行以下命令。
```
& "C:\Program Files\Docker\Docker\Resources\bin\docker-machine.exe" env myvm1 | Invoke-Expression
```
3. 运行 `docker-machine ls`，以验证 `myvm1`是激活的主机，如其旁边的星号所示。
```
PS C:PATH> docker-machine ls
NAME  ACTIVE  DRIVER   STATE     URL                        SWARM   DOCKER       ERRORS
myvm1 *       hyperv   Running   tcp://192.168.203.207:2376         v17.06.2-ce
myvm2 -       hyperv   Running   tcp://192.168.200.181:2376         v17.06.2-ce
```

#### 在Swarm manager集群管理器上部署应用
现在有了 `myvm1`虚拟主机，你可以使用swarm manager的权力来部署应用，部署的语法与教程第3部分中一样，也使用`docker stack deploy `命令，与之前不同，在此环境下，可以使用`docker-compose.yml`文件，而不用去拷贝。命令可能几秒钟就能完成，而部署则需要更多的时间来生效。 在一个集群管理器上使用 `docker service ps <service_name>` 命令来验证所有的服务都被重新部署了。
我们可以通过`docker-machine shell` 配置连接到` myvm1`，并且仍然可以访问本地主机上的文件。 请确保当前位于与教程第三部分相同的目录，其中包括`docker-compose.yml` 。
和之前一样，在 `myvm1`上运行以下命令来部署应用。
```
docker stack deploy -c docker-compose.yml hellolab
```

现在我们可以使用在第3部分中使用的相同的 docker 命令。 只有这一次注意到服务(及相关容器)已经在 `myvm1`和 `myvm2`之间进行了分发。
```
docker stack ps hellolab

ID              NAME              IMAGE                 NODE    DESIRED STATE   CURRENT STATE               ERROR         PORTS
n8b0m9hd1c3i    hellolab_web.1    wushiyi/hello:part1   myvm1   Running      Running about a minute ago             
vn3onk3495ql    hellolab_web.2    wushiyi/hello:part1   myvm2   Running       Running about a minute ago             
d6qx2aimpq4d    hellolab_web.3    wushiyi/hello:part1   myvm1   Running       Running about a minute ago             
zmnsrjk2u2qf    hellolab_web.4    wushiyi/hello:part1   myvm1   Running       Running about a minute ago             
8q78o9yenzy3    hellolab_web.5    wushiyi/hello:part1   myvm2   Running       Running about a minute ago 
```

#### 访问集群
可以通过 `myvm1`或 `myvm2`的 IP 地址访问你的应用程序。
我们创建的网络在这两个虚拟机与负载均衡之间共享。 运行` docker-machine ls `获取虚拟主机的 IP 地址，在浏览器上访问其中一个 IP 地址，点击刷新可以看到不同的ID。
![swarm](https://docs.docker.com/get-started/images/app-in-browser-swarm.png)
有五个可能的容器 id 随机循环，说明了负载均衡。
这两个 IP 地址工作的原因是一个群中的节点参与进入路由网格。 这样可以确保部署在您的`swarm`内个端口的服务始终保留该端口，而不管实际运行容器的是在哪个节点。 下面是一个在端口8080上发布的`my-web`服务的路由网状结构:
![three-node](https://docs.docker.com/engine/swarm/images/ingress-routing-mesh.png)

#### 迭代与扩展你的应用
从这里你可以做你在第二部分和第三部分中所学的一切。
- 通过更改`docker-compose.yml`来扩展应用程序;
- 使用不同的应用，然后重新构建，并推送新的镜像。 (要做到这一点，按照之前构建应用程序和发布镜像的相同步骤进行操作)。

在这两种情况下，只需再次运行 `docker stack deploy` 就可以部署这些更改。
可以使用与 `myvm2`上相同的` docker swarm join `命令，将物理的或虚拟的任何主机加入到这个群集中，并将容量添加到你的集群中。 只要运行` docker stack deploy` 就可以了，应用可以利用这些新资源。

#### 清理和重启
##### 堆栈(Stack)与集群(Swarm)
```
docker stack rm hellolab
```
运行`docker-machine ssh myvm2 "docker swarm leave"`使`myvm2`离开集群节点，运行
`docker-machine ssh myvm1 "docker swarm leave --force`使`swarm manager`关闭集群

##### 取消设置`docker-machine shell`
```
& "C:\Program Files\Docker\Docker\Resources\bin\docker-machine.exe" env -u | Invoke-Expression
```
这会断开`Docker`创建的虚拟机与` shell `的连接，并允许你继续在同一个 shell 中工作。
##### 重启或删除 `Docker`上的主机
如果关闭了本地主机，`Docker` 主机将停止运行。 可以通过运行 `docker-machine ls` 来检查机器的状态。 
```
$ docker-machine ls
NAME    ACTIVE   DRIVER       STATE     URL   SWARM   DOCKER    ERRORS
myvm1   -        virtualbox   Stopped                 Unknown
myvm2   -        virtualbox   Stopped                 Unknown
```
要重新启动停止的主机，运行:
```
docker-machine start <machine-name>
```
要删除主机，运行:
```
docker-machine rm <machine-name>
```
要停止主机，运行:
```
docker-machine stop <machine-name>
```