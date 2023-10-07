---
title: linux软件和服务管理
tags:
  - linux
categories:
  - 操作系统
  - linux
date: 2022-12-01 11:21:20
---

# 进程管理

在操作系统中，所有可以执行的程序与命令都会产生进程。只是有些程序和命令非常简单，如 ls 命令、touch 命令等，它们在执行完后就会结束，相应的进程也就会终结，所以我们很难捕捉到这些进程。但是还有一些程和命令，比如 httpd 进程，启动之后就会一直驻留在系统当中，我们把这样的进程称作常驻内存进程。  

某些进程会产生一些新的进程，我们把这些进程称作子进程，而把这个进程本身称作父进程。比如，我们必须正常登录到 Shell 环境中才能执行系统命令，而 Linux 的标准 Shell 是 bash。我们在 bash 当中执行了 ls 命令，那么 bash 就是父进程，而 ls 命令是在 bash 进程中产生的进程，所以 ls 进程是 bash 进程的子进程。也就是说，子进程是依赖父进程而产生的，如果父进程不存在，那么子进程也不存在了。

## 进程启动的方式

在 Linux 系统中，每个进程都有一个唯一的进程号（PID），方便系统识别和调度进程。通过简单地输出运行程序的程序名，就可以运行该程序，其实也就是启动了一个进程。  

总体来说，启动一个进程主要有 2 种途径，分别是通过手工启动和通过调度启动（事先进行设置，根据用户要求，进程可以自行启动。

### 手工启动进程

手工启动进程指的是由用户输入命令直接启动一个进程，根据所启动的进程类型和性质的不同，其又可以细分为前台启动和后台启动 2 种方式。

#### 前台启动进程

这是手工启动进程最常用的方式，因为当用户输入一个命令并运行，就已经启动了一个进程，而且是一个前台的进程，此时系统其实已经处于一个多进程的状态（一个是 Shell 进程，另一个是新启动的进程）。

假如启动一个比较耗时的进程，然后再把该进程挂起，并使用 ps 命令查看，就会看到该进程在 ps 显示列表中，例如：

```shell
[root@localhost ~]# find / -name demo.jpg <--在根目录下查找 demo.jpg 文件，比较耗时
#此处省略了该命令的部分输出信息
#按“CTRL+Z”组合键，即可将该进程挂起
[root@localhost ~]# ps <--查看正在运行的进程
PID  TTY      TIME   CMD
2573 pts/0  00:00:00 bash
2587 pts/0  00:00:01 find
2588 pts/0  00:00:00 ps
```

> 将进程挂起，指的是将前台运行的进程放到后台，并且暂停其运行.

#### 后台启动进程

进程直接从后台运行，用的相对较少，除非该进程非常耗时，且用户也不急着需要其运行结果的时候，例如，用户需要启动一个需要长时间运行的格式化文本文件的进程，为了不使整个 Shell 在格式化过程中都处于“被占用”状态，从后台启动这个进程是比较明智的选择。  

从后台启动进程，其实就是在命令结尾处添加一个 " &" 符号（注意，& 前面有空格）。输入命令并运行之后，Shell 会提供给我们一个数字，此数字就是该进程的进程号。然后直接就会出现提示符，用户就可以继续完成其他工作，例如：

```shell
[root@localhost ~]# find / -name install.log &
[1] 1920
#[1]是工作号，1920是进程号
```

以上介绍了手工启动的 2 种方式，实际上它们有个共同的特点，就是新进程都是由当前 Shell 这个进程产生的，换句话说，是 Shell 创建了新进程，于是称这种关系为进程间的父子关系，其中 Shell 是父进程，新进程是子进程。  

值得一提的是，一个父进程可以有多个子进程，通常子进程结束后才能继续父进程；当然，如果是从后台启动，父进程就不用等待子进程了。

### Linux调度启动进程

在 Linux 系统中，任务可以被配置在指定的时间、日期或者系统平均负载量低于指定值时自动启动。  

例如，Linux 预配置了重要系统任务的运行，以便可以使系统能够实时被更新，系统管理员也可以使用自动化的任务来定期对重要数据进行备份。  

实现调度启动进程的方法有很多，例如通过 crontab、at 等命令。

# 软件安装

## 安装包

Linux下的软件包可细分为两种，分别是源码包和二进制包。

源码包就是一大堆源代码程序，是由程序员按照特定的格式和语法编写出来的。 我们都知道，计算机只能识别机器语言，也就是二进制语言，所以源码包的安装需要一名“翻译官”将“abcd”翻译成二进制语言，这名“翻译官”通常被称为编译器。

源码包的编译是很费时间的，况且绝多大数用户并不熟悉程序语言，在安装过程中我们只能祈祷程序不要报错，否则初学者很难解决。  

为了解决使用源码包安装方式的这些问题，Linux 软件包的安装出现了使用二进制包的安装方式。

二进制包，也就是源码包经过成功编译之后产生的包。由于二进制包在发布之前就已经完成了编译的工作，因此用户安装软件的速度较快（同 Windows下安装软件速度相当），且安装过程报错几率大大减小。

二进制包是 Linux 下默认的软件安装包，因此二进制包又被称为默认安装软件包。目前主要有以下 2 大主流的二进制包管理系统：

- RPM 包管理系统：功能强大，安装、升级、査询和卸载非常简单方便，因此很多 Linux 发行版都默认使用此机制作为软件安装的管理方式，例如 Fedora、CentOS、SuSE 等。

- DPKG 包管理系统：由 Debian Linux 所开发的包管理机制，通过 DPKG 包，Debian Linux 就可以进行软件包管理，主要应用在 Debian 和 Ubuntu 中。

### 源码包 VS RPM二进制包

源码包一般包含多个文件，为了方便发布，通常会将源码包做打包压缩处理，Linux 中最常用的打包压缩格式为“tar.gz”，因此源码包又被称为 Tarball。

Tarball 是 Linux 系统的一款打包工具，可以对源码包进行打包压缩处理，人们习惯上将最终得到的打包压缩文件称为 Tarball 文件。

源码包需要我们自己去软件官方网站进行下载，包中通常包含以下内容：

- 源代码文件。

- 配置和检测程序（如 configure 或 config 等）。

- 软件安装说明和软件说明（如 INSTALL 或 README）。

总的来说，使用源码包安装软件具有以下几点好处：

- 开源。如果你有足够的能力，则可以修改源代码。

- 可以自由选择所需的功能。

- 因为软件是编译安装的，所以更加适合自己的系统，更加稳定，效率也更高。

- 卸载方便。

但同时，使用源码包安装软件也有几点不足：

- 安装过程步骤较多，尤其是在安装较大的软件集合时（如 LAMP 环境搭建），容易出现拼写错误。

- 编译时间较长，所以安装时间比二进制安装要长。

- 因为软件是编译安装的，所以在安装过程中一旦报错，新手很难解决。

使用 RMP 包安装软件具有以下 2 点好处：

1. 包管理系统简单，只通过几个命令就可以实现包的安装、升级、査询和卸载。

1. 安装速度比源码包安装快得多。

与此同时，使用 RMP 包安装软件有如下不足：

- 经过编译，不能在看到源代码。

- 功能选择不如源码包灵活。

- 依赖性。有时我们会发现，在安装软件包 a 时需要先安装 b 和 c，而在安装 b 时需要先安装 d 和 e。这就需要先安装 d 和 e，再安装 b 和 c，最后才能安装 a。比如，我买了一个漂亮的灯具，打算安装在客厅里，可是在安装灯具之前，客厅需要有顶棚，并且顶棚需要刷好油漆。安装软件和装修及其类似，需要有一定的顺序，但是有时依赖性会非常强。

### rpm包的命名规则

RPM 二进制包命名的一般格式如下：

```
包名-版本号-发布次数-发行商-Linux平台-适合的硬件平台-包扩展名
```

例如，RPM 包的名称是httpd-2.2.15-15.el6.centos.1.i686.rpm，其中：

- httped：软件包名。这里需要注意，httped 是包名，而 httpd-2.2.15-15.el6.centos.1.i686.rpm 通常称为包全名，包名和包全名是不同的，在某些 Linux 命令中，有些命令（如包的安装和升级）使用的是包全名，而有些命令（包的查询和卸载）使用的是包名，一不小心就会弄错。

- 2.2.15：包的版本号，版本号的格式通常为主版本号.次版本号.修正号。

- 15：二进制包发布的次数，表示此 RPM 包是第几次编程生成的。

- el*：软件发行商，el6 表示此包是由 Red Hat 公司发布，适合在 RHEL 6.x (Red Hat Enterprise Unux) 和 CentOS 6.x 上使用。

- centos：表示此包适用于 CentOS 系统。

- i686：表示此包使用的硬件平台，目前的 RPM 包支持的平台如表 1 所示：

| 平台名称 | 适用平台信息                                                 |
| -------- | ------------------------------------------------------------ |
| i386     | 386 以上的计算机都可以安装                                   |
| i586     | 686 以上的计算机都可以安装                                   |
| i686     | 奔腾 II 以上的计算机都可以安装，目前所有的 CPU 是奔腾 II 以上的，所以这个软件版本居多 |
| x86_64   | 64 位 CPU 可以安装                                           |
| noarch   | 没有硬件限制                                                 |

