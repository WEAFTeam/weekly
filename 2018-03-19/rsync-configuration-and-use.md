---
title: rsync的使用与配置
description: rsync的使用与配置
tags:
  - Linux运维
author: 
  - earth
thumbnail: 'http://us-forever.com/img/linux-icon.jpg'
date: 2018-03-25 21:08:56
category: Linux
---
一、什么是rsync
-------
**rsync**，remote synchronize顾名思意就知道它是一款实现远程同步功能的软件，它在同步文件的同时，可以保持原来文件的权限、时间、软硬链接等附加信息。 rsync是用 “rsync 算法”提供了一个客户机和远程文件服务器的文件同步的快速方法，而且可以通过ssh方式来传输文件，这样其保密性也非常好，另外它还是免费的软件。

二、rsync的安装
------
rysnc的官方网站：http://rsync.samba.org
可以从上面得到最新的版本。目前最新版是3.1.2。当然，因为rsync是一款如此有用的软件，所以很多Linux的发行版本都将它收录在内了。

<!-- more -->

　　软件包安装

| 命令|平台 |
|---|---|
|# sudo apt-get  install  rsync |注：在debian、ubuntu 等在线安装方法；|
|# yum install rsync    |注：Fedora、Redhat 等在线安装方法；|
|# rpm -ivh rsync       |注：Fedora、Redhat 等rpm包安装方法；|

　　其它Linux发行版，请用相应的软件包管理方法来安装。

　　源码包安装
```
　　tar xvf  rsync-xxx.tar.gz
　　cd rsync-xxx
　　./configure --prefix=/usr  ;make ;make install   注：在用源码
```
包编译安装之前，您得安装gcc等编译开具才行；
三、rsync的配置
-----------
rsync的主要有以下三个配置文件**rsyncd.conf**(主配置文件)、**rsyncd.secrets**(密码文件)、**rsyncd.motd**(rysnc服务器信息)
比如我们要备份服务器上的/home和/opt，在/home中我想把easylife和samba目录排除在外；
``` conf
　　# Distributed under the terms of the GNU General Public License v2
　　# Minimal configuration file for rsync daemon
　　# See rsync(1) and rsyncd.conf(5) man pages for help

　　# This line is required by the /etc/init.d/rsyncd script
　　pid file = /var/run/rsyncd.pid   
　　port = 873
　　address = 192.168.1.171  
　　#uid = nobody 
　　#gid = nobody    
　　uid = root   
　　gid = root  

　　use chroot = yes  
　　read only = yes 

　　#limit access to private LANs
　　hosts allow=192.168.1.0/255.255.255.0 10.0.1.0/255.255.255.0  
　　hosts deny=*

　　max connections = 5 
　　motd file = /etc/rsyncd.motd

　　#This will give you a separate log file
　　#log file = /var/log/rsync.log

　　#This will log every file transferred - up to 85,000+ per user, per sync
　　#transfer logging = yes

　　log format = %t %a %m %f %b
　　syslog facility = local3
　　timeout = 300

　　[rhel4home]   
　　path = /home    
　　list=yes 
　　ignore errors 
　　auth users = root
　　secrets file = /etc/rsyncd.secrets  
　　comment = This is RHEL 4 data  
　　exclude = easylife/  samba/     

　　[rhel4opt]
　　path = /opt 
　　list=no
　　ignore errors
　　comment = This is RHEL 4 opt 
　　auth users = easylife
　　secrets file = /etc/rsyncd/rsyncd.secrets
```

　　注：关于auth users是必须在服务器上存在的真实的系统用户，如果你想用多个用户以,号隔开，比如auth users = easylife,root
　　设定密码文件

　　密码文件格式很简单，rsyncd.secrets的内容格式为：

　　用户名:密码

　　我们在例子中rsyncd.secrets的内容如下类似的；在文档中说，有些系统不支持长密码，自己尝试着设置一下吧。
``` conf
　　easylife:keer
　　root:mike
```
```
　　chown root.root rsyncd.secrets 　#修改属主
　　chmod 600 rsyncd.secrets     #修改权限
```
　　注：1、将rsyncd.secrets这个密码文件的文件属性设为root拥有, 且权限要设为600, 否则无法备份成功!            出于安全目的，文件的属性必需是只有属主可读。
　　　　2、这里的密码值得注意，为了安全你不能把系统用户的密码写在这里。比如你的系统用户easylife密码是000000，为了安全你可以让rsync中的easylife为keer。这和samba的用户认证的密码原理是差不多的。

　　设定rsyncd.motd 文件;

　 　它是定义rysnc服务器信息的，也就是用户登录信息。比如让用户知道这个服务器是谁提供的等；类似ftp服务器登录时，我们所看到的 linuxsir.org ftp ……。 当然这在全局定义变量时，并不是必须的，你可以用#号注掉，或删除；我在这里写了一个 rsyncd.motd的内容为：
``` conf
　　++++++++++++++++++++++++++++++++++++++++++++++
　　Welcome to use the mike.org.cn rsync services!
           2002------2009
　　++++++++++++++++++++++++++++++++++++++++++++++
```
四、启动rsync服务器
--------

   相当简单，有以下几种方法

