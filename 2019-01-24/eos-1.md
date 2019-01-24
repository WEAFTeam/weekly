---
title: EOS智能合约从零到一-1
description: EOS智能合约从零到一-1
tags:
  - 区块链
author:
  - earth
thumbnail: 'https://weaf.oss-cn-beijing.aliyuncs.com/eos-logo.jpg'
category: 区块链
date: '2019-01-24 16:21:58'
---
一、简介
=========
之前写过关于Solidity的只能合约，但是现在因为公司的业务的原因，我们又要搞eos合约的开发，所以我就开始搞eos只能合约相关的开发，其实之前也是知道使用的是C++写的，但是没有真正看过，这次马上就要开始了。

二、了解EOS
===========
EOS，可以理解为Enterprise Operation System，即为商用分布式应用设计的一款区块链操作系统。EOS是引入的一种新的区块链架构，旨在实现分布式应用的性能扩展。注意，它并不是像比特币和以太坊那样的货币，而是基于EOS软件项目之上发布的代币，被称为区块链3.0。

三、入门准备
========
### 1.开发环境（本文开发环境）
| 名称 | 版本 |描述|
| --- | --- | --- |
|Ubuntu         |18.04|开发环境支持多种操作系统，但是不支持windows,因为我的电脑是安装的ubuntu/windows的双开发系统，所以无需顾虑|
|nodeos         |1.5.0|eos本地搭建节点版本|
|cleos          |1.5.1|eos开发工具|
|keosd          |1.5.0|eos钱包管理工具|
|eosio.cdt	    |1.5.0|eos只能合约开发工具包（The EOSIO Contract Development Toolkit,CDT for short.）|
|eosio.contracts|1.5.2|eos本地智能合约系统使用合约|
|Sublime Text3  |3.1.1|编辑器|

这边也支持使用docker,如果你会docker,可以使用docker并使用windows开发，但是eos官方已经从2018年6月不再对docker镜像进行维护，所以我这边还是推荐使用linux和手里有mac的小伙伴去开发。

### 2.开发技能

1. 至少需要了解一些区块链相关的知识
2. 有过开发语言的经验，最好是C/C++.对Linux/Mac OS使用的经验。
3. 有命令行相关知识。

### 3.可使用编译器

