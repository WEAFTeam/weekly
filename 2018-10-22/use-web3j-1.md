---
title: 使用Web3j（JAVA）开发ETH钱包-1
description: 使用Web3j（JAVA）开发ETH钱包-1
tags:
  - 区块链
author:
  - earth
thumbnail: 'https://weaf.oss-cn-beijing.aliyuncs.com/web3j.png'
category: 区块链
date: '2018-10-27 09:04:44'
---
一、简介
=========
想要使用**web3j**开发ETH钱包，我们需要把准备工作做好，那么先让我们来了解下什么是[web3j:https://web3j.readthedocs.io/en/latest/](https://web3j.readthedocs.io/en/latest/),我这里是最新版本的地址，随着时间的变化，我们需要使用心得版本来编写我们的程序。
![](https://weaf.oss-cn-beijing.aliyuncs.com/web3j-1.png)

**web3j**是一个高度模块化，反应灵敏，类型安全的Java和Android库，用于处理智能合约并与以太坊网络上的客户端（节点）集成：这使您可以使用以太坊区块链，而无需为平台编写自己的集成代码的额外开销。
Java和Blockchain对话提供了区块链，以太坊和web3j的概述。

二、准备工作
=================

想要使用web3j，我们只需使用项目构建工具引入就可

### Maven
**Java 8:**

``` xml
<dependency>
  <groupId>org.web3j</groupId>
  <artifactId>core</artifactId>
  <version>3.6.0</version>
</dependency>
```

**Android:**
``` xml
<dependency>
  <groupId>org.web3j</groupId>
  <artifactId>core</artifactId>
  <version>3.3.1-android</version>
</dependency>
```
### Gradle
**Java 8:**
``` xml
compile ('org.web3j:core:3.6.0')
```
**Android:**
``` xml
compile ('org.web3j:core:3.3.1-android')
```
然后开启自己的节点，上一篇是说使用自己的geth客户端开启，但是我们现在使用infura,来创建自己的项目

[Infura:https://infura.io/](https://infura.io/)

注册成功后创建自己的项目：
![](https://weaf.oss-cn-beijing.aliyuncs.com/infura.png)
可以看到 我们可以使用不同环境的不同节点。

三、账户创建和充值及部分代码。
================

我们现在已经把几乎所有的工作都昨做完了，但是还有一个就是我们钱包的eth的获取，我们可以通过挖矿获取，但是这个是测试环境我们还可以通过其他方式获取（在真实环境我们可以通过交易所来购买）。

现在我们先写下我们创建账户和入门web3j的代码：

``` java
/* We start by creating a new web3j instance to connect to remote nodes on the network. 
   实例化web3j，这里HttpService()可不传参，默认
*/
Web3j web3j = Web3j.build(new HttpService("https://rinkeby.infura.io/v3/3f0abe3dcf554486a363809349898253"));
Admin admin = Admin.build(new HttpService("https://rinkeby.infura.io/v3/3f0abe3dcf554486a363809349898253"));
log.info("Connected to Ethereum client version: "+ web3j.web3ClientVersion().send().getWeb3ClientVersion());

```
创建钱包地址：

### 方法1

这个方法这是和自己的节点，自己创建账户后，对应的钱包文件会存在../keystore/ 的目录下
``` java 
try {
    NewAccountIdentifier newAccountIdentifier = admin.personalNewAccount(password).send();
    log.info("创建ETH账户成功,账户id = {}",newAccountIdentifier.getAccountId());
    return newAccountIdentifier.getAccountId();
} catch (IOException e) {
    log.error("创建ETH账户失败，错误信息：{}",e);
    return null;
}
```

### 方法2

这个方式需要自己制定生产钱包文件的目录
``` java
String walletFileName="";
String password = "123456qwerty";
String walletFilePath="C:\\Users\\yaxuSong\\wallet";
walletFileName = WalletUtils.generateNewWalletFile(password, new File(walletFilePath));
log.info("walletName: "+walletFileName);
```
通过这样的方式 ，我们就可以创建自己的钱包地址，在测试环境中我们使用[https://www.rinkeby.io/#faucet](https://www.rinkeby.io/#faucet)给自己充值测试币

首先我们需要使用[Twitter](https://twitter.com/intent/tweet?text=Requesting%20faucet%20funds%20into%200x0000000000000000000000000000000000000000%20on%20the%20%23Rinkeby%20%23Ethereum%20test%20network.)、[Facebook](https://www.facebook.com/help/community/question/?id=282662498552845)、[Google Plus](https://plus.google.com/)发布新的公开内容，内容是自己的钱包地址，然后粘贴相应的文章的地址到faucet.

我这里使用的是Google Plus.

##### 1.发布带自己钱包地址的公开文章

发布成功之后点击分析图表进入的界面如下，就是自己的文章(post)的地址：[https://plus.google.com/103762921276095780168/posts/fhTRmP6gXDb](https://plus.google.com/103762921276095780168/posts/fhTRmP6gXDb)
![](https://weaf.oss-cn-beijing.aliyuncs.com/googleplus.png)

##### 2.粘贴地址
然后进入[faucet](https://www.rinkeby.io/#faucet)粘贴地址进行充值。
![](https://weaf.oss-cn-beijing.aliyuncs.com/rinkeby-1.png)

粘贴文章地址，并选择充值数量
![](https://weaf.oss-cn-beijing.aliyuncs.com/rinkeby-2.png)
点击充值，显示接受转账
![](https://weaf.oss-cn-beijing.aliyuncs.com/rinkeby-3.png)
显示已经转账成功
![](https://weaf.oss-cn-beijing.aliyuncs.com/rinkeby-4.png)


这里需要注意，有时候测试会有一些卡，可能有很多转账都有发放，所以请等待没有排队对列时进行，不然也不会到账。
##### 3.查看是否到账和账户状态
我们可以通过我们的地址去 [explorer:https://www.rinkeby.io/#explorer](https://www.rinkeby.io/#explorer)

如图：
![](https://weaf.oss-cn-beijing.aliyuncs.com/rinkeby-5.png)

这样我们就有可以操作的ETH进行转账等操作了。

四、参考代码、DEMO
==========
sample :[https://github.com/web3j/sample-project-gradle](https://github.com/web3j/sample-project-gradle)
因为项目是用Gradle构建的，我这边在下载依赖时出现很多问题，后来改了下仓库，具体修改如下：

### build.gradle

``` gradle
group 'org.web3j'
version '0.0.1'

apply plugin: 'java'

sourceCompatibility = 1.8

repositories {
    maven{url 'http://maven.aliyun.com/nexus/content/groups/public/'}
}

ext {
    web3jVersion = '3.4.0'
    logbackVersion = '1.2.3'
    junitVersion = '4.12'
}

dependencies {
    compile "org.web3j:core:$web3jVersion",
            "ch.qos.logback:logback-core:$logbackVersion",
            "ch.qos.logback:logback-classic:$logbackVersion"
    testCompile "junit:junit:$junitVersion"
}

```

五、参考文档
================

Web3j：[https://web3j.readthedocs.io/en/latest/](https://web3j.readthedocs.io/en/latest/)