　　A、--daemon参数方式，是让rsync以服务器模式运行
```
　　#/usr/bin/rsync --daemon  --config=/etc/rsyncd/rsyncd.conf 　#--config用于指定rsyncd.conf的位置,如果在/etc下可以不写
```
　　B、xinetd方式
```
　　修改services加入如下内容
　　# nano -w /etc/services

　　rsync　　873/tcp　　# rsync 
　　rsync　　873/udp　　# rsync
```
　　这一步一般可以不做，通常都有这两行(我的RHEL4和GENTOO默认都有)。修改的目的是让系统知道873端口对应的服务名为rsync。如没有的话就自行加入。

　　设定 /etc/xinetd.d/rsync, 简单例子如下:
```
　　# default: off
　　# description: The rsync server is a good addition to am ftp server, as it \
　　#       allows crc checksumming etc.
　　service rsync
　　{
        disable = no
        socket_type     = stream
        wait            = no
        user            = root
        server          = /usr/bin/rsync
        server_args     = --daemon
        log_on_failure  += USERID
　　}
```
　　上述, 主要是要打开rsync這個daemon, 一旦有rsync client要连接時, xinetd会把它转介給 rsyncd(port 873)。然后service xinetd restart, 使上述设定生效.

　　rsync服务器和防火墙

　　Linux 防火墙是用iptables，所以我们至少在服务器端要让你所定义的rsync 服务器端口通过，客户端上也应该让通过。
```
　　#iptables -A INPUT -p tcp -m state --state NEW  -m tcp --dport 873 -j ACCEPT
　　#iptables -L  查看一下防火墙是不是打开了 873端口
```
　　如果你不太懂防火墙的配置，可以先service iptables stop 将防火墙关掉。当然在生产环境这是很危险的，做实验才可以这么做哟！

五、通过rsync客户端来同步数据
------------

### B1、列出rsync 服务器上的所提供的同步内容；

　　首先：我们看看rsync服务器上提供了哪些可用的数据源

　　# rsync  --list-only  root@192.168.145.5::
　　++++++++++++++++++++++++++++++++++++++++++++++
　　Welcome to use the mike.org.cn rsync services!
    　　       2002------2009
　　++++++++++++++++++++++++++++++++++++++++++++++

　　rhel4home       This is RHEL 4 data

　 　注：前面是rsync所提供的数据源，也就是我们在rsyncd.conf中所写的[rhel4home]模块。而“This is RHEL 4 data”是由[rhel4home]模块中的 comment = This is RHEL 4 data 提供的；为什么没有把rhel4opt数据源列出来呢？因为我们在[rhel4opt]中已经把list=no了。

　　$ rsync  --list-only  root@192.168.145.5::rhel4home 

　　++++++++++++++++++++++++++++++++++++++++++++++
　　Welcome to use the mike.org.cn rsync services!
 　　          2002------2009
　　++++++++++++++++++++++++++++++++++++++++++++++

　　Password: 
　　drwxr-xr-x        4096 2009/03/15 21:33:13 .
　　-rw-r--r--        1018 2009/03/02 02:33:41 ks.cfg
　　-rwxr-xr-x       21288 2009/03/15 21:33:13 wgetpaste
　　drwxrwxr-x        4096 2008/10/28 21:04:05 cvsroot
　　drwx------        4096 2008/11/30 16:30:58 easylife
　　drwsr-sr-x        4096 2008/09/20 22:18:05 giddir
　　drwx------        4096 2008/09/29 14:18:46 quser1
　　drwx------        4096 2008/09/27 14:38:12 quser2
　　drwx------        4096 2008/11/14 06:10:19 test
　　drwx------        4096 2008/09/22 16:50:37 vbird1
　　drwx------        4096 2008/09/19 15:28:45 vbird2

　　后面的root@ip中，root是指定密码文件中的用户名，之后的::rhel4home这是rhel4home模块名
### B2、rsync客户端同步数据；

　　#rsync -avzP root@192.168.145.5::rhel4home rhel4home
　　Password: 这里要输入root的密码，是服务器端rsyncd.secrets提供的。在前面的例子中我们用的是mike，输入的密码并不回显，输好就回车。

　 　注： 这个命令的意思就是说，用root用户登录到服务器上，把rhel4home数据，同步到本地当前目录rhel4home上。当然本地的目录是可以你自己 定义的。如果当你在客户端上当前操作的目录下没有rhel4home这个目录时，系统会自动为你创建一个；当存在rhel4home这个目录中，你要注意 它的写权限。
```
　　#rsync -avzP  --delete linuxsir@linuxsir.org::rhel4home   rhel4home
```
　 　这回我们引入一个--delete 选项，表示客户端上的数据要与服务器端完全一致，如果 linuxsirhome目录中有服务器上不存在的文件，则删除。最终目的是让linuxsirhome目录上的数据完全与服务器上保持一致；用的时候要 小心点，最好不要把已经有重要数所据的目录，当做本地更新目录，否则会把你的数据全部删除；

