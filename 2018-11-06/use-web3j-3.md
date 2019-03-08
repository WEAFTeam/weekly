---
title: 使用Web3j（JAVA）开发ETH钱包-3
description: 使用Web3j（JAVA）开发ETH钱包-3
tags:
  - 区块链
author:
  - earth
thumbnail: 'https://weaf.oss-cn-beijing.aliyuncs.com/web3j.png'
category: 区块链
date: '2018-11-06 17:48:58'
abbrlink: 75cd4959
---
一、简介
=========
之前的文章中已经提到了ETH的转账相关内容，接下来，我们将使用智能合约，发布我们自己的Token,并实现Token的转账等相关操作。
下篇文章我会讲解一些关于事件日志、交易、区块监听相关的内容。

二、智能合约的书写
===========

智能合约是使用[**Solidity**](https://solidity.readthedocs.io/en/develop/)书写的，具体学习可以查看官网。
**Solidity**是一门面向合约的、为实现智能合约而创建的高级编程语言。这门语言受到了 C++，Python 和 Javascript 语言的影响，设计的目的是能在以太坊虚拟机（EVM）上运行。Solidity 是静态类型语言，支持继承、库和复杂的用户定义类型等特性。

web3j可以自动生成智能合约包装器代码，以便在不离开JVM的情况下部署智能合约并与之交互。

下边实现一个简单合约：WEAF

三、合约类的生成
========
### 1.合约代码就不多说了，下表面我们使用solc来编译自己的代码

``` shell
solcjs WEAF.sol  --bin --abi --optimize -o ./WEAF
```

查看生成的文件
![](https://weaf.oss-cn-beijing.aliyuncs.com/weaf_sol.png)
### 2.通过abi接口文件使用web3j命令行工具生成Java合约类

``` shell
web3j solidity generate --javaTypes WEAF_sol_WEAF.bin WEAF_sol_WEAF.abi -o WEAF.java -p top.weaf
```
![](https://weaf.oss-cn-beijing.aliyuncs.com/web3j-10.png)

找到我们生产的合约类，然后更改类名称：WEAF

四、智能合约的部署与转账
===============
### 1.部署合约
首先我们将WEAF.java copy到我们自己项目中，使用我们的钱包发起部署

``` java
Web3j web3j = Web3j.build(new HttpService("https://rinkeby.infura.io/v3/ac1907c0a9314f18967a9609570698ad"));
Credentials credentials =WalletUtils.loadCredentials(password,path);
ContractGasProvider contractGasProvider = new DefaultGasProvider();
WEAF contract = WEAF.deploy(web3j, credentials, contractGasProvider, "0xe3342d40dc85a7a0ed0984d89c8905ef491a25dd",new BigInteger("1000000000000000000000")).send();
String contractAddress = contract.getContractAddress();
log.info("Smart contract deployed to address " + contractAddress);
```
然后通过测试网络可以查看到我们部署的合约。
![](https://weaf.oss-cn-beijing.aliyuncs.com/contract.png)

### 2.合约转账
我们知道所有合约都是部署在以太坊上，发送各种交易都是需要花费矿工费（gas），所以在完成自己代币交易的同时，也需要花费eth作为矿工费。
所以要保证我们的交易完成，要确保账户中ETH充足。

实现代币转载的方法不止一种，我这里写下几种，大家可以选择着用
##### 方法1
直接使用合约这是最简单，毋庸置疑。但是并不统用。
``` java
try {
    Credentials credentials = WalletUtils.loadCredentials(password, filePath);
    WEAF weaf = WEAF.load(contractAddress,web3j,credentials,new DefaultGasProvider());
    TransactionReceipt receipt = weaf.transfer(to,amount).send();
    log.info("转账成功，转账hash = {}",receipt.getTransactionHash());
    return 1;
}catch (Exception e){
    log.error("转账失败，错误信息 = {}",e);
    return 0;
}
```
##### 方法2 
这种方法很统用,只需制定合约地址就可以。
```
try {
    Credentials creds = WalletUtils.loadCredentials(password,filePath);
    RawTransactionManager manager = new RawTransactionManager(web3j, creds);
    String data = encodeTransferData(to, amount);
    BigInteger gasPrice = web3j.ethGasPrice().send().getGasPrice();
    BigInteger gasLimit = Constants.GAS_LIMIT;
    EthSendTransaction transaction = manager.sendTransaction(gasPrice, gasLimit, contractAddress, data, BigInteger.ZERO);
    if (transaction.hasError()){
        log.error("转账失败，失败原因：{}",transaction.getError().getMessage());
        return null;
    }
    log.info("转账hash={}",transaction.getTransactionHash());
    return transaction.getTransactionHash();
}catch (Exception e){
    log.error("转账失败，失败原因：{}",e);
    return null;
}
private String encodeTransferData(String toAddress, BigInteger sum) {

    Function function = new Function(
            "transfer",
            Arrays.<Type>asList(new Address(toAddress),
                    new Uint256(sum)),
            Collections.<TypeReference<?>>emptyList());
    return FunctionEncoder.encode(function);
}

```

五、参考文章
=========

1. Solidity：[https://solidity.readthedocs.io/en/develop/](https://solidity.readthedocs.io/en/develop/)
2. Solidity[中文版]：[https://solidity-cn.readthedocs.io/zh/develop/](https://solidity-cn.readthedocs.io/zh/develop/)
3. How To Deploy Smart Contracts Onto The Ethereum Blockchain:[https://medium.com/mercuryprotocol/dev-highlights-of-this-week-cb33e58c745f](https://medium.com/mercuryprotocol/dev-highlights-of-this-week-cb33e58c745f)
4. Remix(Solidity编程工具web):[https://remix.ethereum.org](https://remix.ethereum.org)
5. Greeter：[https://www.ethereum.org/greeter](https://www.ethereum.org/greeter)
6. Web3j#Smart Contracts:[https://web3j.readthedocs.io/en/latest/smart_contracts.html](https://web3j.readthedocs.io/en/latest/smart_contracts.html)