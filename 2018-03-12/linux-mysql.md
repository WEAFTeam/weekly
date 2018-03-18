---
title: MySQL主从数据库的设置与Xtrabackup备份InnoDB(MySQL)
description: MySQL主从数据库的设置与Xtrabackup备份InnoDB(MySQL)
tags:
  - Linux运维
author: 
  - earth
thumbnail: 'http://us-forever.com/img/mysql-linux-icon.png'
abbrlink: 2f5dded6
date: 2018-03-17 16:08:56
---
一、准备环境
-----------
1. 两台服务器：服务器A、服务器B
2. 服务器A：Red Hat Enterprise Linux Server release 6.5 (Santiago)
3. 服务器B：Red Hat Enterprise Linux Server release 6.5 (Santiago)
4. 服务器A IP：172.16.125.50
5. 服务器B IP：172.16.125.52
6. MySQL版本：5.6.23

二、安装MySQL
------------
具体安装请见

1. [LinuxMySQL的安装(1)](http://us-forever.com/2018/01/15/LinuxMySQL的安装/)
2. [LinuxMySQL的安装(2)](http://us-forever.com/2018/02/02/LinuxMySQL的安装-2/)
3. [LinuxMySQL的安装(3)](http://us-forever.com/2018/02/08/LinuxMySQL的安装-3/)

<!--more-->
三、主从库配置
-----
### 1、主库在/etc/my.cnf里添加以下内容
``` ini
#log日志
log_bin=mysql_bin
#server ID
server_id=2
#忽略同步的库
binlog-ignore-db=information_schema
binlog-ignore-db=cluster
binlog-ignore-db=mysql
#需要同步的库
binlog-do-db=test
```
### 2、从库在/etc/my.cnf里添加以下内容
``` xml
log_bin=mysql_bin
server_id=3
binlog-ignore-db=information_schema
binlog-ignore-db=cluster
binlog-ignore-db=mysql
replicate-do-db=ufind_db
replicate-ignore-db=mysql
log-slave-updates
slave-skip-errors=all
slave-net-timeout=60
```
四、主从库设置
------
### 1、进入主库，我们在主库中创建一个的账户，从库通过使用这个账号来同步数据。
``` sql
CREATE USER 'repl'@'172.16.125.52' IDENTIFIED BY '123456';
```
### 2、赋予相应的权限
``` sql
GRANT FILE ON *.* TO 'repl'@'172.16.125.52' IDENTIFIED BY '123456';

GRANT REPLICATION SLAVE ON *.* TO 'repl'@'172.16.125.52' IDENTIFIED BY '123456';

FLUSH PRIVILEGES;
```
### 3、重启数据库（主库）执行以下命令
```
SHOW MASTER STATUS;
```
![](http://us-forever.com/img/mysqlsync.png)
要记住以上的信息，在设置从库的时候需要填写并设置。
### 4、在从库里边执行以下命令
``` script
stop slave;
change master to master_host='172.16.125.50',master_user='repl',master_password='123456',master_log_file='mysql_bin.000023', master_log_pos=120;
start slave;
```
### 5、然后执行一下命令查看状态
``` shell
show slave status \G;
```
内容如下：
![](http://us-forever.com/img/mysqlsync1.png)
![](http://us-forever.com/img/mysqlsync2.png)
### 6、测试与提示
后期的测试中我们只针对**test**库进行了同步。
所以只能针对**test**进行的操作才有效。

如果后期对一些列库进行操作，需要 添加相应的配置
``` shell	
	#主库配置文件
	binlog-do-db=test
	#从库配置文件
	replicate-do-db=test
```
并查询出最新的master的状态，停止从库。并改变从库的配置重启同步。
五、Xtrabackup的简单介绍
-------------------
Percona XtraBackup 是世界上唯一的开源免费的MySQL热备份软件，可以执行非阻塞操作
InnoDB和XtraDB数据库的备份。 Percona XtraBackup可提供以下优点：

- 备份快速安全可靠
- 备份期间不间断的事务处理
- 节省磁盘空间和网络带宽
- 自动备份验证
- 更快的恢复时间保证正常工作

Percona XtraBackup 为所有版本的Percona服务器，MySQL和MariaDB提供MySQL热备份。 它可执行
流媒体，压缩和增量MySQL备份。

六、Xtrabackup的安装
--------------
如果在互联网下 可使用以下命令安装
``` shell
	wget https://www.percona.com/downloads/XtraBackup/Percona-XtraBackup-2.4.4/binary/redhat/7/x86_64/percona-xtrabackup-24-2.4.4-1.el7.x86_64.rpm
```
获取相应rpm包
安装部分依赖(不同的操作系统可能已安装的库不尽相同)
``` shell
	rpm -ivh mysql-community-libs-compat-5.7.20-1.el7.x86_64.rpm
	#根据mysql版本而定
	yum list|grep perl
	yum -y install perl-DBI.x86_64 perl-DBD-MySQL.x86_64
```
然后安装Xtrabackup
```shell
	rpm -ivh percona-xtrabackup-24-2.4.4-1.el7.x86_64.rpm
```
参考：
```shell
	yum install cmake gcc gcc-c++ libaio libaio-devel automake autoconf bison libtool ncurses-devel libgcrypt-devel libev-devel libcurl-devel vim-common
```
七、Xtrabackup备份MySQL
----------------
``` shell
	xtrabackup --defaults-file=/etc/my.cnf --user=root --password=root --host=localhost --backup --target-dir=/data/backups/
	可指定数据库--databases=test
```
八、Xtrabackup的备份恢复
---------
备份之前必须先关闭MySQL server
然后删除data目录（/var/lib/mysql一般情况是这个）
``` shell
	xtrabackup  --copy-back --target-dir=/data/backups/
```
执行完恢复之后需要设置文件权限
``` shell
	chown -R mysql:mysql /var/lib/mysql
```
然后启动mysql
``` shell
	systemctl start mysqld.service
	#或者使用服务
	service mysqld start
```
九、使用脚本自动备份7天之内的数据
--------------
```xml
#!/bin/sh

# Database info
DB_USER="root"
DB_PASS="root"
DB_HOST="localhost"

# Others vars
BCK_DIR="/opt/app/mysqlbackup"    #the backup file directory
CONF_DIR="/etc/my.cnf"
DATE=`date +%F`
RMDATE=`date -d '-7 day' +%F`

# TODO

mkdir -p $BCK_DIR/$DATE/
#Create dir for save backup data
xtrabackup --defaults-file=$CONF_DIR --user=$DB_USER --password=$DB_PASS --host=$DB_HOST --backup --target-dir=$BCK_DIR/$DATE/
#Backup mysql data
rm -rf $BCK_DIR/$RMDATE
#Delete the backup 7 days ago
#热备份数据库
```
加入crontab
``` xml
	30 2 * * * /bin/sh /home/scripts/mysqlbackup.sh
```

[更多请参考官方文档](https://learn.percona.com/hubfs/Manuals/Percona_Xtra_Backup/Percona_XtraBackup_2.4/Percona-XtraBackup-2.4.9.pdf)


