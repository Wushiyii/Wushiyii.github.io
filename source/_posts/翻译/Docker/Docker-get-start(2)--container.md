---
title: Docker Get start(2)---Container
date: 2018-11-19 14:21:57
tags: [编程,容器,CI/CD,Linux,Docker]
categories: Docker	
---
何为容器？
<!-- more -->
## Docker Get start(2)---Container


#### 引言 

是时候开始用**Docker**开发应用了。 首先从一个包含页面的容器底部开始----容器；在容器之上为将是教程第3部分中将要介绍的服务，它定义了在生产环境中容器的行为； 最后，在顶层是堆栈，它定义了教程第5部分中介绍的所有服务的交互。
这就是一个应用的层级结构:
- **Stack** ：堆栈
- **Service** ：服务
- **Container** ：容器

#### 不需要为开发环境烦恼
在过去，如果你要写一个 Python web，得在PC上装好`python`语言包、`flask`、`redis`、`SQL`... 环境又多又不好配置，应用程序想要按照预期运行，还需要与你的生产环境相匹配。
使用**Docker**，你pull到一个可移植的 `Python `镜像，不需要再安装一堆环境。在此Docker环境中构建`python`程序，可以确保`python`程序与环境的依赖关系能够按预期运行。

#### 使用DockerFile 定义一个容器
`Dockerfile` 定义了在容器内部的环境中进行的操作。 对网络和I/O等资源的访问是在这个容器中进行虚拟化的，容器本身与系统是隔离的，所以还需将端口映射到外网，并在`DockerFile`中具体说明希望pull到该容器的文件。 并且，有了` Dockerfile `文件，可以在任何PC上创建同一个环境的容器，而应用程序的在此环境中的运行都完全相同。

##### DockerFile sample
新建一个空文件，重命名为`DockerFile`,不要有后缀名
```dockerfile
# 使用python2.7作为父容器
FROM python:2.7-slim
# 设置工作路径
WORKDIR /app
# 复制当前文档下的文件到/app下
COPY . /app
# 安装requirements.txt中定义的环境
RUN pip install --trusted-host pypi.python.org -r requirements.txt
# 对外部开放80端口
EXPOSE 80
# 定义容器名称
ENV NAME World
# 设置容器启动命令
CMD ["python", "app.py"]
```
再创建两个文件`requirements.txt` 和`app.py`，并将它们与 `Dockerfile` 放在同一个文件夹中。 这就完成了我们的应用程序，正如你看到的，非常简单。
##### requirements.txt
```
Flask
Redis
```
##### app.py

```python
from flask import Flask
from redis import Redis, RedisError
import os
import socket

# Connect to Redis
redis = Redis(host="redis", db=0, socket_connect_timeout=2, socket_timeout=2)

app = Flask(__name__)

@app.route("/")
def hello():
    try:
        visits = redis.incr("counter")
    except RedisError:
        visits = "<i>cannot connect to Redis, counter disabled</i>"

    html = "<h3>Hello {name}!</h3>" \
           "<b>Hostname:</b> {hostname}<br/>" \
           "<b>Visits:</b> {visits}"
    return html.format(name=os.getenv("NAME", "world"), hostname=socket.gethostname(), visits=visits)

if __name__ == "__main__":
    app.run(host='0.0.0.0', port=80)
```
`requirements.txt`为 `Python` 安装 `Flask` 和` Redis` 库，`app.py`打印环境变量文件名以及输出 `socket.gethostname ()`。 最后，由于` Redis `没有运行,在这里使用它的会失败并产生错误消息。
#### 开始构建应用
我们已经准备好开发这个应用程序, 确保当前处位于新目录的顶层。 以下是` ls `应该展示的内容:
```powershell
ls

Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----       2018/11/19     13:42            687 app.py
-a----       2018/11/19     13:40            531 Dockerfile
-a----       2018/11/19     13:42             12 requirements.txt
```
现在运行 build 命令。 这将创建一个 Docker 映像，使用-t 来使用别名,最后有个`.`不要忘记。
```bash
docker build -t hello .
```
创建好的镜像在哪？ 使用`docker image ls `查看
```bash
docker image ls
REPOSITORY   TAG      IMAGE ID      CREATED        SIZE  
hello        latest   d9d1a64ec3ff  20 hours ago   131MB
```
#### 运行应用
运行应用程序，使用`-p `将PC的端口4000映射到容器发布的端口80:
```bash
docker run -p 4000:80 hello

 * Serving Flask app "app" (lazy loading)
 * Environment: production
   WARNING: Do not use the development server in a production environment.
   Use a production WSGI server instead.
 * Debug mode: off
 * Running on http://0.0.0.0:80/ (Press CTRL+C to quit)
```
之后打印了消息显示，映射到容器的地址是` http://0.0.0.0:80`。 但是这条消息来自容器内部，但是容器不知道容器的端口`80`被映射到了`4000`，从而生成了正确的 URL `http://localhost: 4000`。
此时打开`http://localhost: 4000`,页面如下

