---
title: TensorRT-开发入门
date: 2018-08-12 17:56:37
tags: TensorRT
category: TensorRT
mathjax: true
thumbnail: 'https://s1.ax1x.com/2018/08/12/PcyZQ0.png'
author: Milittle
---

# TensorRT-开发入门

# Docker For Ubuntu 16.04（心路历程）

## 以及在Ubuntu上Docker中使用TensorRT的心路历程，这也是我为以后像搭建出要给实实在在能用的深度学习应用而做的准备



这篇记录在Ubuntu上安装Docker，并且安装nvidia-docker的心路历程，还有NGC（Nvidia GPU Cloud ）的里面的container的使用。（尤其是TensorRT container的一个使用）

## 一、在Ubuntu上安装docker-ce

这里的docker-ce是docker的社区版，因为社区版是不收费的，一般情况下已经够用了。先去官网，[Docker For Ubuntu](https://docs.docker.com/install/linux/docker-ce/ubuntu/) 这里可以找到安装的教程，我这里使用deb文件安装。

1. 去（1）[ubuntu的docker deb文件](https://download.docker.com/linux/ubuntu/dists/)， 根据你的ubuntu的版本，查看Ubuntu版本命令， ` cat /etc/issue` 会显示出ubuntu的版本号，根据版本号在这个网站上（2）[ubuntu版本号名字对应关系表](https://blog.csdn.net/zhengmx100/article/details/78352773)， 然后在（1）链接中找到自己系统的对应名字，点进去` 名字/pool/stable/` 到了这一级目录，就到了选择主机位数的时候了，不清楚的小伙伴，使用` cat /proc/version` 命令查看，我的是amd64。（需要注意一点就是我的ubuntu的名字是xenial）。这里有一个小小的提示，在后面安装nvidia-docker2的时候，需要docker-ce 18.06的支持，所以大家在下载的时候，需要下载这个版本的。（一张图胜过千言万语，下面是deb文件展示，上面是路径，我下载的版本是红色箭头指向的那个版本）

   ![1533028434101](https://s1.ax1x.com/2018/08/12/PcyAWn.png)

2. 下面就开始安装了，安装之前需要废话两句，如果你的系统上已经有了docker怎么办，那要看你是通过apt还是apt-get安装的。命令如下：

```shell
# 查看是否安装命令
apt list --installed | grep docker
# 或者
apt-get list --installed | grep docker
# 以上的两条命令可以查到安装信息，如果你的版本和这个系统的版本是冲突的，那么你可以通过卸载这个当前的版本，安装新的版本，但是前提是你有这个权利做这件事情，sudo权限，而且，你卸载了对其他用户不会造成影响的前提下，如果你卸载了，对其他用户产生影响，后果自负。

# 1 开始安装，如果你是root用户，则不需要加下面的sudo，如果你的用户没有sudo权限，那么需要你让管理员安装，或者，让管理员把你加载sudo组里。
sudo dpkg -i docker-ce_18.06.0_ce_3-0_ubuntu_amd64.deb
# 2 安装结束以后，需要测试一下安装是否成功
sudo docker run hello-world
# 如果以上命令出现一些图2的信息，那么就是成功了
```

![1533029081395](C:\Users\milittle\AppData\Local\Temp\1533029081395.png)

图2

## 二、在ubuntu上安装nvidia-docker和使用NGC container

这里有一个大前提是，你的主机上已经安装过nvidia的驱动了，详细的过程请查看[安装nvidia driver](https://docs.nvidia.com/ngc/ngc-titan-setup-guide/index.html#installing-nvidia-driver)。

1. 安装nvidia-docker(这个步骤一定要在安装docker之后)

```shell
curl -s -L https://nvidia.github.io/nvidia-docker/gpkey | \
	sudo apt-key add -
# 上面是一条命令
curl -s -L https://nvidia.github.io/nvidia-docker/ubuntu16.04/amd64/nvidia-docker.list | \
	sudo tee /etc/apt/sources.list.d/nvidia-docker.list
# 上面是一条命令
## 上面的两条命令都是将源挂载在本地
sudo apt-get update 
sudo apt-get install -y nvidia-docker2 # 这是安装命令
sudo usermod -aG docker $USER # 这一条是将用户添加到docker组中
```

1. 然后去注册NGC[注册教程](https://docs.nvidia.com/ngc/ngc-getting-started-guide/index.html)

上面的教程已经很清楚了。（最主要的是生成那个api key）

![1533044008307](https://s1.ax1x.com/2018/08/12/PcyEzq.png)

附一张注册以后的主页

这里面是所有的container

后面运行示例的时候会下载TensorRT的样例。

1. 本地安装NGC image（这是Docker的镜像文件，可以当作一个application），以及运行image为container

```shell
sudo docker login nvcr.io
#上面的命令是登录nvcr image服务器
sudo docker run --runtime=nvidia --rm nvcr.io/nvidia/cuda:9.0-cudnn7-devel-ubuntu16.04 nvidia-smi
# 上面这条命令是运行cuda环境，后面的那个nvidia-smi是查看GPU信息的，相比大家都很熟悉。
# 这就测试了docker的环境是否是可运行了


# 下面是安装NGC 里面的那些image
# 一下的示例是TensorRT的示例
sudo docker pull nvcr.io/nvidia/tensorrt:18.07-py2
# 上面的命令是将tensorrt的iamge文件下到本地，可能需要点时间。大约2.61G
# 后面就是运行示例
sudo nvidia-docker run -it --rm nvcr.io/nvidia/tensorrt:18.07-py2
# 然后就会进入linux container，这里面已经含有tensorrt的示例了。


# 下面是tensorrt的示例运行。
# c++ example示例运行
# 图4所示

# python example示例运行
# 图5所示


```

![1533046538343](https://s1.ax1x.com/2018/08/12/PcykJs.png)

图3

![1533046802225](https://s1.ax1x.com/2018/08/12/PcyeyV.png)

图4

![1533047137981](https://s1.ax1x.com/2018/08/12/PcyFij.png)

图5

## 三、到此我们就结束了，可能有很多小伙伴说这些太简单了，其实不简单，这些内容是我花了一整天时间，查各种文档，最后简练的将这些东西整合到一起的。如果大家有什么不明白的地方，可以直接给我发邮件或者直接加我qq，我或许可以解答你的疑惑。

qq：329804334

Email：air@weaf.top