- rpm：RPM 包的扩展名，表明这是编译好的二进制包，可以使用 rpm 命令直接安装。此外，还有以 src.rpm 作为扩展名的 RPM 包，这表明是源代码包，需要安装生成源码，然后对其编译并生成 rpm 格式的包，最后才能使用 rpm 命令进行安装。

## RPM包安装

### RPM包默认安装路径

通常情况下，RPM 包采用系统默认的安装路径，所有安装文件会按照类别分散安装到表 1 所示的目录中。

| 安装路径        | 含 义                      |
| --------------- | -------------------------- |
| /etc/           | 配置文件安装目录           |
| /usr/bin/       | 可执行的命令安装目录       |
| /usr/lib/       | 程序所使用的函数库保存位置 |
| /usr/share/doc/ | 基本的软件使用手册保存位置 |
| /usr/share/man/ | 帮助文件保存位置           |

RPM 包的默认安装路径是可以通过命令查询的。 除此之外，RPM 包也支持手动指定安装路径，但此方式并不推荐。因为一旦手动指定安装路径，所有的安装文件会集中安装到指定位置，且系统中用来查询安装路径的命令也无法使用（需要进行手工配置才能被系统识别），得不偿失。

与 RPM 包不同，源码包的安装通常采用手动指定安装路径（习惯安装到 /usr/local/ 中）的方式。既然安装路径不同，同一 apache 程序的源码包和 RPM 包就可以安装到一台 Linux 服务器上（但同一时间只能开启一个，因为它们需要占用同一个 80 端口）。

### RPM 包的安装

```
[root@localhost ~]# rpm -ivh 包全名
```

- -i：安装（install）;

- -v：显示更详细的信息（verbose）;

- -h：打印 #，显示安装进度（hash）;

例如，使用此命令安装 apache 软件包，如下所示：

```shell
[root@localhost ~]# rpm -ivh \
/mnt/cdrom/Packages/httpd-2.2.15-15.el6.centos.1.i686.rpm
Preparing...
####################
[100%]
1:httpd
####################
[100%]
```

注意，直到出现两个 100% 才是真正的安装成功，第一个 100% 仅表示完成了安装准备工作。

此命令还可以一次性安装多个软件包，仅需将包全名用空格分开即可.

如果还有其他安装要求（比如强制安装某软件而不管它是否有依赖性），可以通过以下选项进行调整：  

- -nodeps：不检测依赖性安装。软件安装时会检测依赖性，确定所需的底层软件是否安装，如果没有安装则会报错。如果不管依赖性，想强制安装，则可以使用这个选项。注意，这样不检测依赖性安装的软件基本上是不能使用的，所以不建议这样做。

- -replacefiles：替换文件安装。如果要安装软件包，但是包中的部分文件已经存在，那么在正常安装时会报"某个文件已经存在"的错误，从而导致软件无法安装。使用这个选项可以忽略这个报错而覆盖安装。

- -replacepkgs：替换软件包安装。如果软件包已经安装，那么此选项可以把软件包重复安装一遍。

- -force：强制安装。不管是否已经安装，都重新安装。也就是 -replacefiles 和 -replacepkgs 的综合。

- -test：测试安装。不会实际安装，只是检测一下依赖性。

- -prefix：指定安装路径。为安装软件指定安装路径，而不使用默认安装路径。

### RPM包的升级

```
[root@localhost ~]# rpm -Uvh 包全名
```

-U（大写）选项的含义是：如果该软件没安装过则直接安装；若安装则升级至最新版本。

```
[root@localhost ~]# rpm -Fvh 包全名
```

-F（大写）选项的含义是：如果该软件没有安装，则不会安装，必须安装有较低版本才能升级。

### RPM包的卸载

RPM 软件包的卸载很简单，使用如下命令即可：

```
[root@localhost ~]# rpm -e 包名
```

-e 选项表示卸载，也就是 erase 的首字母。

RPM 软件包的卸载要考虑包之间的依赖性。例如，我们先安装的 httpd 软件包，后安装 httpd 的功能模块 mod_ssl 包，那么在卸载时，就必须先卸载 mod_ssl，然后卸载 httpd，否则会报错。

如果卸载 RPM 软件不考虑依赖性，执行卸载命令会包依赖性错误，例如：

```shell
[root@localhost ~]# rpm -e httpd
error: Failed dependencies:
httpd-mmn = 20051115 is needed by (installed) mod_wsgi-3.2-1.el6.i686
httpd-mmn = 20051115 is needed by (installed) php-5.3.3-3.el6_2.8.i686
httpd-mmn = 20051115 is needed by (installed) mod_ssl-1:2.2.15-15.el6.
centos.1.i686
httpd-mmn = 20051115 is needed by (installed) mod_perl-2.0.4-10.el6.i686
httpd = 2.2.15-15.el6.centos.1 is needed by (installed) httpd-manual-2.2.
15-15.el6.centos.1 .noarch
httpd is needed by (installed) webalizer-2.21_02-3.3.el6.i686
httpd is needed by (installed) mod_ssl-1:2.2.15-15.el6.centos.1.i686
httpd=0:2.2.15-15.el6.centos.1 is needed by(installed)mod_ssl-1:2.2.15-15.el6.centos.1.i686
```

### 查询软件包是否安装

```
[root@localhost ~]# rpm -q 包名
```

-q 表示查询，是 query 的首字母。

注意这里使用的是包名，而不是包全名。因为已安装的软件包只需给出包名，系统就可以成功识别（使用包全名反而无法识别）。

### 查询系统中所有安装的软件包

```
[root@localhost ~]# rpm -qa
libsamplerate-0.1.7-2.1.el6.i686
startup-notification-0.10-2.1.el6.i686
gnome-themes-2.28.1-6.el6.noarch
fontpackages-filesystem-1.41-1.1.el6.noarch
gdm-libs-2.30.4-33.el6_2.i686
gstreamer-0.10.29-1.el6.i686
redhat-lsb-graphics-4.0-3.el6.centos.i686
…省略部分输出…
```

### 查询软件包的详细信息

```shell
[root@localhost ~]# rpm -qi httpd
Name : httpd Relocations:(not relocatable)
#包名
Version : 2.2.15 Vendor:CentOS
#版本和厂商
Release : 15.el6.centos.1 Build Date: 2012年02月14日星期二 06时27分1秒
#发行版本和建立时间
Install Date: 2013年01月07日星期一19时22分43秒
Build Host:
c6b18n2.bsys.dev.centos.org
#安装时间
Group : System Environment/Daemons Source RPM:
httpd-2.2.15-15.el6.centos.1.src.rpm
#组和源RPM包文件名
Size : 2896132 License: ASL 2.0
#软件包大小和许可协议
Signature :RSA/SHA1,2012年02月14日星期二 19时11分00秒，Key ID
0946fca2c105b9de
#数字签名
Packager：CentOS BuildSystem <http://bugs.centos.org>
URL : http://httpd.apache.org/
#厂商网址
Summary : Apache HTTP Server
#软件包说明
Description:
The Apache HTTP Server is a powerful, efficient, and extensible web server.
#描述
```

-i 选项表示查询软件信息，是 information 的首字母。

除此之外，还可以查询未安装软件包的详细信息，命令格式为：

```
[root@localhost ~]# rpm -qip 包全名
```

-p 选项表示查询未安装的软件包，是 package 的首字母。  

注意，这里用的是包全名，且未安装的软件包需使用“绝对路径+包全名”的方式才能确定包。

### 查询软件包的文件列表

通过前面的学习我们知道，rpm 软件包通常采用默认路径安装，各安装文件会分门别类安放在适当的目录文件下。使用 rpm 命令可以查询到已安装软件包中包含的所有文件及各自安装路径，命令格式为：

```shell
[root@localhost ~]# rpm -ql 包名
```

-l 选项表示列出软件包所有文件的安装目录。

例如，查看 apache 软件包中所有文件以及各自的安装位置，可使用如下命令：

```
[root@localhost ~]# rpm -ql httpd
/etc/httpd
/etc/httpd/conf
/etc/httpd/conf.d
/etc/httpd/conf.d/README
/etc/httpd/conf.d/welcome.conf
/etc/httpd/conf/httpd.conf
/etc/httpd/conf/magic
…省略部分输出…
```

同时，rpm 命令还可以查询未安装软件包中包含的所有文件以及打算安装的路径，命令格式如下：

```
[root@localhost ~]# rpm -qlp 包全名
```

-p 选项表示查询未安装的软件包信息，是 package 的首字母。

注意，由于软件包还未安装，因此需要使用“绝对路径+包全名”的方式才能确定包。

### 查询系统文件属于哪个RPM包

rpm -ql 命令是通过软件包查询所含文件的安装路径，rpm 还支持反向查询，即查询某系统文件所属哪个 RPM 软件包。其命令格式如下：

```
[root@localhost ~]# rpm -qf 系统文件名
```

-f 选项的含义是查询系统文件所属哪个软件包，是 file 的首字母。

注意，只有使用 RPM 包安装的文件才能使用该命令，手动方式建立的文件无法使用此命令。

### 查询软件包的依赖关系

使用 rpm 命令安装 RPM 包，需考虑与其他 RPM 包的依赖关系。rpm -qR 命令就用来查询某已安装软件包依赖的其他包，该命令的格式为：

```
[root@localhost ~]# rpm -qR 包名
```

-R（大写）选项的含义是查询软件包的依赖性，是 requires 的首字母。

