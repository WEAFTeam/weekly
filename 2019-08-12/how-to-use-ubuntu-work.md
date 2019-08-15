---
title: 用Ubuntu搭建工作环境
description: 用Ubuntu搭建工作环境
tags:
  - Linux
author:
  - earth
thumbnail: 'https://songyaxu.oss-cn-beijing.aliyuncs.com/blog/ubuntu.png'
category: Linux
date: '2019-08-015 10:34:45'
---
用Ubuntu搭建工作环境
=========

一、Ubuntu简介
--------

### 什么是Ubuntu?

Ubuntu（又称乌班图）是一个以桌面应用为主的开源GNU/Linux操作系统，Ubuntu 是基于Debian GNU/Linux，支持x86、amd64（即x64）、ARM和ppc架构，由全球化的专业开发团队（Canonical Ltd）打造的。
Ubuntu基于Debian发行版和GNOME桌面环境，而从11.04版起，Ubuntu发行版放弃了Gnome桌面环境，改为Unity，与Debian的不同在于它每6个月会发布一个新版本。Ubuntu的目标在于为一般用户提供一个最新的、同时又相当稳定的主要由自由软件构建而成的操作系统。Ubuntu具有庞大的社区力量，用户可以方便地从社区获得帮助。Ubuntu对GNU/Linux的普及特别是桌面普及作出了巨大贡献，由此使更多人共享开源的成果与精彩.

### 为什么是Ubuntu?

- ubuntu基于linux的免费开源桌面PC操作系统，十分契合英特尔的超极本定位，支持x86、64位和ppc架构。
- ubuntu作为经典桌面系统之一，并且它的gnome桌面系统用起来还是很舒服的，而且也是比较漂亮的。
- ubuntu社区比较强大，有自己的应用商店，可安装的软件也非常丰富。
- 相对的教程也比较多，而且市场上几种相对比较好的桌面系统几乎都是和ubuntu有一点的关系。
- 其次，笔者比较熟悉ubuntu,而且在大学时就已经开始使用ubuntu了。

二、安装ubuntu
-----------

