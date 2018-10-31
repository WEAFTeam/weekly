---
title: 使用Web3j（JAVA）开发ETH钱包-2
description: 使用Web3j（JAVA）开发ETH钱包-2
tags:
  - 区块链
author:
  - earth
thumbnail: 'https://weaf.oss-cn-beijing.aliyuncs.com/web3j.png'
category: 区块链
date: '2018-10-29 18:04:44'
---
一、简介
=========
上文中我们谈到了在测试环境中创建账户并使用
通过这样的方式 ，我们就可以创建自己的钱包地址，在测试环境中我们使用[https://www.rinkeby.io/#faucet](https://www.rinkeby.io/#faucet)给自己充值测试币.
那么我们接下来的工作就是研究ETH的转账，并会在下一篇在讲述，如何使用基于ERC-20智能合约创建自己的Token.

二、ETH转账的实现
===========

转账这里存在一些gas的问题，所谓gas我们都知道是转账时需要花费的旷工费。这里我们可以通过**gasLimit**和**gasPrice**来调整，当然我们肯定希望gas小，但是gas一味的小，会导致没有人愿意帮我们打包我们得到交易，导致交易最终无法成交，形成一个pending的状态。如果gas给的多，那个交易成交速度会比较快，但是我们会费的钱就会变多。
在之前实现的基础上，我们可以拿到自己创建的钱包，和对应钱包的文件和密码。

方法1
-------
这个方法需要使用自己的节点，目前Infura节点无法进行当前操作，对应的也是创建账户**方法1**的转账。
毕竟infura无法帮助我们存储钱包文件

``` java
// ...
private BigInteger gNonce; //每个账户发起的交易都会自动生成一个交易识别号，这个是一个递增的号码

// ...省略部分代码

try {
    EthGetTransactionCount ethGetTransactionCount = admin.ethGetTransactionCount(from, DefaultBlockParameterName.LATEST).sendAsync().get();
    BigInteger nonce = ethGetTransactionCount.getTransactionCount();
    PersonalUnlockAccount personalUnlockAccount = admin.personalUnlockAccount(from,password).send();
    if (personalUnlockAccount == null){
        log.error("转账失败，账户地址错误，或者账户密码错误");
        return 0;
    }
    if (gNonce == null){
        gNonce = nonce;
    }
    if (personalUnlockAccount.accountUnlocked())
    {
        BigInteger gasPrice = Contract.GAS_PRICE;
        BigInteger gasLimit = Contract.GAS_LIMIT;
        synchronized(WalletService.class) {
            Transaction transaction = Transaction.createEtherTransaction(from,nonce,gasPrice,gasLimit,to,amount);
            log.info("转账序号：【{}】",gNonce);
            gNonce = gNonce.add(new BigInteger("1"));
            EthSendTransaction transactionResponse = admin.ethSendTransaction(transaction).sendAsync().get();
            if(transactionResponse.hasError()){
                String message=transactionResponse.getError().getMessage();
                log.error("转账失败，错误信息{}",message);
            }else{
                String hash=transactionResponse.getTransactionHash();
                log.info("转账成功，转账hash = {}",hash);
            }
        }
    }
} catch (InterruptedException | ExecutionException | IOException e) {
    log.error("转账失败，失败原因{}",e);
}
```

这种种方式需要先解锁账户，有一定的安全问题，如果你的节点允许全网访问，会有一些人通过动态扫描以太坊节点，尝试连接这些节点，并通过节点发起发出资金的转账操作。

如果你在测试中又发现自己的币无意间被转到一下地址，说明你的节点或者操作存在以上问题。

- Rinkeby: https://rinkeby.etherscan.io/address/0x7097f41f1c1847d52407c629d0e0ae0fdd24fd58
- Ropsten: https://ropsten.etherscan.io/address/0x7097f41F1C1847D52407C629d0E0ae0fDD24fd58
- Kovan: https://kovan.etherscan.io/address/0x7097f41F1C1847D52407C629d0E0ae0fDD24fd58
- Mainnet: https://etherscan.io/address/0x7097f41F1C1847D52407C629d0E0ae0fDD24fd58

所以这里需要注意不要让你的API端口向全网开放。
这里特别注意一下代码中的 _**nonce**_.下边会讲一下它的其他作用。
其实还有还有其他一些转账的方法，可以自己去研究下。

方法2
-------
这种方式需要使用钱包文件来发起交易。

``` java
try {
    Credentials credentials = WalletUtils.loadCredentials(password,filePath);
    TransactionReceipt transferReceipt = Transfer.sendFunds(web3j, credentials, to,amount, Convert.Unit.WEI).send();
    String hash = transferReceipt.getTransactionHash();
    log.info("转账成功，转账hash = {}",hash);
} catch (Exception e) {
    log.error("转账失败，失败原因{}",e);
}
```

这种方式相对安全一些，这里需要注意的是一些ETH的最小单位——**WEI**.

1ETH = 1*10^18 wei;

三、相关问题
========

### 1.ETH交易模块的特点
1. 必须按顺序处理事务（具有1的nonce为1的事务必须在具有2的nonce的事务之前处理）
2. 不跳过（具有4的nonce的事务不能包含在块中，直到具有1,2,3的nonce的事务

通过这种方式，网络能够识别交易的重复并强制执行订单（这对于智能合约至关重要）
这也就上文中说过的**nonce**,
### 2.如何查看我们转账的状态
我们可以通过转账后拿到的TxHash去[https://etherscan.io/](https://etherscan.io/)查询。

### 3.如何取消Pending状态转账
上文说到了转账因为gas设置的比较小，可能造成无法成交，一直处于Pending状态。
然后我们可以通过**nonce**去取消Pending的转账，
我们通过nonce的机制可以得知，我们只要新创建一笔转账，其中nonce和Pending状态的订单的nonce一样就可以替换当前Pending状态得到转账。

那么最好的操作来实现就是发起一笔向自己转账的订单，当然gas我们需要设置得到合理。
转账参考以上两种方法。

### 4.取消或者取代转账
通过之前**3**中所提到了，我们可以知道，其实一个交易发出后，被覆盖，代替，取消等操作的几率很小。但是也不是不可能。只要当前交易不被挖矿和包含在区块中，是可以做相关关操作的。
具体操作方法还是和**3**相似，但是切记不可更改**nonce**.(具体情况还要和实际相比较，毕竟这种操作完成情况几乎为0)

四、转账状态代码实现
=============
我这里只能查洵成功的和失败的，但是有些转账会查不到，但是不能保证当前转账不存在，或者是pending状态。如果您发现了对应的方法请告知我，谢谢
具体实现
``` java
try {
    EthGetTransactionReceipt transactionReceipt = web3j.ethGetTransactionReceipt(transactionHash).send();
    TransactionReceipt transactionReceipt1 = transactionReceipt.getResult();
    if (transactionReceipt1 == null) {
        log.warn("转账Hash内容为空，Hash = 【{}】",transactionHash);
    }else {
        if ("0x0".equals(transactionReceipt1.getStatus())){
            log.info("转账成功！ Hash = {}",transactionHash)
        }
        if ("0x1".equals(transactionReceipt1.getStatus())){
            log.error("转账失败！ Hash = {}",transactionHash)
        }
    }
} catch (IOException e) {
    e.printStackTrace();
}
```

五、参考文章
=========

1. Checking or Replacing a TX After it's Been Sent：[https://kb.myetherwallet.com/transactions/check-status-of-ethereum-transaction.html](https://kb.myetherwallet.com/transactions/check-status-of-ethereum-transaction.html)
2. Cancel pending transactions on Ethereum:[https://jakubstefanski.com/post/2017/10/ethereum-cancel-pending-transaction/](https://jakubstefanski.com/post/2017/10/ethereum-cancel-pending-transaction/)
3. Node auto send a lot of transaction after call eth.sendTransaction() in one time:[https://github.com/ethereum/go-ethereum/issues/16691](Node auto send a lot of transaction after call eth.sendTransaction() in one time)
4. Web3j：[https://web3j.readthedocs.io/en/latest/](https://web3j.readthedocs.io/en/latest/)