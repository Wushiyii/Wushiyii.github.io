---
title: Git基本操作
date: 2019-04-21 14:46:03
tags: [Workflow,git]
categories: Workflow
---
`Git`的一些概念，基础操作等。

<!-- more -->

# Git基本操作

### 什么是版本控制？

你可以把一个版本控制系统（缩写`VCS`）理解为一个“数据库”，在需要的时候，它可以帮你完整地保存一个项目的快照(snapshot)。当你需要查看一个之前的快照（称之为“版本”）时，版本控制系统可以显示出当前版本与上一个版本之间的所有改动的细节。

![vcs](https://www.git-tower.com/learn/content/01-git/01-ebook/cn/01-command-line/02-basics/01-what-is-version-control/what-is-vcs.png)



版本控制与项目的种类，使用的技术和基础框架并无关系：

- 无论是设计开发一个HTML网站或者是一个应用，它的工作原理都是一样的。
- 你可以选择任何你喜欢的工具来工作，它并不关心你用什么样的文本编辑器，绘图程序，文件管理器或其他工具。

因此不要混淆版本控制的备份系统和一般的部署系统。当你开始尝试在你的项目中使用版本控制，并不需要替换和改变开发过程中使用的那些常用工具。

`Git`作为分布式版本控制系统工具，在许多大型项目中被使用（Linux、JQuery、Ruby on Rails）。

### 上手Git

#### 设置环境

初次使用，首先要设置一些最基本的设置，例如你的用户名，你的邮箱地址以及在命令行界面中的一些重要的显示设置。

```bash
$ git config --global user.name "John Doe"
$ git config --global user.email "john@doe.org"
$ git config --global color.ui auto
```

#### 基本工作流程（一次commit）

```bash
$ git clone user@server:git-repo.git  #首先克隆一个仓库
#做出一些改变（增加文件、修改项目文件内容等）
$ git add new-page.html index.html css/* #将改动的文件加入到暂存区
$ git rm error.html #版本控制中移除掉不需要的文件

#提交
$ git commit -m "Implement the new login box" #一次commit的注释的内容要尽可能的详细并且要能回答以下几个问题：为什么要做这次修改？与上一个版本相比你到底改动了什么？

$ git log #以vim方式展现git中的commit历史，按q退出

```

#### 分支

在真实的项目中，所有的开发工作总是同时进行的，并且其每一个主题都处于一个特定的上下文环境下。如果没有分支系统，所有的需求都会在一个master上进行开发，结果可能就像这样：

![no_branch](https://www.git-tower.com/learn/content/01-git/01-ebook/cn/01-command-line/03-branching-merging/01-branching-can-change-your-life/one-context.png)

而在分支系统上工作，工作流会变得简单明了，团队协作分工明确。

![has_branch](https://www.git-tower.com/learn/content/01-git/01-ebook/cn/01-command-line/03-branching-merging/01-branching-can-change-your-life/multiple-contexts.png)

```bash
$ git branch contact-form  #新建一个分支
$ git branch #显示当前项目所有分支
$ git checkout contact-form #切换到目标分支 (也可以在git checkout后面直接加 -b 进行创建与切换)

$ git status #查看当前状态，可以看见当前为 On branch contact-form

# 进行一些操作
$ git add contact.html
$ git commit -m "Add new contact form page"
$ git log #此时查看log，会发现在最顶端有显示这一次修改

$ git checkout master #切换到master
$ git log #在master分支下的log并没有"Add new contact form page"这次commit
```

可以看见，在不同的分支做操作，并不会互相影响，而HEAD指向的是最新的一次commit。此外，你不需要去考虑这些改动最终会到了哪里，整合的目标永远是你的当前的 HEAD 分支，也就是你的工作副本。

![merge](https://www.git-tower.com/learn/content/01-git/01-ebook/cn/01-command-line/03-branching-merging/05-merging/basic-merging.png)

在 Git 中，进行合并是非常简单方便的。它只需要两个步骤：

- （1） 切换到那个需要接收改动的分支上。
- （2） 执行 “git merge” 命令，并且在后面加上那个将要合并进来的分支的名称。

```bash
$ git checkout master #切换到需要接收改动的分支上。
$ git merge contact-form #合并
$ git log #此时log就会显示被合并分支的操作了
```

------

##### 远程仓库

在版本控制系统上，大约90%的操作都是在本地仓库（local repository）中进行的：暂存，提交，查看状态或者历史记录等等。除此之外，如果仅仅只有你一个人在这个项目里工作，你永远没有机会需要设置一个远程仓库（remote repository）。

只有当你需要和你的开发团队**共享数据**时，设置一个远程仓库才有意义。你可以把它想象成一个 “文件管理服务器”，利用这个服务器可以与开发团队的其他成员进行数据交换。
![remote_rep](https://www.git-tower.com/learn/content/01-git/01-ebook/cn/01-command-line/04-remote-repositories/01-introduction/basic-remote-workflow.png)

```bash
$ git remote add crash-course-remote
    https://github.com/gittower/git-crash-course-remote.git #连接一个远程仓库
```

##### 发布一个本地分支

在你明确地决定将一个本地分支发布到远程仓库之前，这些在你本地计算机上创建的分支是不能被其他的团队成员看到的，它只是你的私有分支。这就意味着，你可以保留某些改动仅仅在你私有的本地分支上，而与其他团队成员分享一些其它分支上的改动。

现在让我们来分享 “contact-form” 分支（它直到现在还仅仅是个私有的本地分支）到 “origin” 远程上：

```bash
$ git checkout contact-form
Switched to branch 'contact-form'

$ git push -u origin contact-form
Counting objects: 36, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (31/31), done.
Writing objects: 100% (36/36), 90.67 KiB, done.
Total 36 (delta 12), reused 0 (delta 0)
Unpacking objects: 100% (36/36), done.
To file://Users/tobidobi/Desktop/GitCrashkurs/remote-test.git
 * [new branch]    contact-form -> contact-form
Branch contact-form set up to track remote branch contact-form from origin.
```

这个命令会告诉 Git 来发布你当前本地的 HEAD 分支到 “origin”上，并命名为 “contact-form”（保持相同的分支名在本地和其对应的远程分支是非常有必要的）。
这个 “-u” 参数会自动地在你本地的 “contact-form” 分支和新建的远程分支之间创建一个 “跟踪” 链接。执行 “git branch” 命令来显示分支信息，并且附带上一些特定的参数，在方括号中就会显示出这个建立的跟踪联系：

```bash
$ git branch -vva
* contact-form           56eddd1 [origin/contact-form] Add new contact..
  faq-content            814927a [crash-course-remote/faq-content: ahead
                                    1] Add new question
  master                 2dfe283 Implement the new login box
  remotes/crash-course-remote/faq-content e29fb3f Add FAQ questions
  remotes/crash-course-remote/master      2b504be Change headlines f...
  remotes/origin/contact-form             56eddd1 Add new contact fo...
  remotes/origin/master  56eddd1 Add new contact form page
```

当创建了这个新的远程分支后，发布新的本地提交将会非常简单，执行 “git push” 命令就完成这个操作。

如果某个开发人员拥有对这个远程仓库的操作权限，而且他想在这个你发布的 “contact-form” 上工作，他可以在自己的本地计算机上新建一个本地分支，并跟踪到这个远程分支上。这样他也就同样可以提交自己的改动到 “contact-form” 上了。

##### 删除分支

假设我们在 “contact-form” 分支上的工作已经完成了。并且我们也已经把最终的改动整合到了 “master” 分支。现在我们就不再需要这个分支了。把它删除掉吧：

```bash
$ git branch -d contact-form
```

为了保持一致，我们也有必要删除它所对应的远程分支。附加上一个 “-r” 参数就可以了：

```bash
$ git branch -dr origin/contact-form
```