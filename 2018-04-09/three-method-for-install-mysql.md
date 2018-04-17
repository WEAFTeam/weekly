---
title: Linux下MySQL安装
description: Linux下MySQL安装 
tags:
  - Linux运维
author:
  - earth
thumbnail: 'https://labs.mysql.com/common/logos/mysql-logo.svg?v2'
category: Linux
date: 2018-04-17 12:07:12
---
接下来我将介绍3种方法安装MySQL

第一种
=====

一、查看是否安装了MySQL
-------
使用命令：
``` xml
rpm -qa|grep -i mysql
```
如果使用centos，可能会出现冲突，解决冲突需要卸载mariadb
首先查看是否安装了Mariadb
``` xml
rpm -qa|grep mariadb
```
<!--more-->
然后卸载
``` xml
rpm -e mariadb-libs-5.5.56-2.el7.x86_64
```
强制卸载(可选):
``` xml
rpm -e --nodeps mariadb-libs-5.5.56-2.el7.x86_64
```
![mysql-1](http://us-forever.com/img/linuxMySQL-1.png)
二、如果安装了需删除已安装版本
------
删除命令：
``` xml
rpm -e --nodeps 包名
( rpm -ev mysql-4.1.12-3.RHEL4.1 )
```
删除老版本mysql的开发头文件和库
命令：
```
rm -fr /usr/lib/mysql
rm -fr /usr/include/mysql
```
注意：卸载后/var/lib/mysql中的数据及/etc/my.cnf不会删除，如果确定没用后就手工删除
```
rm -f /etc/my.cnf
rm -fr /var/lib/mysql
```
三、安装mysql准备环境
------
我自mysql官网下载通用的Linux版本安装包
mysql-5.7.20-linux-glibc2.12-x86_64.tar.gz
将下载好的包放在 /usr/local 目录下，或者执行命令：
```
cd /usr/local
wget https://cdn.mysql.com//Downloads/MySQL-5.7/mysql-5.7.20-linux-glibc2.12-x86_64.tar.gz
```
![mysql-2](http://us-forever.com/img/linuxMySQL-2.png)

解压下载的文件
```
tar -zxvf mysql-5.7.20-linux-glibc2.12-x86_64.tar.gz
```
将解压之后的所有文件移动到/usr/local/mysql
```
mv ./mysql-5.7.20-linux-glibc2.12-x86_64/* ./mysql
```
为mysql创建系统用户(可选，新版本会自动创建相应用户)
```
groupadd mysql
useradd -r -g mysql mysql
```
//-r参数表示mysql用户是系统用户，不可用于登录系统
并变更mysql安装目录的所属用户和用户组
```
chown -R mysql:mysql mysql
// -R 迭代处理
```
![mysql-3](http://us-forever.com/img/linuxMySQL-3.png)
四、 安装和初始化
------

初始化数据库
```
./bin/mysqld --initialize --user=mysql --basedir=/usr/local/mysql --datadir=/usr/local/mysql/data --lc_messages_dir=/usr/local/mysql/share --lc_messages=en_US
```
记录刚刚输出的密码：
	jk9tEao<94MC
![mysql-4](http://us-forever.com/img/linuxMySQL-4.png)

配置/etc/my.cnf 
	
[my.cnf](http://us-forever.com/file/my.cnf)

五、 启动登录和设置密码
------
切换到mysql安装目录的bin下启动
```
./mysqld_safe --user=mysql
```
启动后可能无法使用当前窗口

登录进去设置新的密码：
```
./mysql -u root -p
```
``` sql
set password=password("root");
flush privileges;
```
![mysql-5](http://us-forever.com/img/linuxMySQL-5.png)

六、 添加到服务

切换到 support-files目录下，并执行以下命令
```
cp mysql.server /etc/init.d/mysql
```
然后停止当前进程，使用服务启动mysql
```
service mysql start
```

并添加mysql环境变量
在 /etc/profile 的文件末尾追加：
	export PATH=$PATH:/usr/local/mysql/bin
保存后执行
```
source /etc/profile
```
最后使用新密码登录到mysql
![mysql-6](http://us-forever.com/img/linuxMySQL-6.png)

第二种
======

一、查看是否安装了MySQL
-------
使用命令：
``` xml
rpm -qa|grep -i mysql
```
如果使用centos，可能会出现冲突，解决冲突需要卸载mariadb
首先查看是否安装了Mariadb
``` xml
rpm -qa|grep mariadb
```
<!--more-->
然后卸载
``` xml
rpm -e mariadb-libs-5.5.56-2.el7.x86_64
```
强制卸载(可选):
``` xml
rpm -e --nodeps mariadb-libs-5.5.56-2.el7.x86_64
```
![mysql-1](http://us-forever.com/img/linuxMySQL-1.png)
二、如果安装了需删除已安装版本
------
删除命令：
``` xml
rpm -e --nodeps 包名
( rpm -ev mysql-4.1.12-3.RHEL4.1 )
```
删除老版本mysql的开发头文件和库
命令：
```
rm -fr /usr/lib/mysql
rm -fr /usr/include/mysql
```
注意：卸载后/var/lib/mysql中的数据及/etc/my.cnf不会删除，如果确定没用后就手工删除
```
rm -f /etc/my.cnf
rm -fr /var/lib/mysql
```
三、准备安装的环境
wget https://dev.mysql.com/get/Downloads/MySQL-5.7/mysql-5.7.20-1.el7.x86_64.rpm-bundle.tar
或者用自己准备好的包
mysql-5.7.20-1.el7.x86_64.rpm-bundle.tar
解压包
``` 
tar -xvf mysql-5.7.20-1.el7.x86_64.rpm-bundle.tar
```
然后按照以下顺序安装
```
rpm -ivh mysql-community-common-5.7.20-1.el7.x86_64.rpm 
rpm -ivh mysql-community-libs-5.7.20-1.el7.x86_64.rpm 
rpm -ivh mysql-community-client-5.7.20-1.el7.x86_64.rpm 
rpm -ivh mysql-community-server-5.7.20-1.el7.x86_64.rpm
```
四、启动并修改密码
安装完成后就可以启动服务了
```
service mysqld start
```
启动后查看配置文件
```
vi /etc/my.cnf
```
![linuxMySQL-1-2.png](http://us-forever.com/img/linuxMySQL-1-2.png)
找打log文件 进入查找默认root密码
```
cat /var/log/mysqld.log
```
![](http://us-forever.com/img/linuxMySQL-1-3.png)

使用一下命令登录并修改密码
```
mysql -uroot -p
```
修改密码
```
SET PASSWORD FOR 'root'@'localhost' = PASSWORD('newpass');
FLUSH PRIVILEGES;
```
![](http://us-forever.com/img/linuxMySQL-1-4.png)

第三种
======

一、查看是否安装了MySQL数据库
------
```
rpm -qa|grep mysql
```
![](http://us-forever.com/img/mysql1.png)
卸载
```
rpm -e --nodeps mysql-libs-5.1.71-1.el6.x86_64
```
二、安装
安装一下包
```
rpm -ivh MySQL-devel-5.6.23-1.linux_glibc2.5.x86_64.rpm
rpm -ivh MySQL-client-5.6.23-1.linux_glibc2.5.x86_64.rpm
rpm -ivh MySQL-server-5.6.23-1.linux_glibc2.5.x86_64.rpm
rpm -ivh MySQL-embedded-5.6.23-1.linux_glibc2.5.x86_64.rpm
rpm -ivh MySQL-shared-5.6.23-1.linux_glibc2.5.x86_64.rpm
rpm -ivh MySQL-shared-compat-5.6.23-1.linux_glibc2.5.x86_64.rpm
```
三、启动登录设置密码
使用以下命令开启服务
	
	service mysql start

获取初始密码：
![](http://us-forever.com/img/mysql2.png)
使用root登录
	
	mysql -uroot -p

然后试用一下命令设置密码

	SET PASSWORD = PASSWORD('123456');