---
title: Systemd服务管理
tags:
  - linux
categories:
  - 操作系统
  - linux
date: 2022-12-01 11:21:20
---

历史上Linux的启动一直采用init进程，即下面的命令用来启动服务：

```shell
$ sudo /etc/init.d/apache2 start
#或者
$ service apache2 start
```

这种方法有两个缺点:

- 启动时间长。init进程是串行启动，只有前一个进程启动完，才会启动下一个进程。

- 启动[脚本](https://www.linuxcool.com/ "脚本")复杂。init进程只是执行启动[脚本](https://www.linuxcool.com/ "脚本")，不管其他事情。脚本需要自己处理各种情况，这往往使得脚本变得很长。

Systemd就是为了解决这些问题而诞生的。它的设计目标是，为系统的启动和管理提供一套完整的解决方案，根据Linux惯例，字母d是守护进程（daemon）的缩写，Systemd这个名字的含义，就是它要守护整个系统。

## 概述

![img](linux-systemd/XSs1ro2PfC7fRAijeLwTwjJ4X8QTjws87kjr6_rK61M.pngtoken=W.png)

使用了Systemd，就不需要再用init了。Systemd取代了initd，成为系统的第一个进程（PID 等于 1），其他进程都是它的子进程。

Systemd的优点是功能强大，使用方便，缺点是体系庞大，非常复杂。事实上，现在还有很多人反对使用Systemd，理由就是它过于复杂，与操作系统的其他部分强耦合，违反”keep simple, keep stupid”的Unix哲学。

## 管理命令

Systemd并不是一个命令，而是一组命令，涉及到系统管理的方方面面。

* **systemctl 是Systemd的主命令，用于管理系统**

```
#重启系统
$ sudo systemctl reboot

#关闭系统，切断电源
$ sudo systemctl poweroff

#CPU停止工作
$ sudo systemctl halt

#暂停系统
$ sudo systemctl suspend

#让系统进入冬眠状态
$ sudo systemctl hibernate

#让系统进入交互式休眠状态
$ sudo systemctl hybrid-sleep

#启动进入救援状态（单用户状态）
$ sudo systemctl rescue
```

* **systemd-analyze 命令用于查看启动耗时**

```shell
#查看启动耗时
[root@k8s ~]# systemd-analyze
Startup finished in 2.102s (kernel) + 2.416s (initrd) + 24.551s (userspace) = 29.070s
                                                                             
#查看每个服务的启动耗时
[root@k8s ~]# systemd-analyze blame
         11.097s docker.service
          6.188s vdo.service
          6.137s tuned.service
          5.248s kdump.service


#显示瀑布状的启动过程流
[root@k8s ~]# systemd-analyze critical-chain
The time after the unit is active or started is printed after the "@" character.
The time the unit takes to start is printed after the "+" character.

multi-user.target @24.534s
└─docker.service @13.436s +11.097s
  └─network-online.target @13.434s
    └─network.target @13.397s
      └─network.service @12.627s +769ms
        └─NetworkManager-wait-online.service @11.482s +1.143s
          └─NetworkManager.service @8.147s +3.332s
            └─dbus.service @7.517s
              └─basic.target @7.223s
                └─sockets.target @7.222s
                  └─dbus.socket @7.222s
                    └─sysinit.target @7.202s
                      └─systemd-update-utmp.service @7.155s +45ms
                        └─auditd.service @6.569s +584ms
                          └─systemd-tmpfiles-setup.service @6.538s +29ms
                            └─rhel-import-state.service @6.452s +84ms
                              └─local-fs.target @6.450s
                                └─boot.mount @3.685s +815ms
                                  └─local-fs-pre.target @3.684s
                                    └─lvm2-monitor.service @1.416s +2.268s

#显示指定服务的启动流
$ systemd-analyze critical-chain atd.service
```

* **hostnamectl 命令用于查看当前主机的信息**

```
#显示当前主机的信息
[root@k8s ~]# hostnamectl
   Static hostname: k8s.node2
         Icon name: computer-vm
           Chassis: vm
        Machine ID: e52a6046468046e6ae525e5fc2e64dd5
           Boot ID: 43cacabf46904955a4cb2053c6de911f
    Virtualization: vmware
  Operating System: CentOS Linux 7 (Core)
       CPE OS Name: cpe:/o:centos:centos:7
            Kernel: Linux 3.10.0-862.el7.x86_64
      Architecture: x86-64


#设置主机名。
$ sudo hostnamectl set-hostname rhel7
```

* **localectl 命令用于查看本地化设置**

```
#查看本地化设置
[root@k8s ~]# localectl
   System Locale: LANG=zh_CN.UTF-8
       VC Keymap: cn
      X11 Layout: cn

#设置本地化参数。
$ sudo localectl set-locale LANG=en_GB.utf8
$ sudo localectl set-keymap en_GB
```

* timedatectl 命令用于查看当前时区设置

  ```
  #查看当前时区设置
  [root@k8s ~]# timedatectl
        Local time: 六 2022-04-02 16:11:30 CST
    Universal time: 六 2022-04-02 08:11:30 UTC
          RTC time: 六 2022-04-02 08:11:30
         Time zone: Asia/Shanghai (CST, +0800)
       NTP enabled: yes
  NTP synchronized: yes
   RTC in local TZ: no
        DST active: n/a
  
  
  #显示所有可用的时区
  [root@k8s ~]# timedatectl list-timezones  
  America/Indiana/Indianapolis
  America/Indiana/Knox
  America/Indiana/Marengo
  America/Indiana/Petersburg
  America/Indiana/Tell_City
  America/Indiana/Vevay
  America/Indiana/Vincennes
  America/Indiana/Winamac
  America/Inuvik
  America/Iqaluit                                                                                 
  
  #设置当前时区
  $ sudo timedatectl set-timezone America/New_York
  $ sudo timedatectl set-time YYYY-MM-DD
  $ sudo timedatectl set-time HH:MM:SS
  ```

  

* **loginctl 命令用于查看当前登录的用户**

```shell
#列出当前session
[root@k8s ~]# loginctl list-sessions
   SESSION        UID USER             SEAT
         3          0 root
         1          0 root             seat0
         2          0 root

3 sessions listed.

# 列出当前用户
[root@k8s ~]# loginctl list-users
       UID USER
         0 root

1 users listed.

# 查看指定用户的信息
[root@k8s ~]# loginctl show-user root
UID=0
GID=0
Name=root
Timestamp=六 2022-04-02 16:08:01 CST
TimestampMonotonic=68310677
RuntimePath=/run/user/0
Slice=user-0.slice
Display=1
State=active
Sessions=3 2 1
IdleHint=no
IdleSinceHint=0
IdleSinceHintMonotonic=0
Linger=no
```

## Unit

Systemd可以管理所有系统资源，不同的资源统称为 Unit（单位）,Unit一共分成以下12种。

- Service unit：系统服务

- Target unit：多个Unit构成的一个组

- Device Unit：硬件设备

- Mount Unit：文件系统的挂载点

- Automount Unit：自动挂载点

- Path Unit：文件或路径

- Scope Unit：不是由Systemd启动的外部进程

- Slice Unit：进程组

- Snapshot Unit：Systemd快照，可以切回某个快照

- Socket Unit：进程间通信的socket

- Swap Unit：swap文件

- Timer Unit：定时器

### 管理命令

* **查看当前系统的Unit**

```shell
#列出正在运行的Unit
$ systemctl list-units
UNIT                        LOAD   ACTIVE SUB       DESCRIPTION
chronyd.service             loaded active running   NTP client/server
crond.service               loaded active running   Command Scheduler
dbus.service                loaded active running   D-Bus System Message Bus
docker.service              loaded active running   Docker Application Container
getty@tty1.service          loaded active running   Getty on tty1

#列出所有Unit，包括没有找到配置文件的或者启动失败的
$ systemctl list-units --all

#列出所有没有运行的Unit
$ systemctl list-units --all --state=inactive

#列出所有加载失败的Unit
$ systemctl list-units --failed

#列出所有正在运行的、类型为service的Unit
$ systemctl list-units --type=service
```

* **查看unit的状态**

```
#显示系统状态

[root@k8s ~]# systemctl status
● k8s.node2
    State: running
     Jobs: 0 queued
   Failed: 0 units
    Since: 六 2022-04-02 16:06:55 CST; 22min ago
   CGroup: /
           ├─1 /usr/lib/systemd/systemd --switched-root --system --deserialize 2
           ├─user.slice
           │ └─user-0.slice
           │   ├─session-3.scope
           │   │ ├─1886 sshd: root@notty
           │   │ └─1932 /usr/libexec/openssh/sftp-server
           │   ├─session-2.scope
           │   │ ├─ 1876 sshd: root@pts/0
           │   │ ├─ 1888 -bash
           │   │ ├─ 1921 bash -c while true; do sleep 1;head -v -n 8 /proc/memin
           │   │ ├─12099 sleep 1
           │   │ ├─12100 systemctl status
           │   │ └─12101 less
           │   └─session-1.scope
           │     ├─ 709 login -- root
           │     └─1795 -bash
           └─system.slice

#显示单个Unit的状态
$ sysystemctl status bluetooth.service

#显示远程主机的某个Unit的状态
$ systemctl -H root@rhel7.example.com status httpd.service
```

除了status命令，systemctl还提供了三个查询状态的简单方法，主要供脚本内部的判断语句使用:

```
#显示某个Unit是否正在运行
$ systemctl is-active application.service

#显示某个Unit是否处于启动失败状态
$ systemctl is-failed application.service

#显示某个Unit服务是否建立了启动链接
$ systemctl is-enabled application.service
```

* **unit启停命令**

```shell
#立即启动一个服务
$ sudo systemctl start apache.service

#立即停止一个服务
$ sudo systemctl stop apache.service

#重启一个服务
$ sudo systemctl restart apache.service

#杀死一个服务的所有子进程
$ sudo systemctl kill apache.service

#重新加载一个服务的配置文件
$ sudo systemctl reload apache.service

#重载所有修改过的配置文件
$ sudo systemctl daemon-reload

#显示某个 Unit 的所有底层参数
[root@k8s ~]# systemctl show docker
Type=notify
Restart=on-failure
NotifyAccess=main
RestartUSec=100ms
TimeoutStartUSec=0
TimeoutStopUSec=1min 30s
WatchdogUSec=0
WatchdogTimestamp=六 2022-04-02 16:07:22 CST
WatchdogTimestampMonotonic=29053314
StartLimitInterval=60000000
StartLimitBurst=3


#显示某个 Unit 的指定属性的值
[root@k8s ~]# systemctl show -p TasksCurrent docker
TasksCurrent=25

#设置某个 Unit 的指定属性
$ sudo systemctl set-property httpd.service CPUShares=500
```

**依赖关系**

Unit之间存在依赖关系：A依赖于B，就意味着Systemd在启动A的时候，同时会去启动B。

```
[root@k8s ~]# systemctl list-dependencies docker
docker.service
● ├─system.slice
● ├─basic.target
● │ ├─microcode.service
● │ ├─rhel-dmesg.service
● │ ├─selinux-policy-migrate-local-changes@targeted.service
● │ ├─paths.target
● │ ├─slices.target
● │ │ ├─-.slice
● │ │ └─system.slice
● │ ├─sockets.target
● │ │ ├─dbus.socket
● │ │ ├─dm-event.socket
● │ │ ├─rpcbind.socket
● │ │ ├─systemd-initctl.socket
```

### **Unit的配置文件**

每一个Unit都有一个配置文件，告诉Systemd怎么启动这个Unit。Systemd默认从目录/etc/systemd/system/读取配置文件。但是里面存放的大部分文件都是符号链接，指向目录/usr/lib/systemd/system/，真正的配置文件存放在那个目录。

systemctl enable命令用于在上面两个目录之间，建立符号链接关系:

```
$ sudo systemctl enable clamd@scan.service
#等同于
$ sudo ln -s '/usr/lib/systemd/system/clamd@scan.service' '/etc/systemd/system/multi-user.target.wants/clamd@scan.service'
```

如果配置文件里面设置了开机启动，systemctl enable命令相当于激活开机启动，与之对应的，systemctl disable命令用于在两个目录之间，撤销符号链接关系，相当于撤销开机启动。

配置文件的后缀名，就是该Unit的种类，比如sshd.socket；如果省略，Systemd默认后缀名为.service，所以sshd会被理解成sshd.service。

#### 查看配置文件状态

```
[root@k8s system]# systemctl list-unit-files
UNIT FILE                                     STATE
proc-sys-fs-binfmt_misc.automount             static
dev-hugepages.mount                           static
dev-mqueue.mount                              static
proc-sys-fs-binfmt_misc.mount                 static
sys-fs-fuse-connections.mount                 static
sys-kernel-config.mount                       static
sys-kernel-debug.mount                        static
tmp.mount                                     disabled
brandbot.path                                 disabled
systemd-ask-password-console.path             static
systemd-ask-password-plymouth.path            static
systemd-ask-password-wall.path                static
```

每个配置文件的状态，一共有四种：



- enabled：已建立启动链接

- disabled：没建立启动链接

- static：该配置文件没有[Install]部分（无法执行），只能作为其他配置文件的依赖

- masked：该配置文件被禁止建立启动链接

一旦修改配置文件，就要让 SystemD 重新加载配置文件，然后重新启动，否则修改不会生效。

```
$ sudo systemctl daemon-reload
$ sudo systemctl restart httpd.service
```

#### 配置文件的格式

配置文件就是普通的文本文件，可以用文本编辑器打开。

```
# systemctl cat命令可以查看配置文件的内容。
$ systemctl cat atd.service
[Unit]
Description=ATD daemon

[Service]
Type=forking
ExecStart=/usr/bin/atd

[Install]
WantedBy=multi-user.target
```

从上面的输出可以看到，配置文件分成几个区块。每个区块的第一行，是用方括号表示的区别名，比如[Unit]。注意，配置文件的区块名和字段名，都是大小写敏感的。每个区块内部是一些等号连接的键值对。注意，键值对的等号两侧不能有空格。

**[Unit]区块通常是配置文件的第一个区块，用来定义 Unit 的元数据，以及配置与其他 Unit 的关系。它的主要字段如下：**

- Description：简短描述

- Documentation：文档地址

- Requires：当前Unit依赖的其他Unit，如果它们没有运行，当前Unit会启动失败

- Wants：与当前Unit配合的其他Unit，如果它们没有运行，当前Unit不会启动失败

- BindsTo：与Requires类似，它指定的 Unit 如果退出，会导致当前Unit停止运行

- Before：如果该字段指定的Unit也要启动，那么必须在当前Unit之后启动

- After：如果该字段指定的Unit也要启动，那么必须在当前Unit之前启动

- Conflicts：这里指定的Unit 不能与当前Unit同时运行

- Condition...：当前Unit运行必须满足的条件，否则不会运行

- Assert...：当前Unit运行必须满足的条件，否则会报启动失败

**[Install]通常是配置文件的最后一个区块，用来定义如何启动，以及是否开机启动。它的主要字段如下：**

- WantedBy：它的值是一个或多个Target，当前Unit激活时（enable）符号链接会放入/etc/systemd/system目录下面以Target名+.wants后缀构成的子目录中

- RequiredBy：它的值是一个或多个Target，当前Unit激活时，符号链接会放入/etc/systemd/system目录下面以Target 名 + .required后缀构成的子目录中

- Alias：当前Unit 可用于启动的别名

- Also：当前Unit激活（enable）时，会被同时激活的其他Unit

**[Service]区块用来Service的配置，只有Service类型的Unit才有这个区块。它的主要字段如下：**

- Type：定义启动时的进程行为。它有以下几种值。

  - Type=simple：默认值，执行ExecStart指定的命令，启动主进程

  - Type=forking：以fork方式从父进程创建子进程，创建后父进程会立即退出

  - Type=oneshot：一次性进程，Systemd会等当前服务退出，再继续往下执行

  - Type=dbus：当前服务通过D-Bus启动

  - Type=notify：当前服务启动完毕，会通知Systemd，再继续往下执行

  - Type=idle：若有其他任务执行完毕，当前服务才会运行

- ExecStart：启动当前服务的命令

- ExecStartPre：启动当前服务之前执行的命令

- ExecStartPost：启动当前服务之后执行的命令

- ExecReload：重启当前服务时执行的命令

- ExecStop：停止当前服务时执行的命令

- ExecStopPost：停止当其服务之后执行的命令

- RestartSec：自动重启当前服务间隔的秒数

- Restart：定义何种情况Systemd会自动重启当前服务，可能的值包括always（总是重启）、on-success、on-failure、on-abnormal、on-abort、on-watchdog

- TimeoutSec：定义Systemd停止当前服务之前等待的秒数

- Environment：指定环境变量

Unit配置文件的完整字段清单，请参考[官方文档](https://www.freedesktop.org/software/systemd/man/systemd.unit.html)。

### **Target**

启动计算机的时候，需要启动大量的Unit。如果每一次启动，都要一一写明本次启动需要哪些Unit，显然非常不方便。Systemd的解决方案就是Target。

简单说，Target就是一个Unit组，包含许多相关的Unit。启动某个Target的时候，Systemd就会启动里面所有的Unit。从这个意义上说，Target这个概念类似于“状态点”，启动某个Target就好比启动到某种状态。

传统的init启动模式里面，有RunLevel的概念，跟Target的作用很类似。不同的是，RunLevel是互斥的，不可能多个RunLevel同时启动，但是多个Target可以同时启动。

```
#查看当前系统的所有Target
$ systemctl list-unit-files --type=target

#查看一个 Target 包含的所有 Unit
$ systemctl list-dependencies multi-user.target

#查看启动时的默认 Target
$ systemctl get-default

#设置启动时的默认Target
$ sudo systemctl set-default multi-user.target

#切换Target时，默认不关闭前一个Target启动的进程，
# systemctl isolate命令改变这种行为，
#关闭前一个Target里面所有不属于后一个Target的进程
$ sudo systemctl isolate multi-user.target
```

它与init进程的主要差别如下：

- 默认的RunLevel（在/etc/inittab文件设置）现在被默认的Target取代，位置是/etc/systemd/system/default.target，通常符号链接到graphical.target（图形界面）或者multi-user.target（多用户命令行）。

- 启动脚本的位置，以前是/etc/init.d目录，符号链接到不同的RunLevel目录（比如/etc/rc3.d、/etc/rc5.d等），现在则存放在/lib/systemd/system和/etc/systemd/system目录。

- 配置文件的位置，以前init进程的配置文件是/etc/inittab，各种服务的配置文件存放在/etc/sysconfig目录。现在的配置文件主要存放在/lib/systemd目录，在/etc/systemd目录里面的修改可以覆盖原始设置。

## **日志管理**

Systemd统一管理所有Unit的启动日志。带来的好处就是，可以只用journalctl一个命令，查看所有日志（内核日志和应用日志）。日志的配置文件是/etc/systemd/journald.conf。

```shell
#查看所有日志（默认情况下 ，只保存本次启动的日志）
$ sudo journalctl

#查看内核日志（不显示应用日志）
$ sudo journalctl -k

#查看系统本次启动的日志
$ sudo journalctl -b
$ sudo journalctl -b -0

#查看上一次启动的日志（需更改设置）
$ sudo journalctl -b -1

#查看指定时间的日志
$ sudo journalctl --since="2012-10-30 18:17:16"
$ sudo journalctl --since "20 min ago"
$ sudo journalctl --since yesterday
$ sudo journalctl --since "2015-01-10" --until "2015-01-11 03:00"
$ sudo journalctl --since 09:00 --until "1 hour ago"

#显示尾部的最新10行日志
$ sudo journalctl -n

#显示尾部指定行数的日志
$ sudo journalctl -n 20

#实时滚动显示最新日志
$ sudo journalctl -f

#查看指定服务的日志
$ sudo journalctl /usr/lib/systemd/systemd

#查看指定进程的日志
$ sudo journalctl _PID=1

#查看某个路径的脚本的日志
$ sudo journalctl /usr/bin/bash

#查看指定用户的日志
$ sudo journalctl _UID=33 --since today

#查看某个 Unit 的日志
$ sudo journalctl -u nginx.service
$ sudo journalctl -u nginx.service --since today

#实时滚动显示某个 Unit 的最新日志
$ sudo journalctl -u nginx.service -f

#合并显示多个 Unit 的日志
$ journalctl -u nginx.service -u php-fpm.service --since today

#查看指定优先级（及其以上级别）的日志，共有8级
# 0: emerg
# 1: alert
# 2: crit
# 3: err
# 4: warning
# 5: notice
# 6: info
# 7: debug
$ sudo journalctl -p err -b

#日志默认分页输出，--no-pager 改为正常的标准输出
$ sudo journalctl --no-pager

#以 JSON 格式（单行）输出
$ sudo journalctl -b -u nginx.service -o json

#以 JSON 格式（多行）输出，可读性更好
$ sudo journalctl -b -u nginx.serviceqq
 -o json-pretty

#显示日志占据的硬盘空间
$ sudo journalctl --disk-usage

#指定日志文件占据的最大空间
$ sudo journalctl --vacuum-size=1G

#指定日志文件保存多久
$ sudo journalctl --vacuum-time=1years
```

# syslog 和 rsyslog

参考： [Rsyslog日志系统_Syslog (sohu.com)](https://www.sohu.com/a/284486478_639793)

系统日志（Syslog）协议是在一个IP网络中转发系统日志信息的标准，它是在美国加州大学伯克利软件分布研究中心（BSD）的TCP/IP系统实施中开发的，目前已成为工业标准协议，可用它记录设备的日志。Syslog记录着系统中的任何事件，管理者可以通过查看系统记录随时掌握系统状况。系统日志通过Syslog进程记录系统的有关事件，也可以记录应用程序运作事件。通过适当配置，还可以实现运行Syslog协议的机器之间的通信。通过分析这些网络行为日志，可追踪和掌握与设备和网络有关的情况。

在Unix类操作系统上，syslog广泛应用于系统日志。syslog日志消息既可以记录在本地文件中，也可以通过网络发送到接收syslog的服务器。接收syslog的服务器可以对多个设备的syslog消息进行统一的存储，或者解析其中的内容做相应的处理。常见的应用场景是网络管理工具、安全管理系统、日志审计系统。

完整的syslog日志中包含产生日志的程序模块（Facility）、严重性（Severity或 Level）、时间、主机名或IP、进程名、进程ID和正文。在Unix类操作系统上，能够按Facility和Severity的组合来决定什么样的日志消息是否需要记录，记录到什么地方，是否需要发送到一个接收syslog的服务器等。由于syslog简单而灵活的特性，syslog不再仅限于Unix类主机的日志记录，任何需要记录和发送日志的场景，都可能会使用syslog。

rsyslog可以简单的理解为syslog的超集，在老版本的Linux系统中，Red Hat Enterprise Linux 3/4/5默认是使用的syslog作为系统的日志工具，从RHEL 6 开始系统默认使用了rsyslog。

它所做的事就是统一记录系统的各个子系统产生的日志。但是像FTP、HTTP它们都有自己日志记录格式，而不是系统的Rsyslog。在Rsyslog系统有两个进程分别是klogd,syslogd。而为什么需要两个守护进程呢?是因为内核跟其他信息需要记录的详细程度及格式的不同。

- klogd：记录内核信息，系统启动中在登录之前使用的都是物理终端/dev/console，这个时候虚拟终端还没有启动而内核启动日志都存放在/var/log/dmesg文件中，使用dmesg命令可以查看。

- syslogd：记录非内核系统产生的信息，当系统启动/sbin/init程序时产生的日志都存放在以下各个日志文件中。

  - /var/log/message #标准系统错误信息；

  - /var/log/maillog #邮件系统产生的日志信息；

  - /var/log/secure #记录系统的登录情况；

  - /var/log/dmesg #记录linux系统在引导过程中的各种记录信息；

  - /var/log/cron #记录crond计划任务产生的时间信息；

  - /var/log/lastlog #记录每个用户最近的登录事件；

  - /var/log/wtmp #记录每个用户登录、注销及系统启动和停机事件；

  - /var/run/btmp #记录失败的、错误的登录尝试及验证事件；

另外当日志文件过大时会通过系统crontab定义日志滚动的，也称日志切割，由logrotate控制其配置文件/etc/logrotate.conf，然后再由系统任务计划执行。logrotate负责备份和删除旧日志, 以及更新日志文件。后面详说。

## 日志配置

- /etc/rsyslog.conf 是rsyslog服务的总配置文件

- /etc/rsyslog.d该目录是单独配置的rsyslog配置文件

配置文件示例：

```
# 表示将所有facility的info级别,但不包括mail,authpriv,cron相关的信息,记录到 /var/log/messages文件

*.info;mail.none;authpriv.none;cron.none /var/log/messages

# 表示将权限,授权相关的所有基本的信息,记录到/var/log/secure文件中.这个文件的权限是600

authpriv.* /var/log/secure

# 表示将mail相关的所有基本的信息记录到/var/log/maillog文件中,可以看到路径前面有一个"-"而"-" 表示异步写入磁盘

mail.* -/var/log/maillog

# 表示将任务计划相关的所有级别的信息记录到/var/log/cron文件中

cron.* /var/log/cron

# 表示将所有facility的emerg级别的信息,发送给登录到系统上的所有用户

*.emerg *

# 表示将uucp及news的crit级别的信息记录到/var/log/spooler文件中

uucp,news.crit /var/log/spooler

# 表示将local7的所有级别的信息记录到/var/log/boot.log文件中上面说过local0 到local7这8个是用户自定义使用的,这里的local7记录的是系统启动相关的信息

local7.* /var/log/boot.log
```

配置文件 **/etc/rsyslog.conf** 有下面几个核心配置：

- modules，模块，配置加载的模块，如：ModLoad imudp.so配置加载UDP传输模块

- global directives，全局配置，配置ryslog守护进程的全局属性，比如主信息队列大小（MainMessageQueueSize）

- rules，规则（选择器+动作），每个规则行由两部分组成，selector部分和action部分，这两部分由一个或多个空格或tab分隔，selector部分指定源和日志等级，action部分指定对应的操作

- 模板（templates）： 允许你指定日志信息的格式，也可用于生成动态文件名，或在规则中使用

- 输出（outputs）

**syslog协议格式定义如下：**

```
facility.priority action
```

facility标识系统需要记录日志的子系统，大概有以下子系统:

| 日志类型      | 日志内容                                 |
| ------------- | ---------------------------------------- |
| auth          | 用户认证时产生的日志                     |
| authpriv      | ssh、ftp等登录信息的验证信息             |
| daemon        | 一些守护进程产生的日志                   |
| ftp           | FTP产生的日志                            |
| lpr           | 打印相关活动                             |
| mark          | 服务内部的信息，时间标识                 |
| news          | 网络新闻传输协议(nntp)产生的消息。       |
| syslog        | 系统日志                                 |
| security      |                                          |
| uucp          | Unix-to-Unix Copy 两个unix之间的相关通信 |
| console       | 针对系统控制台的消息。                   |
| cron          | 系统执行定时任务产生的日志。             |
| kern          | 系统内核日志                             |
| local0~local7 | 自定义程序使用                           |
| mail          | 邮件日志                                 |
| user          | 用户进程                                 |

priority用来标识日志级别,级别越低信息越详细，有以下日志级别，从上到下，级别从高到低，记录的信息越来越少：

|      | 日志等级     | 说明                                                       |
| ---- | ------------ | ---------------------------------------------------------- |
| 7    | emerg        | 紧急情况，系统不可用（例如系统崩溃），一般会通知所有用户。 |
| 6    | alert        | 需要立即修复的告警。                                       |
| 5    | crit         | 危险情况，例如硬盘错误，可能会阻碍程序的部分功能。         |
| 4    | error/err    | 一般错误消息。                                             |
| 3    | warning/warn | 警告。                                                     |
| 2    | notice       | 不是错误，但是可能需要处理。                               |
| 1    | info         | 通用性消息，一般用来提供有用信息。                         |
| 0    | debug        | 调试程序产生的信息。                                       |
|      | none         | 没有优先级，不记录任何日志消息。                           |
