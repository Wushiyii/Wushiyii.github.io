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

