---
title: chapter-01-AIR
tags: TensorFlow
mathjax: true
category: TensorFlow
author: milittle
abbrlink: 8e8e4531
date: 2018-03-14 18:49:23
thumbnail: https://s1.ax1x.com/2018/03/18/9oakkQ.png
---

#  第一篇文章-TensorFlow Install

1. 首先介绍一些我们这个组织，这是有四个人构成得一个组织，组织可以叫FOUR ELEMENTS。（也可以叫WEAF）分别对应WELL、EARTH、AIR、FLAME。（WEAF）。
2. 其次我想做一下自我介绍，我的英文学名叫milittle。我开设的这个周刊名字叫AIR-周刊。希望把自己学习的一些内容分享给大家，也激励自己。学更多的知识。以后大家有什么要交流的，也可以一起交流。（邮箱地址会在文章末尾给出）

接下来我讲一下我后续每周在`AIR-周刊`里面会讲到的内容：

* 主要涉及TensorFlow框架使用多一些
* 后续也会分享一些机器学习方面的算法
* 也会有一些在人工智能方面的杂谈

上面说了一些，我想把这块做好，文章内容有什么变化，后续的文章里面会有所提及。

今天就介绍一些TensorFlow的简述和安装：

1. TensorFlow是Google公司在2015年12月份开源的一个机器学习库，代码链接[TensorFLow](https://github.com/tensorflow/tensorflow)。
2. 第二点为什么现在TensorFlow这么火，在人工智能界已经算得上是称霸的地位，我们可以从下面的图中可以看出TensorFlow的数据占据了一大半市场。

![-](https://s1.ax1x.com/2018/03/14/94kzp6.jpg)

3. 原因是什么呢
   * 最主要的原因就是本身具有图运算的这个概念。使用简单，而且可以让程序员快捷的实现一些算法。从而可以用TensorFlow解决一些现实中的问题。图运算的概念我们后续会慢慢深入。大家不要着急。
   * 还有一个原因，我想不用说大家也都知道，既然说了是Google的开源框架，那么技术就一定很牛逼。引得广大程序员的喜爱也是必然发生的事情。
   * 而且用这个框架可以快速的解决一些机器学习的算法问题。是的编程效率也不断提高。
4. TensorFlow支持Mac、Windows、Linux。以后我们的实验有可能通过Windows进行，也有可能在Linux进行，而且以后的代码都是基于python3.X，所以希望大家可以实现基本的python3的语法知识和编程知识。还有就是TensorFlow支持CPU版本和GPU版本，安装的时候都有很多的注意事项，基于GPU版本的可能会比较麻烦。但是后续我会给大家出一个教程，分别在Windows下面和Linux下面配置自己的独立环境。让你的机器学习算法跑在你自己的机器上面。完成一些看起来炫酷的程序。

接下来我介绍一下TensorFlow的Windows CPU安装方法：

1. 首先打开电脑，这个是一定的~
2. 去TensorFlow的[官网](https://www.tensorflow.org/install/)下载Windows的版本。点击下面红色箭头的地方---随意，都可以跳转到一个关于windows安装的界面。（可能需要科学上网，逃）

![-](https://s1.ax1x.com/2018/03/14/94AS1K.png)

3. 点开界面以后的注意事项：
   * windows7及其以后的操作系统版本
   * 决定安装哪个TensorFlow的版本，GPU还是CPU（GPU会有有一些第三方的库依赖，CUDA），接下来我们的教程是CPU版本安装。
   * 决定怎么安装TensorFlow：可选方式有native pip 和 Anaconda等（我们使用Anaconda）
   * 最后一步验证你的安装效果

接下来一步一步来：

第一步、我们决定用Anaconda来安装TensorFlow，你要知道Anaconda是什么呢，它就是可以很好的管理python的一些依赖库。让你在不同python版本之间切换自如。所以我们使用这个工具来安装我们的TensorFlow。Anaconda也可以集成Spyder这些编程工具，使得你编写代码会方便一些。

第二步、首先你去[Anaconda官网](https://www.anaconda.com/download/)下载windows版本的Anaconda，具体安装就和普通的安装软件类似。这个地方需要注意的是不同python版本需要不同的Anaconda，别下错了。

第三步、安装好以后，我们打开Anaconda的控制台，就是开始里面找到Anaconda的应用，然后里面有一个Anaconda Prompt。打开以后，我们就开始了我们创建一个独立的TensorFlow独立的环境。

> ` conda create -n tensorflow pip python=3.5`
>
> 上面这命令的意思就是说在Anaconda管理的环境里面给我独立的创建一个python环境来，这个里面python的版本是3.5。注意一下，这个地方还没有安装tensorflow呢，上面的tensorflow只不过是创建的一个环境名字而已。

![-](https://s1.ax1x.com/2018/03/14/94kjt1.png)

>`activate tensorflow`
>
>上面的命令是激活这个tensorflow的环境，你可以通过这个环境，添加一些你自己的python库，定制自己的python环境，这也是我使用Anaconda的原因，但是并不是只有Anaconda支持这样的方式。不要和我抬杠。

![-](https://s1.ax1x.com/2018/03/14/94kXkR.png)

第四步，也就是正儿八经的安装TensorFlow的阶段，**这里解释一下，上面为什么我执行的是tensorflow1，因为我的电脑上面已经有tensorflow这个环境了**

>`pip install --ignore-installed --upgrade tensorflow`
>
>这个命令就是使用pip正常的安装tensorflow，这里的pip管理起来和普通的pip管理是一个道理，这里就不赘述了。

![-](https://s1.ax1x.com/2018/03/14/94kL79.png)

第五步，测试TensorFlow是否安装上

>`python`
>
>上面的命令是进入python解释器，然后执行下面的import语句
>
>`import tensorflow as tf`
>
>如果上面的命令执行完，如下图中一样，就算安装成功了，下面的那些语句是写了一个hello world！！！

![-](https://s1.ax1x.com/2018/03/22/9Hf66P.png)

今天是为了我们以后在TensorFlow上开发所做的准备。希望大家安装顺利。我的个人邮箱是air@weaf.top。有什么问题可以单独发邮件问我。感谢你们的驻足。有什么不好的地方，可以给出意见。