这边我们首先到[ubuntu download:https://ubuntu.com/download/desktop](https://ubuntu.com/download/desktop)页面选择我们要安装的版本并下载。
我这里没有没有选用最新的19.04的版本(一般对ubuntu而言，双数版本是稳定的版本)，这里其实选择哪个版本都无所谓，我这里选择18.04.3 LTS进行下载。

我这里是win10的系统，我选择了安装双系统。
下载之后我们将我的系统盘压缩出一个单独未使用的盘，供我们安装ubuntu。安装的时候直接选用那个作为系统盘就可以了。另外这边ubuntu有自己的bios所以直接安装就可以。

安装过程中我们需要选择一些语言，和一些开发包什么的，依据自己的需求吧。这边记得选择图形界面。
还有就是用户设置，我们可以单独设置一个用户作为我们平时使用登录的用户。

这里我就不在多余赘述具体的安装过程了，后边也许会成具体的安装教程（看自己的时间），不过笔者会在最后给出一些其他的教程供大家参考。

三、环境的配置
------

这里应该是本文的重点。

我们平时工作需要的软件有很多.....

### 配置源

这边在国内为了更好的使用ubuntu，我们需要配置清华的源。
- 清华源[https://mirrors.tuna.tsinghua.edu.cn/help/ubuntu/](https://mirrors.tuna.tsinghua.edu.cn/help/ubuntu/)
- 阿里云源[https://opsx.alibaba.com/guide?lang=zh-CN&document=69a2341e-801e-11e8-8b5a-00163e04cdbb](https://opsx.alibaba.com/guide?lang=zh-CN&document=69a2341e-801e-11e8-8b5a-00163e04cdbb)

这边我们只需要在原基础上加新的源就ok(这边我加了两个)
``` shell
cd /etc/apt/sources.list.d/
sudo vim aliyun.list
sudo vim tsinghua.list
sudo apt-get update
```

我们可以通过安装一个小动画来测试安装结果：
``` shell
sudo apt-get install sl
sl
```
![ubuntu-sl](https://songyaxu.oss-cn-beijing.aliyuncs.com/blog/ubuntu-sl.gif)

### 安装JDK

这边如果在安装ubuntu的时候选择了，可能会有自带的。我们可以通过
``` shell
java -version
```
查看
另外如需安装这边我们选择简单的方式直接安装openjdk8
``` shell
sudo apt-get install openjdk-8-jdk
java -version
```
这边最高的版本应该是11
``` shell
sudo apt-get install openjdk-11-jdk
java -version
```
这边还可以安装一些其他的工具如：visualvm等
``` shell
sudo apt-get install visualvm
```
另外这边jps如果不好使的话，需要添加环境变量来解决。

### 安装编译器

- Eclipse[https://www.eclipse.org/downloads/packages/](https://www.eclipse.org/downloads/packages/)
- IntelliJ IDEA[http://www.jetbrains.com/idea/download/#section=linux](http://www.jetbrains.com/idea/download/#section=linux)

这两款安装起来比较方便，下载对应linux的包，然后在指定位置解压。直接运行就可以了。
而且会自动创建图标，我们通过右键**添加到收藏夹**，以后就可以直接点击图标进入了。
另外其他语言的编译器如WebStorm、CLion安装都很方便。

``` shell
cd /home/songyaxu/下载
tar -zxvf ideaIU-2019.2.tar.gz
cd idea-IU-192.5728.98/bin/
./idea.sh 
```

``` shell
cd /home/songyaxu/下载
tar -zxvf eclipse-jee-2019-06-R-linux-gtk-x86_64.tar.gz
cd 
```

### 安装git
``` shell
sudo apt-get install git
git --version
```
### 安装搜狗输入法

搜狗输入法是基于fcitx的，所以这边需要先安装fcitx。
``` shell
sudo add-apt-repository ppa:fcitx-team/nightly
sudo apt-get update
sudo apt-get install fcitx
sudo apt-get install fcitx-config-gtk
sudo apt-get install fcitx-table-all
sudo apt-get install im-switch
```

然后直接去官网[https://pinyin.sogou.com/linux/?r=pinyin](https://pinyin.sogou.com/linux/?r=pinyin)
下载deb安装包后直接运行。

### 安装SublimeText

这边高版本ubuntu直接到**Ubuntu软件**这个软件中搜索sublime直接安装即可。
这边sublime-text3可能会导致无法输入中文，
参照以下方案
``` shell
git clone https://github.com/lyfeyaj/sublime-text-imfix.git
cd sublime-text-imfix
sudo ./sublime-imfix
```
然后显示修复成功。重启就可以输入中文了。 另外这边我们还可以通过**subl filename.txt**来直接使用sublime-text3

### 安装chrome

这边高版本的ubuntu可以直接在**Ubuntu软件**直接搜索Chromium来下载安装chrome,但是这个产品和chrome略有区别。

也可以去官网下载最新版本

- Google下载 [https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb](https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb)
- 百度云下载 提取码: ue4g [https://pan.baidu.com/s/1XTsqfZR1F26JZ2TNH7hKxw](https://pan.baidu.com/s/1XTsqfZR1F26JZ2TNH7hKxw)

然后双击安装

### deepin wine安装

deepin wine是wine的深度定制版本，那么wine又是什么呢？
wine是一个能在多种操作系统上运行windows应用的兼容层应用。也就是安装了wine之后我们就能在linux上安装windows的软件了。
具体安装
``` shell
git clone https://github.com/wszqkzqk/deepin-wine-ubuntu.git
cd deepin-wine-for-ubuntu
sudo ./install.sh
```

### 安装微信
微信肯定是必不可少的，但是如果想安装微信可能需要费点功夫。
当前安装基于deepin wine,我们在阿里云镜像下载中心下载微信的包。然后点击安装就可以了。

[deepin wine wechat](https://mirrors.aliyun.com/deepin/pool/non-free/d/deepin.com.wechat/)

### 安装Foxmail
当前安装基于deepin wine,我们在阿里云镜像下载中心下载Foxmail的包。然后点击安装就可以了。

[deepin wine foxmail](http://mirrors.aliyun.com/deepin/pool/non-free/d/deepin.com.foxmail/)

### 安装QQ
当前安装基于deepin wine,我们在阿里云镜像下载中心下载QQ的包。然后点击安装就可以了。

[deepin wine qq](https://mirrors.aliyun.com/deepin/pool/non-free/d/deepinwine-qq/)


### 安装钉钉
在阿里云镜像中心下载相应安装包，点击安装就可以了。

[dingtalk](http://mirrors.aliyun.com/deepin/pool/non-free/d/dingtalk/)


### 安装企业微信
当前安装基于deepin wine,我们在阿里云镜像下载中心下载企业微信的包。然后点击安装就可以了。

[deepin wine wxwork](http://mirrors.aliyun.com/deepin/pool/non-free/d/deepin.com.weixin.work/)

### 安装网易云音乐 

网易云音乐还是很良心的，这边官网直接支持Ubuntu18.04安装包的下载。

[网易云音乐](https://music.163.com/#/download)后双击安装即可。

### 安装FTP工具

这边可以选择使用在带的Remmina远程桌面客户端,带有RDP、SFTP、SSH、VNC等协议的连接。
或者我们这边可以下载ilezilla
``` shell
sudo apt-get install filezilla
```

### 安装Postman
Postman我们直接可以在官网上下载linux版本进行安装，简单方便。

[Postman](https://www.getpostman.com/downloads/)

###安装pdf阅读器

福昕pdf阅读器是非常好的一个阅读器，我们可以再官网下载并安装。

[foxit](https://www.foxitsoftware.cn/)
### 安装navicat
这边直接安装了Navicat Premium 11版本，然后根据破解方法，自写了以下脚本
``` sh
#!/bin/sh
echo "start_navicat"
echo "delete user.reg"
cd /home/songyaxu/.navicat64/
rm user.reg
echo "delete successful"
/usr/share/navicat/start_navicat
``` 
放在navicat的安装目录** /usr/share/navicat ** 下（其中**user.reg**需找到自己相对用户的文件夹下）
然后找到快速启动图标文件将内容改成以下内容。
``` desktop
[Desktop Entry]
Version=1.0
Type=Application
Terminal=false
Name=Navicat
Exec=/usr/share/navicat/navicatstart.sh
Icon=navicat
Categories=Development;
```
主要改了**Exec**项
以上文件在**/usr/share/applications/navicat.desktop**

四、其他配置
----------

### 配置快速启动
有时候一些应用没有快速启动方式，每次启动都需要使用控制台，并留下一个无法关闭的窗口，
我们可以通过一下方式配置快速启动图标。
``` shell
cd /usr/share/applications
```
切换到**/usr/share/applications**目录，我们这里以postman为例，创建一个叫postman.desktop的文件，并输入一下内容。
``` desktop
[Desktop Entry]

Encoding=UTF-8

Name=Postman

Exec=/app/Postman/app/Postman

Icon=/app/Postman/app/resources/app/assets/icon.png

Terminal=false

Type=Application

Categories=Development;
```
点击显示应用程序就可以看到我们设置的快速启动图标的应用程序了。

### 快速截图
使用**shift+PrtScr**快捷键创建快速选取截屏，非常好用哦。保存的文件放在当前用户下的**图片**文件夹里边

### 显示微信等应用托盘

安装TopIconPlus的gnome-shell扩展。
``` shell
sudo apt-get install gnome-shell-extension-top-icons-plus gnome-tweaks
```
然后用r命令重启gnome-shell，最后用gnome-tweaks开启这个扩展。

### 乱码问题

我们首先拷贝windows的所有字体到**usr/local/share/fonts**下，然后使用以下命令更新我们系统的字体。
``` shell
fc-cache -fv
```
其他乱码可以参考以下文章
[ubuntu 18.04 下 wine 中文无法正常显示的解决方案](https://blog.abreto.net/archives/2018/05/ubuntu-18-04-wine-chinese-problem-solution.html)

### 其他软件的安装
可以到一下地址去寻找安装包

[阿里云镜像下载中心](https://mirrors.aliyun.com/deepin/pool/non-free/)

五、参考文档
----------

1. [工作环境换成Ubuntu18.04小记](https://www.cnblogs.com/dunitian/p/9773214.html)
2. [Ubuntu常用软件安装（小集合）](https://www.cnblogs.com/dunitian/p/6670560.html)
3. [2019年wine QQ最完美解决方案](https://www.lulinux.com/archives/1319)
4. [ubuntu 18.04 下 wine 中文无法正常显示的解决方案](https://blog.abreto.net/archives/2018/05/ubuntu-18-04-wine-chinese-problem-solution.html)
5. [记 Win10 + Ubuntu18.04 安装](https://www.cnblogs.com/tanrong/p/9166595.html)