### 提取RPM包文件

在服务器使用过程，如果系统文件被误修改或误删除，可以考虑使用 cpio 命令提取出原 RPM 包中所需的系统文件，从而修复被误操作的源文件。

RPM 包允许逐个提取包中文件，使用的命令格式如下：

```shell
[root@localhost ~] rpm2cpio 包全名|cpio -idv .文件绝对路径
```

该命令中，rpm2cpio 就是将 RPM 包转换为 cpio 格式的命令，通过 cpio 命令即可从 cpio 文件库中提取出指定文件。

举个例子，假设我们不小心把 /bin/ls 命令删除了，通常有以下 2 种方式修复：

1. 将 `coreutils-8.4-19.el6.i686` 包（包含 ls 命令的 RPM 包）通过 -force 选项再安装一遍；
2. 使用 cpio 命令从 `coreutils-8.4-19.el6.i686` 包中提取出 /bin/ls 文件，然后将其复制到相应位置；

```shell
[root@localhost ~]# mv /bin/ls /root/
#把/bin/ls命令移动到/root/目录下，造成误删除的假象
[root@localhost ~]# ls
-bash: ls: command not found
#这时执行ls命令，系统会报"命令没有找到"错误
[root@localhost ~]# rpm2cpio /mnt/cdrom/Packages/coreutils-8.4-19.el6.i686.rpm
|cpio -idv ./bin/ls
#提取ls命令文件到当前目录下
[root@localhost ~]# cp /root/bin/ls /bin/
#把提取出来的ls命令文件复制到/bin/目录下
[root@localhost ~]#ls
anaconda-ks.cfg bin inittab install.log install.log.syslog ls
#可以看到，ls命令又可以正常使用了
```

#### cpio命令

cpio 命令用于从归档包中存入和读取文件，换句话说，cpio 命令可以从归档包中提取文件（或目录），也可以将文件（或目录）复制到归档包中。  

归档包，也可称为文件库，其实就是 cpio 或 tar 格式的文件，该文件中包含其他文件以及一些相关信息（文件名、访问权限等）。归档包既可以是磁盘中的文件，也可以是磁带或管道。

cpio 命令可以看做是备份或还原命令，因为它可以将数据（文件）备份到 cpio 归档库，也可以利用 cpio 文档库对数据进行恢复。

使用 cpio 命令备份或恢复数据，需注意以下几点：

- 使用 cpio 备份数据时如果使用的是绝对路径，那么还原数据时会自动恢复到绝对路径下；同理，如果备份数据使用的是相对路径，那么数据会还原到相对路径下。

- cpio 命令无法自行指定备份（或还原）的文件，需要目标文件（或目录）的完整路径才能成功读取，因此此命令常与 find 命令配合使用。

- cpio 命令恢复数据时不会自动覆盖同名文件，也不会创建目录（直接解压到当前文件夹）。

cpio 命令主要有以下 3 种基本模式：  

1. "-o" 模式：指的是 copy-out 模式，就是把数据备份到文件库中，命令格式如下：

```
[root@localhost ~]# cpio -o[vcB] > [文件丨设备]
```

各选项含义如下：

- -o：copy-out模式，备份；

- -v：显示备份过程；

- -c：使用较新的portable format存储方式；

- -B：设定输入/输出块为 5120Bytes，而不是模式的 512Bytes；

比如，使用 cpio 备份数据的命令如下：

```
[root@localhost ~]#find /etc -print | cpio -ocvB > /root/etc.cpio
#利用find命令指定要备份/etc/目录，使用>导出到etc.cpio文件
[root@localhost ~]# ll -h etc.cpio
-rw--r--r--.1 root root 21M 6月5 12:29 etc.cpio
#etc.cpio文件生成
```

2. "-i" 模式：指的是 copy-in 模式，就是把数据从文件库中恢复，命令格式如下：

```
[root@localhost ~]# cpio -i[vcdu] < [文件|设备]
```

各选项的含义为：

- -i：copy-in 模式，还原；

- -v：显示还原过程；

- -c：较新的 portable format 存储方式；

- -d：还原时自动新建目录；

- -u：自动使用较新的文件覆盖较旧的文件；

比如，使用 cpio 恢复之前备份的数据，命令如下：

```shell
[root@localhost ~]# cpio -idvcu < /root/etc.cpio
#还原etc的备份
#如果大家査看一下当前目录/root/，就会发现没有生成/etc/目录。这是因为备份时/etc/目录使用的是绝对路径，所以数据直接恢复到/etc/系统目录中，而没有生成在/root/etc/目录中
```

3. "-p" 模式：指的是复制模式，使用 -p 模式可以从某个目录读取所有文件，但并不将其备份到 cpio 库中，而是直接复制为其他文件。  

例如，使用 -p 将 /boot/ 复制到 /test/boot 目录中可以执行如下命令：

```shell
[root@localhost ~]# cd /tmp/
#进入/tmp/目录
[root@localhost tmp]#rm -rf*
#删除/tmp/目录中的所有数据
[root@localhost tmp]# mkdir test
#建立备份目录
[root@localhost tmp]# find /boot/ -print | cpio -p /tmp/test
#备份/boot/目录到/tmp/test/目录中
[root@localhost tmp]# ls test/boot
#在/tmp/test/目录中备份出了/boot/目录
```

## Linux重建RPM数据库

我们知道，RPM 包是很多 Linux 发行版（Fefora、RedHat、SuSE 等）采用的软件包管理方式，安装到系统中的各 RPM 包，其必要信息都会保存到 RPM 数据库中，以便用户使用 rpm 命令对软件包执行查询、安装和卸载等操作。  

但并非所有的用户操作都“按常理出牌”，例如 RPM 包在升级过程被强行退出、RPM 包安装意外中断等误操作，都可能使 RPM 数据库出现故障，后果是当安装、删除、査询软件包时，请求无法执行，如图 1 所示：

![image-20221201191608562](linux-software/image-20221201191608562.png)

这时就需要重建 RPM 数据库，执行如下 2 步操作：  

1. 删除当前系统中已损坏的RPM数据库，执行如下命令：

   ```
   [root@localhost ~]# rm -f /var/lib/rpm/_db.*
   ```

2. 重建 RPM 数据库，执行如下命令：

```
[root@localhost -]# rpm -rebuilddb
```

这一步需花费一定时间才能完成。

除了用户误操作导致 RPM 数据库崩溃，有些黑客入侵系统后，为避免系统管理员通过 RPM 包校验功能检测出问题，会更改 RPM 数据库。

## YUM安装

RPM 软件包（包含 SRPM 包）的依赖性主要体现在 RPM 包安装与卸载的过程中。  

例如，如果采用最基础的方式（基础服务器方式）安装 Linux 系统，则 gcc 这个软件是没有安装的，需要自己手工安装。当你使用 rpm 命令安装 gcc 软件的 RPM 包，就会发生依赖性错误，错误提示信息如下所示：

```shell
[root@localhost ~]# rpm -ivh /mnt/cdrom/Packages/ gcc-4.4.6-4.el6.i686.rpm
error: Failed dependencies: <―依赖性错误
cloog-ppi >= 0.15 is needed by gcc-4.4.6-4.el6.i686
cpp = 4.4.6-4.el6 is needed by gcc-4.4.6-4.el6.i686
glibc-devel >= 2.2.90-12 is needed by gcc-4.4.6-4.el6.i686
```

除此之外，报错信息中还会明确给出各个依赖软件的版本要求：

- ">="：表示版本要大于或等于所显示版本；

- "<="：表示版本要小于或等于所显示版本；

- "="：表示版本要等于所显示版本；

Linux 系统中，RPM 包之间的依赖关系大致可分为以下 3 种：

