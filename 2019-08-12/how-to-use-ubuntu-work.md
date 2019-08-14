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

### 安装钉钉

### 安装企业微信

### 安装网易云音乐 

网易云音乐还是很良心的，这边官网直接支持Ubuntu18.04安装包的下载。
[下载](https://music.163.com/#/download)后双击安装即可。





