---
title: EOS智能合约从零到一-2
description: EOS智能合约从零到一-2
tags:
  - 区块链
author:
  - earth
thumbnail: 'https://weaf.oss-cn-beijing.aliyuncs.com/eos-logo.jpg'
category: 区块链
date: '2019-01-25 13:49:23'
---
一、简介
=========
本章节主要介绍钱包、钱包工具和我们安装工具的一些使用方法和对工具的一些了解，以便使我们接下来开发不会遇到问题。

二、了解eos工具
===========
那么我们已经安装好了工具，也下载好了相应的eos生态所需的东西，那么他们是怎么工作的，又是怎么联系到一起的呢？如下图

- nodeos (node + eos = nodeos) - 这个是EOS生态系统的核心，它可以通过插件的配置来运行一个EOS节点，主要用途是：区块生产、提供API服务，和本地开发环境。
- cleos (cli + eos = cleos) - 命令行工具，我们可以通过它来和区块链交互，或者用来管理钱包。
- keosd (key + eos = keosd) - 是EOS钱包用来安全存储EOS私钥的工具。
- eosio-cpp - 这个之前可能还没接触过，这个是eosio.cdt的一部分，它是用来把C++代码编译成WASM和生产ABI的工具。

![eos-8]()

三、玩转Wallet
========
### 1.创建钱包
我们这里开始使用之前安装的工具了，这个我们先看下cleos 都有什么命令：
![eos-9]()
这里我们看到有很多命令，后边有些我们会陆续的说到，其他说不到的，有兴趣的小伙伴可以去了解一下。
今天我们主要用到wallet命令，然后我们查看下wallet的用法。
![eos-10]()
这里边的命令我们都会用到，我简单列一个表，方便copy:

| 命令 | 使用方式 |描述|展示|
| --- | --- | --- | --- |
|cleos wallet create       |cleos wallet create --to-console/cleos wallet create --file|创建钱包,这里需要指定输出方式，输出的密码我们需要记下来，以便以后使用，我这里添加了一个**-n**的参数来指定钱包名称，不指定默认是**default**|![eos-11]()|
|cleos wallet list         |cleos wallet list                                          |列出所有钱包 |![eos-12]()|
|cleos wallet open         |cleos wallet open                                          |打开一个存在的钱包，所谓打开，是使用list的时候可以看见。 **-n**参数指定钱包名称，不指定默认是**default** |![eos-13]()|
|cleos wallet lock/lock_all|cleos wallet lock / cleos wallet lock_all                  |**-n**参数，锁上指定钱包，或者锁上所有打开的钱包|![eos-14]()|
|cleos wallet unlock       |cleos wallet unlock                                        |**-n**参数，解锁指定钱包，这时我们要用到之前创建钱包保存的密码，我们可以使用 **--password xxxxxxxx** 来指定，或者等待后续输入，_我们要注意钱包list的区别，发现解锁的钱包右上角会有一个*号_|![eos-15]()|
|cleos wallet keys         |cleos wallet keys                                          |可以列出当前所有解锁的钱包的公钥|因为现在我们还没创建任何key所以显示是一个空数组**[]**|
|cleos wallet create_key   |cleos wallet create_key                                    |**-n**参数，指定key的钱包在哪个，默认是**default**,然后我们可以使用keys命令来查看|![eos-16]()|
|cleos wallet remove_key   |cleos wallet remove_key KEY -n WALLET_NAME                 |这里指定了钱包还需要指定key,并且还得输入密码 也可以使用**--password xxxxxxxx** 来指定密码，不指定钱包默认是**default**|![eos-17]()|
|cleos wallet import       |cleos wallet import --private-key PRIVATE_KEY -n WALLET_NAME|这里我们指定私钥来导入key,我们使用**-n**来指定导入的钱包，不指定钱包默认是**default**|![eos-18]()|
|cleos wallet private_keys |cleos wallet private_keys                                  |列出钱包的私钥，我们使用**-n**来指定钱包，不指定钱包默认是**default**，也可以使用**--password xxxxxxxx** 来指定密码|![eos-19]()|
|cleos wallet stop         |cleos wallet stop                                          |(这个我没有用过，一般不会用到)||


五、参考文章
=========

1. eos 开发者：[https://developers.eos.io/eosio-home/docs/introduction](https://developers.eos.io/eosio-home/docs/introduction)
2. EOS.GITHUB:[https://github.com/EOSIO/eos](https://github.com/EOSIO/eos)