　　設定 rsync client

　　设定密码文件
```
　　#rsync -avzP  --delete  --password-file=rsyncd.secrets   root@192.168.145.5::rhel4home rhel4home
```
　　这次我们加了一个选项 --password-file=rsyncd.secrets，这是当我们以root用户登录rsync服务器同步数据时，密码将读取rsyncd.secrets这个文件。这个文件内容只是root用户的密码。我们要如下做；

　　# touch rsyncd.secrets
　　# chmod 600 rsyncd.secrets
　　# echo "mike"> rsyncd.secrets

　　# rsync -avzP  --delete  --password-file=rsyncd.secrets   root@192.168.145.5::rhel4home rhel4home

　　注：这里需要注意的是这份密码文件权限属性要设得只有属主可读。

　　　　这样就不需要密码了；其实这是比较重要的，因为服务器通过crond 计划任务还是有必要的；
### B3、让rsync客户端自动与服务器同步数据

　 　服务器是重量级应用，所以数据的网络备份还是极为重要的。我们可以在生产型服务器上配置好rsync 服务器。我们可以把一台装有rysnc机器当做是备份服务器。让这台备份服务器，每天在早上4点开始同步服务器上的数据；并且每个备份都是完整备份。有时 硬盘坏掉，或者服务器数据被删除，完整备份还是相当重要的。这种备份相当于每天为服务器的数据做一个镜像，当生产型服务器发生事故时，我们可以轻松恢复数 据，能把数据损失降到最低；是不是这么回事？？

　　step1：创建同步脚本和密码文件
```　　
　　#mkdir   /etc/cron.daily.rsync
　　#cd  /etc/cron.daily.rsync 
　　#touch rhel4home.sh  rhel4opt.sh 
　　#chmod 755 /etc/cron.daily.rsync/*.sh  
　　#mkdir /etc/rsyncd/
　　#touch /etc/rsyncd/rsyncrhel4root.secrets
　　#touch /etc/rsyncd/rsyncrhel4easylife.secrets
　　#chmod 600  /etc/rsyncd/rsync.*
```
　 　注： 我们在 /etc/cron.daily/中创建了两个文件rhel4home.sh和rhel4opt.sh ，并且是权限是755的。创建了两个密码文件root用户用的是rsyncrhel4root.secrets ，easylife用户用的是 rsyncrhel4easylife.secrets，权限是600；

　　我们编辑rhel4home.sh，内容是如下的：
```
　　#!/bin/sh
　　#backup 192.168.145.5:/home 
　　/usr/bin/rsync   -avzP  --password-file=/etc/rsyncd/rsyncrhel4root.secrets    root@192.168.145.5::rhel4home   /home/rhel4homebak/$(date +'%m-%d-%y')
```
　　我们编辑 rhel4opt.sh ，内容是：
```
　　#!/bin/sh
　　#backup 192.168.145.5:/opt 
　　/usr/bin/rsync   -avzP  --password-file=/etc/rsyncd/rsyncrhel4easylife.secrets    easylife@192.168.145.5::rhel4opt   /home/rhel4hoptbak/$(date +'%m-%d-%y')
```
　　注：你可以把rhel4home.sh和rhel4opt.sh的内容合并到一个文件中，比如都写到rhel4bak.sh中；

　　接着我们修改 /etc/rsyncd/rsyncrhel4root.secrets和rsyncrhel4easylife.secrets的内容；
```
　　# echo "mike" > /etc/rsyncd/rsyncrhel4root.secrets
　　# echo "keer"> /etc/rsyncd/rsyncrhel4easylife.secrets
```
　 　然后我们再/home目录下创建rhel4homebak 和rhel4optbak两个目录，意思是服务器端的rhel4home数据同步到备份服务器上的/home/rhel4homebak 下，rhel4opt数据同步到 /home/rhel4optbak/目录下。并按年月日归档创建目录；每天备份都存档；
```
　　#mkdir /home/rhel4homebak
　　#mkdir /home/rhel4optbak
```
　　step2：修改crond服务器的配置文件 加入到计划任务
```
　　#crontab  -e
```
　　加入下面的内容：

　　# Run daily cron jobs at 4:10 every day  backup rhel4 data:  
　　10 4 * * * /usr/bin/run-parts   /etc/cron.daily.rsync   1> /dev/null

　　注：第一行是注释，是说明内容，这样能自己记住。
　　　　第二行表示在每天早上4点10分的时候，运行 /etc/cron.daily.rsync 下的可执行脚本任务；
　　　　
　　配置好后，要重启crond 服务器；
```
　　# killall crond    注：杀死crond 服务器的进程；
　　# ps aux |grep crond  注：查看一下是否被杀死；
　　# /usr/sbin/crond    注：启动 crond 服务器；
　　# ps aux  |grep crond  注：查看一下是否启动了？
　　root      3815  0.0  0.0   1860   664 ?        S    14:44   0:00 /usr/sbin/crond
　　root      3819  0.0  0.0   2188   808 pts/1    S+   14:45   0:00 grep crond
```