[![页面](https://docs.docker.com/get-started/images/app-in-browser.png)](https://docs.docker.com/get-started/images/app-in-browser.png)

在` Windows` 系统上，`ctrl + c`只会退出，不会停止容器。 因此，首先键入 `ctrl + c `来获得打开另一个 shell，然后输入 `docker contianer ls `来列出正在运行的容器(更多`docker container ls` 参数[命令](https://www.yiibai.com/docker/container_ls.html))，接着键入` docker container stop <Container NAME or ID> `来停止容器。 否则，尝试重新运行容器时，将从`Deamo`进程中获得一个错误响应。

现在让我们在后台挂起模式下运行应用程序:
```bash
docker run -d -p 4000:80 hello

ce8d40b8aa206b670bf3a1b8109f12db0eb30d49439a231047bd78aad74b7865
```
挂起容器讲话返回一串容器ID
```powershell
docker container ls
CONTAINER     ID     IMAGE    COMMAND  CREATED              STATUS                 
ce8d40b8aa20  hello  "python  app.py"  About a minute ago   Up About a minute   
PORTS                  NAMES
0.0.0.0:4000->80/tcp   ecstatic_swanson

docker stop ce8d
ce8d
```
#### 分享你的镜像
为了证明我们刚刚创建的内容的可移植性，让我们上传我们构建的镜像并在其他地方运行它。

##### 登陆
首先在`hub.docker.com`注册，然后在Terminal输入`docker login`。
##### 标记镜像
在上传镜像前，先要进行一次标记，标记的作用是用来提供版本迭代的机制，你也可以不使用标记。
命令的语法是`docker tag image username/repository:tag`。
sample:
```powershell
docker tag hellogordon/get-started:part2
```
运行 docker 映像来查看你的新标记的图像。
```
REPOSITORY               TAG                 IMAGE ID            CREATED             SIZE
friendlyhello            latest              d9e555c53008        3 minutes ago       195MB
gordon/get-started       part2               d9e555c53008        3 minutes ago       195MB
python                   2.7-slim            1c7128a655f6        5 days ago          183MB
...
```

##### 发布镜像
`docker push username/repository:tag`

一旦完成，这个上传的结果是公开可用的。 如果您登录到 Docker Hub，您会看到那里有一个新的镜像，带有它的 `pull `命令。
##### 从远程镜像仓库中拉取并运行镜像

从现在开始，你可以使用 docker run 在任何机器上运行你的应用程序，命令如下:

`docker run -p 4000:80 username/repository:tag`

如果映像在机器上的本地位置不可用，Docker 会从存储库中提取它。
```powershell
$ docker run -p 4000:80 gordon/get-started:part2
Unable to find image 'gordon/get-started:part2' locally
part2: Pulling from gordon/get-started
10a267c67f42: Already exists
f68a39a6a5e4: Already exists
9beaffc0cf19: Already exists
3c1fe835fb6b: Already exists
4c9f1fa8fcb8: Already exists
ee7d8f576a14: Already exists
fbccdcced46e: Already exists
Digest: sha256:0601c866aab2adcc6498200efd0f754037e909e5fd42069adeff72d1e2439068
Status: Downloaded newer image for gordon/get-started:part2
 * Running on http://0.0.0.0:80/ (Press CTRL+C to quit)
```
无论 `docker run `在哪里执行，它都会拉取你需要的镜像，以及 `Python` 和 `requirements.txt` 中的所有依赖项，并运行您的代码。 它们都在一个精巧的小包中，`Docker `无需在主机上安装任何东西就可以运行。

> 全文教程翻译自Docker官方文档。    —— [Docker](https://docs.docker.com/get-started/part2/)