1. [Sublime Text:https://www.sublimetext.com/](https://www.sublimetext.com/)
2. [Atom Editor:https://atom.io/](https://atom.io/)
3. [CLion:https://www.jetbrains.com/clion/](https://www.jetbrains.com/clion/)
4. [Eclipse:http://www.eclipse.org/downloads/packages/release/oxygen/1a/eclipse-ide-cc-developers](http://www.eclipse.org/downloads/packages/release/oxygen/1a/eclipse-ide-cc-developers)
5. [Visual Studio Code:https://code.visualstudio.com/](https://code.visualstudio.com/)

我这边使用的是一个编辑器，并没有使用编译器，配置环境比较麻烦。
如果有需要我这里有看到两篇，大家可以借鉴：

1. [Visual Studio Code Setup:https://infinitexlabs.com/setup-ide-for-eos-development/](https://infinitexlabs.com/setup-ide-for-eos-development/)
2. [CLion Setup: https://infinitexlabs.com/setup-ide-for-eos-development/](https://infinitexlabs.com/setup-ide-for-eos-development/)

四、准备环境
===============
### 1.创建一个开发相关的目录
这里我我在当前用户根目录下创建一个eos目录，并在下边创建一个contracts目录存放合约。

``` shell
cd ~
mkdir eos
cd eos
mkdir contracts
```
![eos-1](https://weaf.oss-cn-beijing.aliyuncs.com/eos-1.png)
### 2.下载和安装
下载和安装eosio
这里我们在下载之前切换到我们创建的eos目录
```shell
cd ~/eos
wget https://github.com/eosio/eos/releases/download/v1.5.0/eosio_1.5.0-1-ubuntu-18.04_amd64.deb
sudo apt install ./eosio_1.5.0-1-ubuntu-18.04_amd64.deb
```
然后启动钱包工具keosd
``` shell
keosd &
```
启动成功你可以看到如下输出：
![eos-2](https://weaf.oss-cn-beijing.aliyuncs.com/eos-2.png)
然后我们还是在**~/eos**目录下启动我们的本地节点
```shell
nodeos -e -p eosio \
--plugin eosio::producer_plugin \
--plugin eosio::chain_api_plugin \
--plugin eosio::http_plugin \
--plugin eosio::history_plugin \
--plugin eosio::history_api_plugin \
--data-dir CONTRACTS_DIR/eosio/data \
--config-dir CONTRACTS_DIR/eosio/config \
--access-control-allow-origin='*' \
--contracts-console \
--http-validate-host=false \
--verbose-http-errors \
--filter-on='*' >> nodeos.log 2>&1 &
```
![eos-3](https://weaf.oss-cn-beijing.aliyuncs.com/eos-3.png)
启动节点后我们可以看到当前目录下有一个**nodeos.log**的文件，这个是本地节点的log输出文件，我们使用tail 命令来动态的查看输出

```shell
tail -fn 400 nodeos.log
```
可以看见生产区块的日志
![eos-5](https://weaf.oss-cn-beijing.aliyuncs.com/eos-5.png)

有时候我们非关闭电脑，或者节点时，再次使用以上命令就会出现以下错误
![eos-4](https://weaf.oss-cn-beijing.aliyuncs.com/eos-4.png)
出现当前问题我们可以加一个参数**--replay-blockchain --hard-replay-blockchain**
也就是使用如下命令来启动
```shell
nodeos -e -p eosio --plugin eosio::producer_plugin --plugin eosio::chain_api_plugin --plugin eosio::http_plugin --plugin eosio::history_plugin --plugin eosio::history_api_plugin --data-dir CONTRACTS_DIR/eosio/data --config-dir CONTRACTS_DIR/eosio/config --access-control-allow-origin='*' --contracts-console --http-validate-host=false --verbose-http-errors --filter-on='*' --replay-blockchain --hard-replay-blockchain >> nodeos.log 2>&1 &
```
通过脚本查看钱包账户的一些信息
```shell
cleos wallet list
```
![eos-6](https://weaf.oss-cn-beijing.aliyuncs.com/eos-6.png)

可以看到没有钱包，或者都是非open状态（我们这里是因为没有钱包）

检查eos节点
直接使用浏览器，或者使用命令行
```shell
curl http://localhost:8888/v1/chain/get_info
```
![eos-7](https://weaf.oss-cn-beijing.aliyuncs.com/eos-7.png)

安装CDT
```shell
cd ~/eos
wget https://github.com/EOSIO/eosio.cdt/releases/download/v1.5.0/eosio.cdt_1.5.0-1_amd64.deb
sudo apt install ./eosio.cdt_1.5.0-1_amd64.deb
```
卸载（可能不会用到）
```shell
sudo apt remove eosio.cdt
```
安装Sublime Text3
如果有需要的小伙伴可以看看

- [Linux instal setup](http://www.sublimetext.com/docs/3/linux_repositories.html)
- [解决ubuntu下sublime text3不能输入中文问题](https://blog.csdn.net/lu_embedded/article/details/79558280)

五、参考文章
=========

1. 百度百科-EOS：[https://baike.baidu.com/item/EOS/20441174?fr=aladdin](https://baike.baidu.com/item/EOS/20441174?fr=aladdin)
2. 设置eos开发环境ide：[https://infinitexlabs.com/setup-ide-for-eos-development/](https://infinitexlabs.com/setup-ide-for-eos-development/)
3. eos 开发者：[https://developers.eos.io/eosio-home/docs/introduction](https://developers.eos.io/eosio-home/docs/introduction)
4. EOS.GITHUB:[https://github.com/EOSIO/eos](https://github.com/EOSIO/eos)
5. [Linux install setup](http://www.sublimetext.com/docs/3/linux_repositories.html)
6. [解决ubuntu下sublime text3不能输入中文问题](https://blog.csdn.net/lu_embedded/article/details/79558280)
