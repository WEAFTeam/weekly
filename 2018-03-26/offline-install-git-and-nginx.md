---
title: Nginx和Git的离线安装
description: Nginx和Git的离线安装
tags:
  - Linux运维
author:
  - earth
thumbnail: 'http://nginx.org/nginx.png'
category: Linux
abbrlink: a86291b8
date: 2018-04-01 12:07:12
---
一、准备工作
========
一般情况下为了确保安装没有任何问题，我们先使用有网络环境的下安装的方法，去检测当前机器具体需要安装什么冬天链接库，然后按照提示缺失的库去下载相应的库。
我们按照正常的流程，去解压nginx
```
	tar -zxvf nginx-1.13.8.tar.gz
```
进入解压后的目录执行
```
	cd nginx-1.13.8
	./configure
```
出现以下错误：
![](http://us-forever.com/img/nginxerror.png)

我们按照有网络环境的方法去检测缺失的库及其版本。
```
	yum -y install gcc gcc-c++ autoconf automake make
```
<!--more-->
显示如下：
![](http://us-forever.com/img/liberror.png)

我们下载号相应的库
下边提供几个下载的网址：

- [http://mirrors.163.com/centos/6/os/x86_64/Packages/](http://mirrors.163.com/centos/6/os/x86_64/Packages/)
- [http://rpmfind.net/](http://rpmfind.net/)
- [https://pkgs.org](https://pkgs.org)

下边是下载好的库
![](http://us-forever.com/img/gcclib.png)
安装相应的库（**集体安装情况具体分析**）：
```
	rpm -ivh mpfr-2.4.1-6.el6.x86_64.rpm
	rpm -ivh cpp-4.4.7-18.el6.x86_64.rpm
	rpm -Uvh tzdata-2016j-1.el6.noarch.rpm
	rpm -Uvh glibc-common-2.12-1.209.el6.x86_64.rpm glibc-2.12-1.209.el6.x86_64.rpm glibc-headers-2.12-1.209.el6.x86_64.rpm glibc-devel-2.12-1.209.el6.x86_64.rpm kernel-headers-2.6.32-696.el6.x86_64.rpm
	rpm -ivh libgomp-4.4.7-18.el6.x86_64.rpm
	rpm -Uvh libstdc++-4.4.7-18.el6.x86_64.rpm
	rpm -ivh libstdc++-devel-4.4.7-18.el6.x86_64.rpm
	rpm -ivh ppl-0.10.2-11.el6.x86_64.rpm
	rpm -ivh cloog-ppl-0.15.7-1.2.el6.x86_64.rpm
	rpm -Uvh libgcc-4.4.7-18.el6.x86_64.rpm
	rpm -ivh gcc-4.4.7-18.el6.x86_64.rpm
	rpm -ivh gcc-c++-4.4.7-18.el6.x86_64.rpm
	rpm -ivh automake-1.11.1-4.el6.noarch.rpm autoconf-2.63-5.1.el6.noarch.rpm
```
然后执行./configure
发现错误：
![](http://us-forever.com/img/nginxerror1.png)
	然后安装一下库
```
	rpm -Uvh pcre-7.8-7.el6.x86_64.rpm
	rpm -ivh pcre-devel-7.8-7.el6.x86_64.rpm
	rpm -Uvh zlib-1.2.3-29.el6.x86_64.rpm
	rpm -ivh zlib-devel-1.2.3-29.el6.x86_64.rpm
	rpm -i --force --nodeps krb5-devel-1.10.3-65.el6.x86_64.rpm
	rpm -Uvh openssl-1.0.1e-57.el6.x86_64.rpm
	rpm -ivh openssl-devel-1.0.1e-57.el6.x86_64.rpm
```
然后执行
```
	./configure
	make
	make install
```
安装完成
二、查看版本信息
=====
根据安装完成的信息查看nginx.
三、简介
-----
不同操作系统的Linux的安装可能不太一样。
本教程使用的是CentOS或者RHEL。
四、准备环境
----
如果有网络的情况下肯定相当容易：
```
yum install curl-devel expat-devel gettext-devel openssl-devel zlib-devel gcc perl-ExtUtils-MakeMaker

yum install git

```
若果yum源安装的版本较低，不执行**yum install git**命令，并按照四、五步骤操作。
如果是离线需要完成后续操作：
首先我们需要到官网下载相应的安装包
[https://www.kernel.org/pub/software/scm/git/](https://www.kernel.org/pub/software/scm/git/)
选择**tar.gz**
三、下载和安装依赖
------
然后最麻烦的地方就是依赖动态链接库的下载。
需要下载的库可以到这两个网站上去找：
[fr2.rpmfind.net](http://www.rpmfind.net/linux/RPM/index.html)
[Linux Packages Search-pkgs.org](https://pkgs.org/)
我这里提供了一些
<!--more-->
```
cpio-2.11-24.el7.x86_64.rpm
curl-7.29.0-42.el7.x86_64.rpm
expat-2.1.0-10.el7_3.x86_64.rpm
expat-devel-2.1.0-10.el7_3.x86_64.rpm
gdbm-devel-1.10-8.el7.x86_64.rpm
gettext-0.19.8.1-2.el7.x86_64.rpm
gettext-devel-0.19.8.1-2.el7.x86_64.rpm
krb5-devel-1.15.1-8.el7.x86_64.rpm
libcurl-7.29.0-42.el7.x86_64.rpm
libcurl-devel-7.29.0-42.el7.x86_64.rpm
libdb-devel-5.3.21-20.el7.x86_64.rpm
openssl-1.0.2k-8.el7.x86_64.rpm
openssl-devel-1.0.2k-8.el7.x86_64.rpm
perl-5.16.3-292.el7.x86_64.rpm
perl-devel-5.16.3-292.el7.x86_64.rpm
perl-ExtUtils-CBuilder-0.28.2.6-292.el7.noarch.rpm
perl-ExtUtils-Install-1.58-292.el7.noarch.rpm
perl-ExtUtils-MakeMaker-6.68-3.el7.noarch.rpm
perl-ExtUtils-Manifest-1.61-244.el7.noarch.rpm
perl-ExtUtils-ParseXS-3.18-3.el7.noarch.rpm
systemtap-sdt-devel-3.1-3.el7.x86_64.rpm
zlib-1.2.7-17.el7.x86_64.rpm
zlib-devel-1.2.7-17.el7.x86_64.rpm
```
这是git需要的一些库，需要安装的不是很多，但是安装的库也需要依赖。
```
rpm -ivh perl-5.16.3-292.el7.x86_64.rpm
rpm -ivh perl-devel-5.16.3-292.el7.x86_64.rpm
rpm -ivh zlib-devel-1.2.7-17.el7.x86_64.rpm
rpm -ivh libcurl-devel-7.29.0-42.el7.x86_64.rpm
rpm -ivh curl-7.29.0-42.el7.x86_64.rpm
rpm -ivh zlib-devel-1.2.7-17.el7.x86_64.rpm
rpm -ivh openssl-devel-1.0.2k-8.el7.x86_64.rpm
rpm -ivh perl-ExtUtils-MakeMaker-6.68-3.el7.noarch.rpm
rpm -ivh gettext-devel-0.19.8.1-2.el7.x86_64.rpm
```
以上库需要安装，并需要安装对应依赖。
有时候有些包可能互相依赖，安装时可使用一下命令
```
rpm -ivh perl-ExtUtils-MakeMaker-6.68-3.el7.noarch.rpm perl-ExtUtils-Install-1.58-292.el7.noarch.rpm zlib-devel-1.2.7-17.el7.x86_64.rpm
```
编译安装可能需要的包
```
rpm -ivh cloog-ppl-0.15.7-1.2.el6.x86_64.rpm
rpm -ivh cpp-4.4.7-18.el6.x86_64.rpm
rpm -ivh gcc-4.4.7-18.el6.x86_64.rpm
rpm -ivh gcc-c++-4.4.7-18.el6.x86_64.rpm
rpm -ivh libgcc-4.4.7-18.el6.x86_64.rpm
rpm -ivh libgomp-4.4.7-18.el6.x86_64.rpm
rpm -ivh libstdc++-4.4.7-18.el6.x86_64.rpm
rpm -ivh libstdc++-devel-4.4.7-18.el6.x86_64.rpm
rpm -ivh mpfr-2.4.1-6.el6.x86_64.rpm
rpm -ivh ppl-0.10.2-11.el6.x86_64.rpm
```
有时会发生冲突可以使用枪支卸载，或者不考虑依赖安装。
```
rpm -e  --nodeps mariadb-libs-5.5.56-2.el7.x86_64
	//强制卸载

rpm -i --force --nodeps krb5-devel-1.15.1-8.el7.x86_64.rpm
	//强制安装 --force可选
```
五、解压和安装
------
解压安装包
```
tar -zxvf git-2.15.1.tar.gz
```
进入解压后的文件夹
![](http://us-forever.com/img/git-1.png)
执行一下命令
```
./configure 
```
检查没有任何出错
![](http://us-forever.com/img/git-2.png)
然后执行以下命令进行编译
```
make
```
检查没有问题执行安装
```
make install
```
检查没有任何出错
![](http://us-forever.com/img/git-4.png)
六、查看安装结果
----

``` xml
git --version
```

![](http://us-forever.com/img/git-5.png)
