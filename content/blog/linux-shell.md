---
title: shell
tags:
  - shell
categories:
  - linux
date: 2022-12-01 11:01:29 
---



# 环境变量

在 Linux 系统中，环境变量是用来定义系统运行环境的一些参数，比如每个用户不同的家目录（HOME）、邮件存放位置（MAIL）等。

值得一提的是，Linux 系统中环境变量的名称一般都是大写的，这是一种约定俗成的规范。

我们可以使用 env 命令来查看到 Linux 系统中所有的环境变量，执行命令如下：

```shell
[root@localhost ~]# env
ORBIT_SOCKETDIR=/tmp/orbit-root
HOSTNAME=livecd.centos
GIO_LAUNCHED_DESKTOP_FILE_PID=2065
TERM=xterm
SHELL=/bin/bash
......
```

下面是几个常用的环境变量：

| 环境变量名称 | 作用                                   |
| ------------ | -------------------------------------- |
| HOME         | 用户的主目录（也称家目录）             |
| SHELL        | 用户使用的 Shell 解释器名称            |
| PATH         | 定义命令行解释器搜索用户执行命令的路径 |
| EDITOR       | 用户默认的文本解释器                   |
| **RANDOM**   | 生成一个随机数字                       |
| **LANG**     | 系统语言、语系名称                     |
| HISTSIZE     | 输出的历史命令记录条数                 |
| HISTFILESIZE | 保存的历史命令记录条数                 |
| PS1          | Bash解释器的提示符                     |
| MAIL         | 邮件保存路径                           |

Linux 作为一个多用户多任务的操作系统，能够为每个用户提供独立的、合适的工作运行环境，因此，一个相同的环境变量会因为用户身份的不同而具有不同的值。

例如，使用下述命令来查看 HOME 变量在不同用户身份下都有哪些值：

```shell
[root@localhost ~]# echo $HOME
/root
[root@localhost ~]# su - user1 <--切换到 user1 用户身份
[user1@localhost ~]$ echo $HOME
/home/user1
```



其实，环境变量是由固定的变量名与用户或系统设置的变量值两部分组成的，我们完全可以自行创建环境变量来满足工作需求。例如，设置一个名称为 WORKDIR 的环境变量，方便用户更轻松地进入一个层次较深的目录，执行命令如下：

```shell
[root@localhost ~]# mkdir /home/work1
[root@localhost ~]# WORKDIR=/home/work1
[root@localhost ~]# cd $WORKDIR
[root@localhost work1]# pwd
/home/work1
```

但是，这样的环境变量不具有全局性，作用范围也有限，默认情况下不能被其他用户使用。如果工作需要，~~可以使用 export 命令将其提升为全局环境变量~~，这样其他用户就可以使用它了：

```shell
[root@localhost work1]# su user1  <-- 切换到 user1，发现无法使用 WORKDIR 自定义变量
[user1@localhost ~]$ cd $WORKDIR
[user1@localhost ~]$ echo $WORKDIR

[user1@localhost ~]$ exit <--退出user1身份
[root@localhost work1] export WORKDIR
[root@localhost work1] su user1
[user1@localhost ~]$ cd $WORKDIR
[user1@localhost work1]$ pwd
/home/work1
```