1. 树形依赖（A-B-C-D）：要想安装软件 A，必须先安装 B，而安装 B 需要先安装 C…….解决此类型依赖的方法是从后往前安装，即先安装 D，再安装 C，然后安装 B，最后安装软件 A。
2. 环形依赖（A-B-C-D-A）：各个软件安装的依赖关系构成“环状”。解决此类型依赖的方法是用一条命令同时安装所有软件包，即使用 rpm -ivh 软件包A 软件包B ...。
3. 模块依赖：软件包的安装需要借助其他软件包的某些文件（比如库文件），解决模块依赖最直接的方式是通过 [http://www.rpmfind.net](http://www.rpmfind.net/) 网站找到包含此文件的软件包，安装即可。

以上 3 种 RPM 包的依赖关系，给出的解决方案都是手动安装，比较麻烦。yum安装给出了解决方案。

yum，全称"Yellow dog Updater,Modified"，CentOS 系统上的软件包管理器，它能够自动下载 RPM 包并安装，更重要的是，它可以自动处理软件包之间的依赖性关系，一次性安装所有依赖的软件包，无需一个个安装。

yum 在服务器端存有所有的 RPM 包，并将各个包之间的依赖关系记录在文件中，当管理员使用 yum 安装 RPM 包时，yum 会先从服务器端下载包的依赖性文件，通过分析此文件从服务器端一次性下载所有相关的 RPM 包并进行安装。

yum 软件可以用 rpm 命令安装，安装之前可以通过如下命令查看 yum 是否已安装：

```
[root@localhost ~]# rpm -qa | grep yum
yum-metadata-parser-1.1.2-16.el6.i686
yum-3.2.29-30.el6.centos.noarch
yum-utils-1.1.30-14.el6.noarch
yum-plugin-fastestmirror-1.1.30-14.el6.noarch
yum-plugin-security-1.1.30-14.el6.noarch
```

### yum 源

使用 yum 安装软件包之前，需指定好 yum 下载 RPM 包的位置，此位置称为 yum 源。换句话说，yum 源指的就是软件安装包的来源。  

使用 yum 安装软件时至少需要一个 yum 源。yum 源既可以使用网络 yum 源，也可以将本地光盘作为 yum 源。接下来就给大家介绍这两种 yum 源的搭建方式。

#### 网络 yum 源搭建

一般情况下，只要你的主机网络正常，可以直接使用网络 yum 源，不需要对配置文件做任何修改。

网络 yum 源配置文件位于 /etc/yum.repos.d/ 目录下，文件扩展名为"*.repo"（只要扩展名为 "*.repo" 的文件都是 yum 源的配置文件）。

```shell
[root@localhost ~]# ls /etc/yum.repos.d/
CentOS-Base.repo
CentOS-Media.repo
CentOS-Debuginfo.repo.bak
CentOS-Vault.repo
```

可以看到，该目录下有 4 个 yum 配置文件，通常情况下 CentOS-Base.repo 文件生效。我们可以尝试打开此文件，命令如下：

```shell
[root@localhost yum.repos.d]# vim /etc/yum.repos.d/ CentOS-Base.repo
[base]
name=CentOS-$releasever - Base
mirrorlist=http://mirrorlist.centos.org/? release= $releasever&arch=$basearch&repo=os
baseurl=http://mirror.centos.org/centos/$releasever/os/$basearch/
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-6
…省略部分输出…
```

此文件中含有 5 个 yum 源容器，这里只列出了 base 容器，其他容器和 base 容器类似。base 容器中各参数的含义分别为：

- [base]：容器名称，一定要放在[]中。

- name：容器说明，可以自己随便写。

- mirrorlist：镜像站点，这个可以注释掉。

- baseurl：我们的 yum 源服务器的地址。默认是 CentOS 官方的 yum 源服务器，是可以使用的。如果你觉得慢，则可以改成你喜欢的 yum 源地址。

- enabled：此容器是否生效，如果不写或写成 enabled 则表示此容器生效，写成 enable=0 则表示此容器不生效。

- gpgcheck：如果为 1 则表示 RPM 的数字证书生效；如果为 0 则表示 RPM 的数字证书不生效。

- gpgkey：数字证书的公钥文件保存位置。不用修改。

#### 本地 yum 源

在无法联网的情况下，yum 可以考虑用本地光盘（或安装映像文件）作为 yum 源。  

Linux 系统安装映像文件中就含有常用的 RPM 包，我们可以使用压缩文件打开映像文件（iso文件），进入其 Packages 子目录，如图 1 所示

![image-20221201192806805](linux-software/image-20221201192806805.png)

可以看到，该子目录下含有几乎所有常用的 RPM 包，因此使用系统安装映像作为本地 yum 源没有任何问题。  

在 /etc/yum.repos.d/ 目录下有一个 CentOS-Media.repo 文件，此文件就是以本地光盘作为 yum 源的模板文件，只需进行简单的修改即可，步骤如下：

1. 放入 CentOS 安装光盘，并挂载光盘到指定位置。命令如下：

   ```shell
   [root@localhost ~]# mkdir /mnt/cdrom
   #创建cdrom目录，作为光盘的挂载点
   [root@localhost ~]# mount /dev/cdrom /mnt/cdrom/
   mount: block device/dev/srO is write-protected, mounting read-only
   #挂载光盘到/mnt/cdrom目录下
   ```

2. 修改其他几个 yum 源配置文件的扩展名，让它们失效，因为只有扩展名是"*.repo"的文件才能作为 yum 源配置文件。当也可以删除其他几个 yum 源配置文件，但是如果删除了，当又想用网络作为 yum 源时，就没有了参考文件，所以最好还是修改扩展名。 命令如下：

   ```
   [root@localhost ~]# cd /etc/yum.repos.d/
   [root@localhost yum.repos.d]# mv CentOS-Base, repo CentOS-Base.repo.bak
   [root@localhost yum.repos.d]#mv CentOS-Debuginfo.repo CentOS-Debuginfo.repo.bak
   [root@localhost yum.repos.d]# mv CentOS-Vault.repo CentOS-Vault.repo.bak
   ```

3. 修改光盘 yum 源配置文件 CentOS-Media.repo，参照以下方修改：

   ```
   [root@localhost yum.repos.d]# vim CentOS-Media.repo
   [c6-media]
   name=CentOS-$releasever - Media
   baseurl=file:///mnt/cdrom
   #地址为你自己的光盘挂载地址
   #file:///media/cdrom/
   #file:///media/cdrecorder/
   #注释这两个的不存在地址
   gpgcheck=1
   enabled=1
   #把enabled=0改为enabled=1, 让这个yum源配置文件生效
   gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-6
   ```

   

### yum命令

#### 查询命令

使用 yum 对软件包执行查询操作，常用命令可分为以下几种：  

- yum list：查询所有已安装和可安装的软件包。例如：

```
[root@localhost yum.repos.d]# yum list
#查询所有可用软件包列表
Installed Packages
#已经安装的软件包
ConsdeKit.i686 0.4.1-3.el6
@anaconda-CentOS-201207051201 J386/6.3
ConsdeKit-libs.i686 0.4.1-3.el6 @anaconda-CentOS-201207051201 J386/6.3
…省略部分输出…
Available Packages
#还可以安装的软件包
389-ds-base.i686 1.2.10.2-15.el6 c6-media
389-ds-base-devel.i686 1.2.10.2-15.el6 c6-media
#软件名 版本 所在位置（光盘）
…省略部分输出…
```

- yum list 包名：查询执行软件包的安装情况。例如：

```
[root@localhost yum.repos.d]# yum list samba
Available Packages samba.i686 3.5.10-125.el6 c6-media
#查询 samba 软件包的安装情况
```

* yum search 关键字：从 yum 源服务器上查找与关键字相关的所有软件包。例如：

  ```
  [root@localhost yum.repos.d]# yum search samba
  #搜索服务器上所有和samba相关的软件包
  ========================N/S Matched:
  samba =============================
  samba-client.i686：Samba client programs
  samba-common.i686：Files used by both Samba servers and clients
  samba-doc.i686: Documentation for the Samba suite
  …省略部分输出…
  Name and summary matches only, use"search all" for everything.
  ```

* yum info 包名：查询执行软件包的详细信息。例如：

  ```
  [root@localhost yum.repos.d]# yum info samba
  #查询samba软件包的信息
  Available Packages <-没有安装
  Name : samba <-包名
  Arch : i686 <-适合的硬件平台
  Version : 3.5.10 <―版本
  Release : 125.el6 <—发布版本
  Size : 4.9M <—大小
  Repo : c6-media <-在光盘上
  …省略部分输出…
  ```

#### 安装命令

```
[root@localhost yum.repos.d]# yum -y install 包名
```

其中：

- install：表示安装软件包。

- -y：自动回答 yes。如果不加 -y，那么每个安装的软件都需要手工回答 yes；

#### 升级命令

使用 yum 升级软件包，需确保 yum 源服务器中软件包的版本比本机安装的软件包版本高。

yum 升级软件包常用命令如下：

- yum -y update：升级所有软件包。不过考虑到服务器强调稳定性，因此该命令并不常用。

- yum -y update 包名：升级特定的软件包。

#### yum 卸载命令

使用 yum 卸载软件包时，会同时卸载所有与该包有依赖关系的其他软件包，即便有依赖包属于系统运行必备文件，也会被 yum 无情卸载，带来的直接后果就是使系统崩溃。  

除非你能确定卸载此包以及它的所有依赖包不会对系统产生影响，否则不要使用 yum 卸载软件包。

```
[root@localhost yum.repos.d]# yum remove 包名
#卸载指定的软件包
```

### yum管理软件组

在安装 Linux 系统时，我们可以根据需要自定义安装软件包，如图 1 所示：

![image-20221201193431754](linux-software/image-20221201193431754.png)

选择“Customize now”，会进入图 2 所示的页面：

![img](linux-software/Gk8n9TwHIGRZyAMAdEBo31Tz2VP4gvs6u7b66A9r3A0.jpegtoken=W.jpeg)

图 2 中所示为 Linux 列出的许多软件包组，例如编辑器、系统工具、开发工具等。在此页面，我们可以根据需要选择要安装的软件包。  

除了像图 1、图 2 这样在系统安装过程中自选软件包组进行安装之外，当系统安装完成后，我们也可以通过 yum 命令来管理图 2 中的这些软件包组。  

yum 命令除了可以对软件包进行查询、安装、升级和卸载外，还可完成对软件包组的查询、安装和卸载操作。

既然是软件包组，说明包含不只一个软件包，通过 yum 命令可以查询某软件包组中具体包含的软件包，命令格式如下：

```
[root@localhost ~]#yum groupinfo 软件组名
#查询软件组中包含的软件
```

例如，查询 Web Server 软件包组中包含的软件包，可使用如下命令：

```
[root@localhost ~]#yum groupinfo "Web Server"
#查询软件组"Webserver"中包含的软件
```

使用 yum 安装软件包组的命令格式如下：

```
[root@localhost ~]#yum groupinstall 软件组名
#安装指定软件组，组名可以由grouplist查询出来
```

yum 卸载软件包组的命令格式如下：

```
[root@localhost ~]# yum groupremove 软件组名
#卸载指定软件组
```

yum 软件包组管理命令更适合安装功能相对集中的软件包集合。例如，在初始安装 Linux 时没有安装图形界面，但后来发现需要图形界面的支持，这时可以手工安装图形界面软件组（X Window System 和 Desktop），就可以使用图形界面了。

## 源码包安装

Linux 系统中，绝大多数软件的源代码都是用 C 语言编写的，少部分用 [C++](http://m.biancheng.net/cplus/)（或其他语言）编写。因此要想安装源码包，必须安装 gcc 编译器（如果涉及 C++ 源码程序，还需要安装 gcc-c++）。

除了安装编译器，还需要安装 make 编译命令。要知道，编译源码包可不像编译一个 hello.c 文件那样轻松，包中含大量的源码文件，且文件之间有着非常复杂的关联，直接决定着各文件编译的先后顺序，因此手动编译费时费力，而使用 make 命令可以完成对源码包的自动编译。

### 安装流程

本节仍然以安装 apache 为例，安装过程分为如下几步：  

1. 下载 apache 源码包。该软件的源码包可通过官方网站 http://httpd.apache.org/download.cgi#apache24 下载，得到的源码包格式为压缩包（ ".tar.gz" 或 ".tar.bz2" ）。  

> 将各种文件分门别类保存在对应的目录中，应该成为合格 Linux 管理员约定俗成的习惯。Linux 系统中用于保存源代码的位置主要有 2 个，分别是 "/usr/src" 和 "/usr/local/src"，其中 "/usr/src" 用来保存内核源代码，"/usr/local/src" 用来保存用户下载的源代码。

2. 将源码包进行解压缩，使用命令如下：

   ```
   [root@localhost ~]#tar -zxvf httpd-2.2.9.tar.gz|more
   ```

3. 进入解压目录，执行如下命令：

   ```
   [root@localhost ~]# ls
   anaconda-ks.cfg httpd-2.2.9 httpd-2.2.9.tar.gz install.log install.log.syslog
   [root@localhost ~]# cd httpd-2.2.9
   ```

4. `./configure` 软件配置与检查。这一步主要完成以下 3 项任务：

   1. 检测系统环境是否符合安装要求。

   2. 定义需要的功能选项。通过 "./configure--prefix=安装路径" 可以指定安装路径。注意，configure 不是系统命令，而是源码包软件自带的一个脚本程序，所以必须采用 "./configure" 方式执行（"./" 代表在当前目录下）。  

      > "./configure" 支持的功能选项较多，可执行 "./configure--help" 命令查询其支持的功能

   3. 把系统环境的检测结果和定义好的功能选项写入 Makefile 文件，因为后续的编译和安装需要依赖这个文件的内容。

   此步具体执行代码如下：

   ```shell
   [root@localhost httpd-2.2.9]# ./configure --prefix=/usr/local/apache2
   checking for chosen layout...Apache
   checking for working mkdir -p…yes
   checking build system type...i686-pc-linux-gnu
   checking host system type...i686-pc-linux-gnu
   checking target system typa...i686-pc-linux-gnu
   …省略部分输出…
   ```

5. make 编译。make 会调用 gcc 编译器，并读取 Makefile 文件中的信息进行系统软件编译。编译的目的就是把源码程序转变为能被 Linux 识别的可执行文件，这些可执行文件保存在当前目录下。执行的编译命令如下：

```
[root@localhost httpd-2.2.9]# make
```

6. 正式开始安装软件，这里通常会写清程序的安装位置，如果没有，则建议读者把安装的执行过程保存下来，以备将来删除软件时使用。安装指令如下：

```
[root@localhost httpd-2.2.9]# make install
```

整个过程不报错，即为安装成功。  

安装源码包过程中，如果出现“error”（或“warning”）且安装过程停止，表示安装失败；反之，如果仅出现警告信息，但安装过程还在继续，这并不是安装失败，顶多使软件部分功能无法使用。

> 注意，如果在 "./configure" 或 "make" 编译中报错，则在重新执行命令前一定要执行 make clean 命令，它会清空 Makefile 文件或编译产生的 ".o" 头文件。

### 卸载源码包

通过源码包方式安装的各个软件，其安装文件独自保存在 /usr/local/ 目录下的各子目录中。例如，apache 所有的安装文件都保存在 /usr/local/apache2 目录下。这就为源码包的卸载提供了便利。  

源码包的卸载，只需要找到软件的安装位置，直接删除所在目录即可，不会遗留任何垃圾文件。需要读者注意的是，在删除软件之前，应先将软件停止服务。  

以删除 apache 为例，只需关闭 apache 服务后执行如下命令即可：

```
[root@localhost ~]# rm -rf /usr/local/apache2/
```

### 升级源码包

Linux 系统中更新用源码包安装的软件，除了卸载重装这种简单粗暴的方法外，还可以下载补丁文件更新源码包，用新的源码包重新编译安装软件。比较两种方式，后者更新软件的速度更快。  

使用补丁文件更新源码包，省去了用 ./configured 生成新的 Makefile 文件，还省去了大量的编译工作，因此效率更高。

#### diff命令

Linux 系统中可以使用 diff 命令对比出新旧软件的不同，并生成补丁文件。

diff 命令基本格式为：

```
[root@localhost ~]# diff 选项 old new
#比较old和new文件的不同
```

此命令中可使用如下几个选项：

- -a：将任何文档当作文本文档处理；

- -b：忽略空格造成的不同；

- -B：忽略空白行造成的不同；

- -I：忽略大小写造成的不同；

- -N：当比较两个目录时，如果某个文件只在一个目录中，则在另一个目录中视作空文件；

- -r：当比较目录时，递归比较子目录；

- -u：使用同一输出格式；

从生成补丁文件，到使用其实现更新软件的目的，为了让读者清楚地了解整个过程的来龙去脉，下面我们自己创建两个文件（分别模拟旧软件和新软件），通过对比新旧文件生成补丁文件，最后利用补丁文件更新旧文件，具体步骤如下：  

1. 创建两个文件，执行如下命令：

```shell
[root@localhost ~]# mkdir test
#建立测试目录
[root@localhost ~]# cd test
#进入测试目录
[root@localhost test]# vi old.txt
our
school
is
lampbrother
#文件old.txt，为了便于比较，将每行分开
[root@localhost test]# vi new.txt
our
school
is
lampbrother
in
Beijing
#文件new.txt
```

2. 利用 diff 命令，比较两个文件（old.txt 和 new.txt）的不同，并生成补丁文件（txt.patch），执行代码如下：

```shell
[root@localhost test]# diff -Naur /root/test/old.txt /root/test/new.txt > txt. patch
#比较两个文件的不同，同时生成txt.patch补丁文件
[root@localhost test]#vi txt.patch
#查看一下这个文件
--/root/test/old.txt 2012-11-23 05:51:14.347954373 +0800
#前一个文件
+ + + /root/test/new.txt 2012-11-23 05:50:05.772988210 +0800
#后一个文件
@@-2, 3+2, 5@@
school
is
lampbrother
+in
+beijing
#后一个文件比前一个文件多两行（用+表示）
```

3. 利用补丁文件 txt.patch 更新 old.txt 旧文件，实现此步操作需利用 patch 命令，该命令基本格式如下：

```
[root@localhost test]# patch -pn < 补丁文件
#按照补丁文件进行更新
```

-pn 选项中，n 为数字（例如 p1、p2、p3 等），pn 表示按照补丁文件中的路径，指定更新文件的位置。  

这里对 -pn 选项的使用做一下额外说明。我们知道，补丁文件是要打入旧文件的，但是当前所在目录和补丁文件中记录的目录不一定是匹配的，需要 "-pn" 选项来同步两个目录。  

例如，当前位于 "/root/test/" 目录下（要打补丁的旧文件就在当前目录下），补丁文件中记录的文件目录为 "/root/test/dd.txt"，如果写入 "-p1"（在补丁文件目录中取消一级目录），那么补丁文件会打入 "root/test/root/test/old.txt" 文件中，这显然是不对的；如果写入的是 "-p2"（在补丁文件目录中取消二级目录），补丁文件会打入 "/root/test/test/old.txt" 文件中，这显然也不对。如果写入的是 "-p3"（在补丁文件目录中取消三级目录），补丁文件会打入 "/root/test/old.txt" 文件中，old.txt 文件就在这个目录下，所以应该用 "-p3" 选项。  

如果当前所在目录是 "/root/" 目录呢？因为补丁文件中记录的文件目录为 "/root/test/old.txt"，所以这里就应该用 "-p2" 选项（代表取消两级目录），补丁打在当前目录下的 "test/old.txt" 文件上。  

因此，-pn 选项可以这样理解，即想要在补丁文件中所记录的目录中取消几个 "/"，n 就是几。去掉目录的目的是和当前所在目录匹配。  

现在更新 "old.txt" 文件，命令如下：

```
[root@localhost test]# patch -p3 < txt.patch
patching file old.txt
#给old.txt文件打补丁
[root@localhost test]# cat old.txt
#查看一下dd.txt文件的内容
our
school
is
lampbrother
in
Beijing
#多出了in Beijing两行
```

可以看到，通过使用补丁文件 txt.patch 对旧文件进行更新，使得旧文件和新文件完全相同。

通过这个例子，大家要明白以下两点：

1. 给旧文件打补丁依赖的不是新文件，而是补丁文件，所以即使新文件被删除也没有关系。
2. 补丁文件中记录的目录和当前所在目录需要通过 "-pn" 选项实现同步，否则更新可能失败。

本节仍以 apache 为例，通过从官网上下载的补丁文件 "mod_proxy_ftp_CVE-2008-2939.diff"，更新 httpd-2.2.9 版本的 apache。

具体更新步骤如下：

1. 从 apache 官网上下载补丁文件；
2. 复制补丁文件到 apache 源码包解压目录中，执行命令如下：

```
[root@localhost ~]# cp mod_proxy_ftp_CVE-2008-2939.diff httpd-2.2.9
```

3. 给旧 apache 打入补丁，具体执行命令如下：

```shell
[root@localhost ~]# cd httpd-2.2.9
#进入apache源码目录
[root@localhost httpd-2.2.9]# vi mod_proxy_ftp_CVE-2008-2939.diff
#查看补丁文件
--modules/proxy/mod_proxy_ftp.c (Revision 682869)
+ + + modules/proxy/mod_proxy_ftp.c (Revision 682870)
…省略部分输出…
#查看一下补丁文件中记录的目录，以便一会儿和当前所在目录同步
[root@localhost httpd-2.2.9]# patch - p0 < mod_proxy_ftp_CVE-2008-2939.diff
#打入补丁
```

为什么是 "-p0" 呢？因为当前在 "/root/httpd-2.2.9" 目录中，但补丁文件中记录的目录是 "modules/proxy/mod_proxy_ftp.c"，就在当前所在目录中，因此一个 "/" 都不需要去掉，所以是 "-p0"。

4. 重新编译 apache 源码包，执行如下命令：

```
[root@localhost httpd-2.2.9]# make
```

5. 安装 apache，执行如下命令：

   ```
   [root@localhost httpd-2.2.9]# make install
   ```

   不难发现，打补丁更新软件的过程比安装软件少了 "./configure" 步骤，且编译时也只是编译变化的位置，编译速度更快。



## linux函数库

Linux 系统中存在大量的函数库。简单来讲，函数库就是一些函数的集合，每个函数都具有独立的功能且能被外界调用。我们在编写代码时，有些功能根本不需要自己实现，直接调用函数库中的函数即可。  

需要注意的是，函数库中的函数并不是以源代码的形式存在的，而是经过编译后生成的二进制文件，这些文件无法独立运行，只有链接到我们编写的程序中才可以运行。

Linux 系统中的函数库分为 2 种，分别是静态函数库（简称静态库）和动态函数库（也称为共享函数库，简称动态库或共享库），两者的主要区别在于，程序调用函数时，将函数整合到程序中的时机不同： 

- 静态函数库在程序编译时就会整合到程序中，换句话说，程序运行前函数库就已经被加载。这样做的好处是程序运行时不再需要调用外部函数库，可直接执行；缺点也很明显，所有内容都整合到程序中，编译文件会比较大，且一旦静态函数库改变，程序就需要重新编译。

- 动态函数库在程序运行时才被加载（如图 1 所示），程序中只保存对函数库的指向（程序编译仅对其做简单的引用）。![image-20221201195735254](linux-software/image-20221201195735254.png)

使用动态函数库的好处是，程序生成的可执行程序体积比较小，且升级函数库时无需对整个程序重新编译；缺点是，如果程序执行时函数库出现问题，则程序将不能正确运行。

Linux 系统中，静态函数库文件扩展名是 ".a"，文件通常命令为 libxxx.a（xxx 为文件名）；动态函数库扩展名为 ".so"，文件通常命令为 libxxx.so.major.minor（xxx 为文件名，major 为主版本号，minor 为副版本号）。

目前，Linux 系统中大多数都是动态函数库（主要考虑到软件的升级方便），其中被系统程序调用的函数库主要存放在 "/usr/lib" 和 "/lib" 中；Linux 内核所调用的函数库主要存放在 "/lib/modules" 中。

注意，函数库（尤其是动态函数库）的存放位置非常重要，轻易不要做更改。

### 函数库安装

Linux 发行版众多，不同 Linux 版本安装函数库的方式不同。CentOS 中，安装函数库可直接使用 yum 命令。  

例如，安装 curses 函数库命令如下：

```
[root@Linux ~]# yum install ncurses-devel
```

正常情况下，函数库安装完成后就可以直接被系统识别，但凡事都有万一。这里先想一个问题，如何查看可执行程序调用了哪些函数库呢？通过以下命令即可：

```
[root@localhost ~]# ldd -v 可执行文件名
```

-v 选项的含义是显示详细版本信息（不是必须使用）。

例如，查看 ls 命令调用了哪些函数库，命令如下：

```
[root@localhost ~]# ldd /bin/ls
linux-gate.so.1 => (0x00d56000)
libselinux.so.1 =>/lib/libselinux.so.1 (0x00cc8000)
librt.so.1 =>/lib/librt.so.1 (0x00cb8000)
libcap.so.2 => /lib/libcap.so.2 (0x00160000)
libacl.so.1 => /lib/libacl.so.1 (0x00140000)
libc.so.6 => /lib/libc.so.6 (0x00ab8000)
libdl.so.2 => /lib/libdl.so.2 (0x00ab0000)
/lib/ld-linux.so.2 (0x00a88000)
libpthread.so.0 => /lib/libpthread.so.0 (0x00c50000)
libattr.so.1 =>/lib/libattr.so.1 (0x00158000)
```

如果函数库安装后仍无法使用（运行程序时会提示找不到某个函数库），这时就需要对函数库的配置文件进行手动调整，也很简单，只需进行如下操作：  

1. 将函数库文件放入指定位置（通常放在 "/usr/lib" 或 "/lib" 中），然后把函数库所在目录写入 "/etc/ld.so.conf" 文件。例如：

```shell
[root@localhost ~]# cp *.so /usr/lib/
#把函数库复制到/usr/lib/目录中
[root@localhost ~]# vi /etc/ld.so.conf
#修改函数库配置文件
include ld.so.conf.d/*.conf
/usr/lib
#写入函数库所在目录（其实/usr/lib/目录默认已经被识别）
```

注意，这里写入的是函数库所在的目录，而不单单是函数库的文件名。另外，如果自己在其他目录中创建了函数库文件，这里也可以直接在 "/etc/ld.so.conf" 文件中写入函数库文件所在的完整目录。

2. 使用 ldconfig 命令重新读取 /etc/ld.so.conf 文件，把新函数库读入缓存。命令如下：

```
[root@localhost ~]# ldconfig
#从/etc/ld.so.conf文件中把函数库读入缓存
[root@localhost ~]# ldconfig -p
#列出系统缓存中所有识别的函数库
```

# 服务管理

我们知道，系统服务是在后台运行的应用程序，并且可以提供一些本地系统或网络的功能。我们把这些应用程序称作服务，也就是 Service。不过，我们有时会看到 Daemon 的叫法，Daemon 的英文原意是"守护神"，在这里是"守护进程"的意思。  

那么，什么是守护进程？它和服务又有什么关系呢？守护进程就是为了实现服务、功能的进程。比如，我们的 apache 服务就是服务（Service），它是用来实现 Web 服务的。那么，启动 apache 服务的进程是哪个进程呢？就是 httpd 这个守护进程（Daemon）。也就是说，守护进程就是服务在后台运行的真实进程。  

如果我们分不清服务和守护进程，那么也没有什么关系，可以把服务与守护进程等同起来。在 Linux 中就是通过启动 httpd 进程来启动 apache 服务的，你可以把 httpd 进程当作 apache 服务的别名来理解。

## 服务分类

Linux 中的服务按照安装方法不同可以分为 RPM 包默认安装的服务和源码包安装的服务两大类。其中，RPM 包默认安装的服务又因为启动与自启动管理方法不同分为独立的服务和基于 xinetd 的服务。服务分类的关系图如图 1 所示。

![img](linux-software/SPC6ORvq73CSaTEuCmViMxwYfPvpFUmn-lRpI1FFV9I.jpegtoken=W.jpeg)

源码包安装到我们手工指定的位置当中，而 RPM 包安装到系统默认位置当中（可以通过"rpm -ql 包名"命令查询）。也就是说，RPM 包安装到系统默认位置，可以被服务管理命令识别；但是源码包安装到手工指定位置，当然就不能被服务管理命令识别了（可以手工修改为被服务管理命令识别）。

所以，RPM 包默认安装的服务和源码包安装的服务的管理方法不同，我们把它们当成不同的服务分类。

RPM 包默认安装的服务又可以分为两种：

- 独立的服务：就是独立启动的意思，这种服务可以自行启动，而不用依赖其他的管理服务。因为不依赖其他的管理服务，所以，当客户端请求访问时，独立的服务响应请求更快速。目前，Linux 中的大多数服务都是独立的服务，如 apache 服务、FTP 服务、Samba 服务等。

- 基于 xinetd 的服务：这种服务就不能独立启动了，而要依靠管理服务来调用。这个负责管理的服务就是 xinetd 服务。xinetd 服务是系统的超级守护进程，其作用就是管理不能独立启动的服务。当有客户端请求时，先请求 xinetd 服务，由 xinetd 服务去唤醒相对应的服务。当客户端请求结束后，被唤醒的服务会关闭并释放资源。这样做的好处是只需要持续启动 xinetd 服务，而其他基于 xinetd 的服务只有在需要时才被启动，不会占用过多的服务器资源。但是这种服务由于在有客户端请求时才会被唤醒，所以响应时间相对较长。

源码包安装的服务。这些服务是通过源码包安装的，所以安装位置都是手工指定的。由于不能被系统中的服务管理命令直接识别，所以这些服务的启动与自启动方法一般都是源码包设计好的。每个源码包的启动脚本都不一样，一般需要查看说明文档才能确定。

### 查看服务

我们已经知道 Linux 服务的分类了，那么应该如何区分这些服务呢？首先要区分 RPM 包默认安装的服务和源码包安装的服务。源码包安装的服务是不能被服务管理命令直接找到的，而且一般会安装到 /usr/local/ 目录中。  

也就是说，在 /usr/local/ 目录中的服务都应该是通过源码包安装的服务。RPM 包默认安装的服务都会安装到系统默认位置，所以是可以被服务管理命令（如 service、chkconfig）识别的。

其次，在 RPM 包默认安装的服务中怎么区分独立的服务和基于 xinetd 的服务？这就要依靠 chkconfig 命令了。chkconfig 是管理 RPM 包默认安装的服务的自启动的命令，这里仅利用这条命令的查看功能。使用这条命令还能看到 RPM 包默认安装的所有服务。命令格式如下：

```
[root@localhost ~]# chkconfig --list [服务名]
```

- --list：列出 RPM 包默认安装的所有服务的自启动状态；

```
[root@localhost ~]# chkconfig -list
#列出系统中RPM包默认安装的所有服务的自启动状态
abrt-ccpp 0:关闭 1:关闭 2:关闭 3:启用 4:关闭 5:启用 6:关闭
abrt-oops 0:关闭 1:关闭 2:关闭 3:启用 4:关闭 5:启用 6:关闭
…省略部分输出…
udev-post 0:关闭 1:启用 2:启用 3:启用 4:启用 5:启用 6:关闭
ypbind 0:关闭 1:关闭 2:关闭 3:关闭 4:关闭 5:关闭 6:关闭
```

这条命令的第一列为服务的名称，后面的 0~6 代表在不同的运行级别中这个服务是否开机时自动启动。这些服务都是独立的服务，因为它们不需要依赖其他任何服务就可以在相应的运行级别启动或自启动。但是没有看到基于 xinetd 的服务，那是因为系统中默认没有安装 xinetd 这个超级守护进程，需要我们手工安装。

安装命令如下：

```
[root@localhost ~]# rpm -ivh /mnt/cdrom/Packages/ xinetd-2.3.14-34.el6.i686.rpm
Preparing...
###############
[100%]
1:xinetd
###############
[100%]
#xinetd超级守护进程
```

这里需要注意的是，在 Linux 中基于 xinetd 的服务越来越少，原先很多基于 xinetd 的服务在新版本的 Linux 中已经变成了独立的服务。安装完 xinetd 超级守护进程之后，我们再查看一下，命令如下：

```
[root@localhost ~]# chkconfig --list
abrt-ccpp 0:关闭 1:关闭 2:关闭 3:启用 4:关闭 5:启用 6:关闭
abrt-oops 0:关闭 1:关闭 2:关闭 3:启用 4:关闭 5:启用 6:关闭
…省略部分输出…
udev-post 0:关闭 1:启用 2:启用 3：启用 4:启用 5:启用 6:关闭
xinetd 0:关闭 1:关闭 2:关闭 3:启用 4:启用 5:启用 6:关闭
ypbind 0:关闭 1:关闭 2:关闭 3:关闭 4:关闭 5:关闭 6:关闭
基于 xinetd 的服务：
chargen-dgram：关闭
chargen-stream：关闭
cvs：关闭
daytime-dgram：关闭
daytime-stream：关闭
discard-dgram：关闭
discard-stream：关闭
echo-dgram：关闭
echo-stream：关闭
rsync：关闭
tcpmux-server：关闭
time-dgram：关闭
time-stream：关闭
```

在刚刚的独立的服务之下出现了一些基于 xinetd 的服务，这些服务没有自己的运行级别，因为它们不是独立的服务，到底在哪个运行级别可以自启动，则要看 xinetd 服务是在哪个运行级别自启动的。

## 服务启动

独立的服务要想启动，主要有两种方法。  

1. 使用/etc/init.d/目录中的启动脚本来启动独立的服务,我们以启动 RPM 包默认安装的 httpd 服务为例，命令如下：

   ```shell
   [root@localhost ~]# /etc/init.d/httpd start
   正在启动httpd:
   [确定]
   #启动httpd服务
   [root@localhost ~]# /etc/init.d/httpd status
   httpd (pid 13313)正在运行…
   #查询httpd服务状态，并能够看到httpd服务的PID
   [root@localhost ~]#/etc/init.d/httpd stop
   停止 httpd:
   [确定]
   #停止httpd服务
   [root@localhost ~]#/etc/init.d/httpd restart
   停止httpd:
   [失败]
   正在启动httpd:
   [确定]
   重启动httpd服务
   ```

2. 使用service命令来启动独立的服务:

   ```
   [root@localhost ~]# service 独立服务名 start|stop|restart|...
   ```

   service 命令实际上只是一个脚本，这个脚本仍然需要调用 /etc/init.d/ 中的启动脚本来启动独立的服务。而且 service 命令是红帽系列 Linux 的专有命令，其他的 Linux 发行版本不一定拥有这条命令，所以我们并不推荐使用 service 命令来启动独立的服务。

### 开机启动

有三种方式实现开机自启。

#### chkconfig命令

管理独立服务的自启动，用法如下:

```
chkconfig [--level 运行级别][独立服务名][on|off]
```

- --level: 设定在哪个运行级别中开机自启动（on），或者关闭自启动（off）；

```shell
[root@localhost ~]# chkconfig --list | grep httpd
httpd 0:关闭 1:关闭 2:关闭 3:关闭 4:关闭 5:关闭 6:关闭
#查询httpd的自启动状态。所有的级别都是不自启动的
[root@localhost ~]# chkconfig --level 2345 httpd on
#设置apache服务在进入2、3、4、5级别时自启动
[root@localhost ~]# chkconfig --list | grep httpd
httpd 0:关闭 1:关闭 2:启用 3:启用 4:启用 5:启用 6:关闭
#查询apache服务的自启动状态。发现在2、3、4、5这4个运行级别中变为了"启用"
```

#### 修改 /etc/rc.d/rc.local 文件，设置服务自启动

这个文件是在系统启动时，在输入用户名和密码之前最后读取的文件（注意：/etc/rc.d/rc.loca和/etc/rc.local 文件是软链接，修改哪个文件都可以）。这个文件中有什么命令，都会在系统启动时调用。

如果我们把服务的启动命令放入这个文件，这个服务就会在开机时自启动。命令如下：

```
[root@localhost ~]#vi /etc/rc.d/rc.local
#!/bin/sh
#
#This script will be executed *after* all the other init scripts.
#You can put your own initialization stuff in here if you don't want to do the full Sys V style init stuff.
touch /var/lock/subsys/local
/etc/rc.d/init.d/httpd start
#在文件中加入apache的启动命令
```

这样，只要重启之后，apache 服务就会开机自启动了。推荐大家使用这种方法管理服务的自启动，有两点好处：  

- 第一，如果大家都采用这种方法管理服务的自启动，当我们碰到一台陌生的服务器时，只要查看这个文件就知道这台服务器到底自启动了哪些服务，便于集中管理。

- 第二，chkconfig 命令只能识别 RPM 包默认安装的服务，而不能识别源码包安装的服务。 源码包安装的服务的自启动也是通过 /etc/rc.d/rc.local 文件实现的，所以不会出现同一台服务器自启动了两种安装方法的同一个服务。

还要注意一下，修改 /etc/rc.d/rc.local 配置文件的自启动方法和 chkconfig 命令的自启动方法是两种不同的自启动方法。所以，就算通过修改 /etc/rc.d/rc.local 配置文件的方法让某个独立的服务自启动了，执行"chkconfig --list"命令并不到有什么变化。

#### 使用 ntsysv 命令管理自启动

使用 ntsysv 命令调用窗口模式来管理服务的自启动，非常简单。命令格式如下：

```
[root@localhost ~]# ntsysv [--level 运行级别]
```

- --level 运行级别：可以指定设定自启动的运行级别；

```
[root@localhost ~]# ntsysv --level 235
#只设定2、3、5级别的服务自启动
[root@localhost ~]# ntsysv
#按默认的运行级别设置服务自启动
```

![img](linux-software/-e9uN_GHZ0wQwHFxOHKDySU95y_eK26idnpc_1jiUjU.jpegtoken=W.jpeg)

这个命令的操作是这样的：

- 上下键：在不同服务之间移动；

- 空格键：选定或取消服务的自启动。也就是在服务之前是否输入"*"；

- Tab键：在不同项目之间切换；

- F1键：显示服务的说明；

需要注意的是，ntsysv 命令不仅可以管理独立服务的自启动，也可以管理基于 xinetd 服务的自启动。也就是说，只要是 RPM 包默认安装的服务都能被 ntsysv 命令管理。但是源码包安装的服务不行。  

这样管理服务的自启动多么方便，为什么还要学习其他的服务自启动管理命令呢？ ntsysv 命令虽然简单，但它是红帽系列 Linux 的专有命令，其他的 Linux 发行版本不一定拥有这条命令，而且条命令也不能管理源码包安装的服务，所以我们推荐大家使用 /etc/rc.d/rc.local 文件来管理服务的自启动。

## xinetd 服务

基于 xinetd 的服务没有自己独立的启动脚本程序，是需要依赖 xinetd 的启动脚本来启动的。xinetd 本身是独立的服务，所以 xinetd 服务自己的启动方法和独立服务的启动方法是一致的。  

但是，所有基于 xinetd 这个超级守护进程的其他服务就不是这样的了，必须修改该服务的配置文件，才能启动基于 xinetd 的服务。所有基于 xinetd 服务的配置文件都保存在 /etc/xinetd.d/ 目录中。  

我们使用 Telnet 服务来举例。Telnet 服务是用来进行系统远程管理的，端口是 23。不过需要注意的是，Telnet 服务的远程管理数据在网络中是明文传输的，非常不安全，所以在生产服务器上是不建议启动 Telnet 服务的。在生产服务器上，远程管理使用的是 ssh 协议，ssh 协议是加密的，更加安全。  

Telnet 服务也是分为"客户端/服务器端"的，其中服务器端是用来启动 Telnet 服务的，并不安全；客户端是用来连接服务器端或测试服务器的端口是否开启的，在实际工作中我们主要使用 Telnet 客户端来测试远程服务器开启了哪些端口。

虽然 Telnet 服务不安全，但 Telnet 服务是基于 xinetd 的服务，我们使用 Telnet 服务来学习一下基于 xinetd 服务的启动管理。在目前的 Linux 系统中，Telnet 的服务器端都是不安装的，如果进行测试，则需要手工安装。安装命令如下：

```
[root@localhost ~]#rpm-ivh/mnt/cdroin/Packages/telnet-server-0.17-47.el6.i686.rpm
[100%]
###############
Preparing...
1:telnet-server
###############
[100%]
#安装
[root@localhost ~]# chkconfig -list
#安装之后查询一下
…省略部分输出...
基于xinetd的服务：
chargen-dgram：关闭
chargen-stream：关闭
cvs：关闭
daytime-dgram：关闭
daytime-stream：关闭
discard-dgram：关闭
discard-stream：关闭
echo-dgram：关闭
echo-stream：关闭
rsync：关闭
tcpmux-server：关闭
telnet:关闭
time-dgram：关闭
time-stream：关闭
#Telnet服务已经安装，是基于xinetd的服务，自启动状态是关闭
```

接下来我们就要启动 Telnet 服务了。既然基于 xinetd 服务的配置文件都在 /etc/xinetd.d/ 目录中，那么 Telnet 服务的配置文件就是 /etc/xinetd.d/telnet。我们打开这个文件看看，如下：

```
[root@localhost ~]#vi /etc/xinetd.d/telnet
#default: on
#description: The telnet server serves telnet sessions; it uses \
#unencrypted username/password pairs for authentication.
service telnet
#服务的名称为telnet
{
flags = REUSE
#标志为REUSE，设定TCP/IP socket可重用
socketjtype = stream
#使用 TCP协议数据包
wait = no
#允许多个连接同时连接
user = root
#启动服务的用户为root
server = /usr/sbin/in.telnetd
#服务的启动程序
log_on_failure += USERID
#登录失败后，记录用户的ID
disable = yes
#服务不启动
}
```

如果想要启动 Telnet 服务，则只需要把 /etc/xinetd.d/telnet 文件中的"disable=yes"改为"disable=no"即可，"disable"代表取消，"disable=yes"代表取消是 yes，当然不启动服务；"disable=no"代表取消是 no，当然就是启动服务了。具体命令如下：

```shell
[root@localhost ~]#vi /etc/xinetd.d/telnet
#修改配置文件
service telnet {
…省略部分输出…
disable = no
#把 yes 改为no
}
[root@localhost ~]# service xinetd restart
#重启xinetd服务
停止 xinetd:
[确定]
正在启动xinetd:
[确定]
[root@localhost ~]# netstat -tlun|grep 23
tcp 0 0 :::23 :::* LISTEN
#查看端口，23端口启动，表示Telne服务已经启动了
```

基于 xinetd 服务的启动都是这样的，只需修改 /etc/xinetd.d/ 目录中的配置文件，然后重启 xientd 服务即可。

### 基于xientd 服务的自启动

基于 xinetd 服务的自启动管理有两种方法，分别是通过 chkconfig 命令管理自启动和通过 ntsysv 命令管理自启动。但是不能通过修改 /etc/rc.d/rc.local 配置文件来管理自启动，因为基于 xinetd 的服务没有自己的启动脚本程序。我们分别来看看。

#### 使用 chkconfig 命令管理自启动

chkconfig 自启动管理命令可以管理所有 RPM 包默认安装的服务，所以也可以用来管理基于 xinetd 服务的自启动。命令格式如下：

```
[root@localhost ~]# chkconfig 服务名 on|off
#基于xinetd的服务没有自己的运行级别，而依靠xinetd服务的运行级别，所以不用指定--level选项
```

```
[root@localhost ~]# chkconfig telnet on
#启动Telnet服务的自启动
[root@localhost ~]# chkconfig --list|grep telnet
telnet:启用
#查看服务的自启动，Telnet服务变为了"启用"
[root@localhost ~]# chkconfig telnet off
#关闭Telnet服务的自启动
[root@localhost ~]# chkconfig --list|grep telnet
telnet:关闭
#查看服务的自启动，Telnet服务变为了 "关闭"
```

## Linux源码包服务管理

源码包服务中所有的文件都会安装到指定目录当中，并且没有任何垃圾文件产生（Linux 的特性），所以服务的管理脚本程序也会安装到指定目录中。源码包服务的启动管理方式就是在服务的安装目录中找到管理脚本，然后执行这个脚本。  

问题来了，每个服务的启动脚本都是不一样的，我们怎么确定每个服务的启动脚本呢？还记得在安装源码包服务时，我们强调需要査看每个服务的说明文档吗（一般是 INSTALL 或 READEM）？在这个说明文档中会明确地告诉大家服务的启动脚本是哪个文件。  

我们用 apache 服务来举例。一般 apache 服务的安装位置是 /usr/local/apache2/ 目录，那么 apache 服务的启动脚本就是 /usr/local/apache2/bin/apachectl 文件（查询 apache 说明文档得知）。启动命令如下：

```
[root@localhost ~]# /usr/local/apache2/bin/apachectl start|stop|restart|...
#源码包服务的启动管理
```

源码包服务的白启动管理也不能依靠系统的服务管理命令，而只能把标准启动命令写入 /etc/rc.d/rc.local 文件中。系统在启动过程中读取 /etc/rc.d/rc.local 文件时，就会调用源码包服务的启动脚本，从而让该服务开机自启动。

在默认情况下，源码包服务是不能被系统的服务管理命令所识别和管理的，但是如果我们做一些设定，则也是可以让源码包服务被系统的服务管理命令所识别和管理的。不过笔者并不推荐大家这样做，因为这会让本来区别很明确的源码包服务和 RPM 包服务变得容易混淆，不利于系统维护和管理。  

我们做一个实验，看看如何把源码包安装的 apache 服务变为和 RPM 包默认安装的 apache 服务一样，可以被 service、chkconfig、ntsysv 命令所识别。实验如下：

```shell
[root@localhost ~]# ln -s /usr/local/apache2/bin/apachectl /etc/±nit.d/apache
#service命令其实只是在/etc/init.d/目录中查找是否有服务的启动脚本，所以我们只需要做一个软链接,把源码包的启动脚本链接到/etc/init.d/目录中,就能被service命令所管理了。为了照顾大家的习惯，我把软链接文件命名为apache,注意这不是RPM包默认安装的apache服务
[root@localhost ~]# service apache restart
#虽然RPM包默认安装的apache服务被卸载了,但是service命令也能够生效
```

让源码包安装的apache服务能被chkconfig命令管理自启动:

```shell
[root@localhost ~]# vi /etc/init.d/apache
#修改源码包安装的apache服务的启动脚本(注意此文件是软链接,所以修改的还是源码包启动脚本)
#!/bin/sh
#
#chkconfig: 35 86 76
#指定httpd脚本可以被chkconfig命令所管理
#格式是：chkconfig：运行级别 启动顺序 关闭顺序
#这里我们让apache服务在3和5级别中能被chkconfig命令所管理，启动顺序是S86，关闭顺序是K76
#(自定顺序，不要和系统中已有的启动顺序冲突)
#description: source package apache
#说明，内容随意
#以上两句话必须加入,才能被chkconfig命令所识别 ...省略部分输出...
[root@localhost ~]# chkconfig --add apache
#让chkconfig命令能够管理源码包安装的apache服务
[root01ocalhost ~]# chkconfig --list | grep apache
apache 0:关闭 1:关闭 2:关闭 3:关闭 4:关闭 5:关闭 6:关闭
#很神奇吧,虽然RPM包默认安装的apache服务被删除了,但是chkconfig命令可以管理源码包安装的tapache服务
```

