---
title: Red Hat 搭建Jenkins
date: 2020-06-02 12:10:03
tags: [Workflow,CD/CI]
categories: Workflow
---
在Red Hat上从0到1搭建Jenkins

<!-- more -->

# Red Hat搭建Jenkins

### 1.安装JDK、Git、Maven

### 2.安装Jenkins

#### 1.添加Jenkins的repo源

```bash
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
```

#### 2.导入密钥

```bash
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
```

#### 3.yum安装Jenkins

```bash
sudo yum install jenkins
```

#### 4.启动Jenkins

```bash
sudo service jenkins start
```

#### 5.初始化Jenkins

1. 打开jenkins页面（默认port为8080)
2. 使用默认密码登录（cat /var/lib/jenkins/secrets/initialAdminPassword）
3. 安装插件（默认安装即可）；安装完成后即可

#### 6.设置Jenkins

1. 在 /系统管理/全局工具配置
   - 配置Maven、Git、JDK
2.  打开 /系统管理/系统配置
   - Maven ops
   - Jenkins启动路径
   - Git Parameter
   - Git plugin配置全局user信息
   - Publish over ssh 配置ssh公钥、服务器地址、用户名等
3. 

#### 7.配置部署Git项目的一些插件

首先先安装几个必须用到的插件

- [Git Parameter Plug-In](https://plugins.jenkins.io/git-parameter)](https://plugins.jenkins.io/workflow-aggregator)
- [Pipeline Maven Integration Plugin](https://plugins.jenkins.io/pipeline-maven)
- [Publish Over SSH](https://plugins.jenkins.io/publish-over-ssh)
- [SSH plugin](https://plugins.jenkins.io/ssh)

#### 8. 配置项目与插件

1. 新建一个maven项目，
2. 在Maven Info Plugin Configuration配置log rotation
3. 添加`git parameter`配置，并增加对应的BRANCH变量名
4. 在源码管理tab，填好对应的git地址
5. 在编译tab中，填好maven编译指令，例如：`mvn clean install -pl xxx -am -Pdev -Dmaven.test.skip=true`
6. 也在同一tab中，在Post Steps中点击add post-build step，选择send file or exec command over SSH，此时会跳出一个SSH面板，在上面填好相应的编译前置、编译后置、编译完成后执行等指令即可
7. 到此结束配置，可以回到Jenkins首页选择项目进行部署了