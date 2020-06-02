---
title: 搭建Jenkins
date: 2020-02-26 22:10:03
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