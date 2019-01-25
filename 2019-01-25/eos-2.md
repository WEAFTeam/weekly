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

![eos-8](https://weaf.oss-cn-beijing.aliyuncs.com/eos-8.png)

三、玩转Wallet
========
### 1.创建钱包
我们这里开始使用之前安装的工具了，这个我们先看下cleos 都有什么命令：
![eos-9](https://weaf.oss-cn-beijing.aliyuncs.com/eos-9.png)
这里我们看到有很多命令，后边有些我们会陆续的说到，其他说不到的，有兴趣的小伙伴可以去了解一下。
今天我们主要用到wallet命令，然后我们查看下wallet的用法。
![eos-10](https://weaf.oss-cn-beijing.aliyuncs.com/eos-10.png)
这里边的命令我们都会用到，我简单列一个表，方便copy:

| 命令 | 使用方式 |描述|展示|
| --- | --- | --- | --- |
|cleos wallet create       |cleos wallet create --to-console/cleos wallet create --file|创建钱包,这里需要指定输出方式，输出的密码我们需要记下来，以便以后使用，我这里添加了一个**-n**的参数来指定钱包名称，不指定默认是**default**|![eos-11](https://weaf.oss-cn-beijing.aliyuncs.com/eos-11.png)|
|cleos wallet list         |cleos wallet list                                          |列出所有钱包 |![eos-12](https://weaf.oss-cn-beijing.aliyuncs.com/eos-12.png)|
|cleos wallet open         |cleos wallet open                                          |打开一个存在的钱包，所谓打开，是使用list的时候可以看见。 __-n__ 参数指定钱包名称，不指定默认是**default** |![eos-13](https://weaf.oss-cn-beijing.aliyuncs.com/eos-13.png)|
|cleos wallet lock/lock_all|cleos wallet lock / cleos wallet lock_all                  |**-n**参数，锁上指定钱包，或者锁上所有打开的钱包|![eos-14](https://weaf.oss-cn-beijing.aliyuncs.com/eos-14.png)|
|cleos wallet unlock       |cleos wallet unlock                                        |**-n**参数，解锁指定钱包，这时我们要用到之前创建钱包保存的密码，我们可以使用 **--password xxxxxxxx** 来指定，或者等待后续输入，_我们要注意钱包list的区别，发现解锁的钱包右上角会有一个*号_|![eos-15](https://weaf.oss-cn-beijing.aliyuncs.com/eos-15.png)|
|cleos wallet keys         |cleos wallet keys                                          |可以列出当前所有解锁的钱包的公钥|因为现在我们还没创建任何key所以显示是一个空数组**[]**|
|cleos wallet create_key   |cleos wallet create_key                                    |**-n**参数，指定key的钱包在哪个，默认是**default**,然后我们可以使用keys命令来查看|![eos-16](https://weaf.oss-cn-beijing.aliyuncs.com/eos-16.png)|
|cleos wallet remove_key   |cleos wallet remove_key KEY -n WALLET_NAME                 |这里指定了钱包还需要指定key,并且还得输入密码 也可以使用 __--password xxxxxxxx__ 来指定密码，不指定钱包默认是**default**|![eos-17](https://weaf.oss-cn-beijing.aliyuncs.com/eos-17.png)|
|cleos wallet import       |cleos wallet import --private-key PRIVATE_KEY -n WALLET_NAME|这里我们指定私钥来导入key,我们使用**-n**来指定导入的钱包，不指定钱包默认是**default**|![eos-18](https://weaf.oss-cn-beijing.aliyuncs.com/eos-18.png)|
|cleos wallet private_keys |cleos wallet private_keys                                  |列出钱包的私钥，我们使用**-n**来指定钱包，不指定钱包默认是**default**，也可以使用**--password xxxxxxxx** 来指定密码|![eos-19](https://weaf.oss-cn-beijing.aliyuncs.com/eos-19.png)|
|cleos wallet stop         |cleos wallet stop                                          |(这个我没有用过，一般不会用到)||

### 2.wallet实践

这边我们开始到测试网创建自己的账号，然后导入，为我们后边使用测试网测试打好基础。

1. [Fastest EOS Block Explorer:https://kylin.eosx.io/](https://kylin.eosx.io/)
2. [麒麟测试网：https://www.cryptokylin.io/](https://www.cryptokylin.io/)
3. [Jungle 测试网：https://monitor.jungletestnet.io/](https://monitor.jungletestnet.io/)
我们使用以上网站就足够了

##### 使用麒麟测试网
麒麟测试网需要和eosx.io配合使用，Jungle单个自己其实就足够了，但是我们使用eosx.io自己其实也就满足了我们大多数的要求。

我们先使用麒麟测试网创建账户，获取 EOS token（这里注意到我说EOS是token，其实EOS是运行在eosio生态系统上的代币，这里没有coin一说，后面我们会讲到eosio.token,大家就会知道了）。

+ [http://faucet.cryptokylin.io/create_account?valid_account_name](http://faucet.cryptokylin.io/create_account?valid_account_name) :刚我就是使用这个创建的testforeos11账户并拿到私钥导入到本地的。
+ [http://faucet.cryptokylin.io/get_token?valid_account_name](http://faucet.cryptokylin.io/get_token?valid_account_name)：可以获取EOS代币，每天做多1000.0000EOS。

上边两点需要我们关注

1. **需要替换掉[valid_account_name]为我们的名称**。
2. **账户名称需要是12为，并且只允许输入字符 a-z 和1-5**。

以下信息是我使用麒麟测试网创建账户得到的，我们需要保存下来（如果是主网我们需谨慎保管好私钥，防止丢失和泄漏）。

``` json
{
    "msg":"succeeded",
    "keys":{
        "active_key":{
            "public":"EOS5FEdnkyV9Rd61dsgQ5fxrNjyHFX2iK2uFo7CsGfWVV59ebfBMi",
            "private":"5HyeMsoDJi8vJCdnyZuga1JoEWq3nyM71Xb2Mnuv5zugsK6jr4W"
        },
        "owner_key":{
            "public":"EOS7TFLT13QVegchWyich21M8fWXBauRaZLWey3z5oGUkr2r3jEL3",
            "private":"5HsRwxQW4eTt6z6RPjnTVihR9swiFTGcxiCkhhUjBheh87BzxQZ"
        }
    },
    "account":"testforeos11"
}
```
我们注意到有两个公钥和私钥，这里和EOS的权限系统有关，后续我们会讲到。
下边是我们私用faucet获取的EOS代币
![eos-20](https://weaf.oss-cn-beijing.aliyuncs.com/eos-20.png)

我们可以使用eosx.io来查看具体信息，这边有兴趣的可以研究下这个网站，里边有很多东西后续我们可以用到。
![eos-21](https://weaf.oss-cn-beijing.aliyuncs.com/eos-21.png)

##### 使用Jungle测试网

Jungle和麒麟测试网不同，需要我们自己来生产公私钥，然后创建账户。
我们在本地生产两个（分别对应上边麒麟测试网中的active_key和owner_key）或者直接使用上边我们麒麟测试网的,我这里直接使用以上两个。
如图
![eos-22](https://weaf.oss-cn-beijing.aliyuncs.com/eos-22.png)
下边会有一些详情，我们现在只需关注**newaccount**就可以了。
我们通过网站头部导航**Faucet**获取EOS 然后通过**Account Info**查看账户信息
![eos-23](https://weaf.oss-cn-beijing.aliyuncs.com/eos-23.png)

### 3.创建账户

##### 概念介绍
在eos中账户的概念并非和其他币如ETH的概念一样模棱两可，在这里eos的账户很明确。

以下是官方原文：
![eos-24](https://weaf.oss-cn-beijing.aliyuncs.com/eos-24.png)
这是我的翻译：
帐户是一组授权，存储在区块链中，用于标识发件人/收件人。它具有灵活的授权结构，使其可以由个人或组织/部门拥有，具体取决于如何配置权限。需要一个帐户才能向区块链发送或接收有效交易本教程系列使用两个“用户”帐户，bob和alice，以及默认的eosio帐户进行配置。此外，账户还可以为不同的合约来指定。

##### 创建账户
```shell
cleos create account eosio test1 EOS7HAsFJgnmfTsKcnYXVba67h6Fg5HZa2c1S3aGAuoxQGGCVx77G
cleos create account eosio test2 EOS7HAsFJgnmfTsKcnYXVba67h6Fg5HZa2c1S3aGAuoxQGGCVx77G
```
上边的eosio账户是eosio系统的默认账户（具体请查看），我们创建test1和test2是使用eosio账户的默认配置进行初始化的。（创建账户时eosio所在的钱包和当前公钥所在的钱包必须是unlocked状态，也就是右上角有*号）
这里我们需要注意的就是，现在的权限是依赖于eosio生态的，所以我们的本地节点需要启动起来。这里我们将账户和一个公钥关联起来，每一个EOSIO的账户都会关联一个公钥。
![eos-25](https://weaf.oss-cn-beijing.aliyuncs.com/eos-25.png)

##### 查看账户相关信息

```shell
cleos get account test1
```
使用以上命令可以查看账户绑定的公钥
![eos-26](https://weaf.oss-cn-beijing.aliyuncs.com/eos-26.png)

注意，实际上test1、test2同时拥有所有者和活动公钥。 EOSIO具有独特的授权结构，为您的帐户增加了安全性。 在使用与您的活动权限相关联的密钥时，您可以通过保持所有者密钥不开放来最小化帐户的风险。 这样，如果您的有效密钥遭到入侵，您可以使用所有者密钥重新控制您的帐户。
另外这里是本地环境，所以这里一个账户一个key拥有两个授权，其实我们通过麒麟测试网创建的账户来看，我们可以使用不同的key来绑定不同的授权。来确保我们账户的安全。这就是eos的强大的地方。
![eos-27](https://weaf.oss-cn-beijing.aliyuncs.com/eos-27.png)
这里我通过指定API 使用cleos 工具获取麒麟测试网的账户公钥信息，我们可以看到不同的权限绑定在不同的key上。

四、补充eosio
=========
上边有用到eosio的账户，这是一个默认系统账户，也是eosio生态的一个主要的账户，这里我们在本地开发，避免不了使用这个账户，所以我们需要创建一个这个账户，那么我们使用我们默认的值来创建**default**钱包，并在里边创建一个key,
每一个EOSIO都会有这么一个key,我们称作Development Key。
这里有一个方法导入Development Key（这里会默认导入**default**钱包，导入时保证默认钱包是unlocked状态），使用一下脚本：
```shell
cleos wallet import
```
然后他会提示我们输入private key:
然后我们可以使用以下私钥：
``` text
5KQwrPbwdL6PhXujxW37FSSQZ1JiwsST4cqQzDeyXtP79zkvFD3
```
**这里是测试网络使用的，请勿放在主网的节点上！**
这边默认的eosio的账户对应的key是EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV
但是我们发现使用获取账户名称的时候获取不到...
![eos-28](https://weaf.oss-cn-beijing.aliyuncs.com/eos-28.png)

五、使用Scatter钱包
========

##### 创建网络节点
我们也可以使用[Scatter:https://get-scatter.com/](https://get-scatter.com/)钱包工具来查看并管理我们的钱包,我这里使用的是chrome的插件。
我们首先设置我们的节点，点开设置->NetWorks->New 来新建节点信息,填写完成并点击save(保存)
通过访问 [http://localhost:8888/v1/chain/get_info](http://localhost:8888/v1/chain/get_info)
获取到本地节点信息(主要是chain_id)
``` json
{
    "server_version":"ea08cfd3",
    "chain_id":"cf057bbfb72640471fd910bcb67639c22df9f92470936cddc1ade0e2f2e7dc4f",
    "head_block_num":984839,
    "last_irreversible_block_num":984838,
    "last_irreversible_block_id":"000f07063e914c8d5025bb3bd33fa29eb5bfcf09c6dfb881cd9afe507482cb20",
    "head_block_id":"000f07077dcb7d44e6e0c08e5c0bb76fc111bba2f10fb3584033d237a1c0e287",
    "head_block_time":"2019-01-25T09:18:29.000",
    "head_block_producer":"eosio",
    "virtual_block_cpu_limit":200000000,
    "virtual_block_net_limit":1048576000,
    "block_cpu_limit":199900,
    "block_net_limit":1048576,
    "server_version_string":"v1.5.0"
}
```
如下图
![eos-29](https://weaf.oss-cn-beijing.aliyuncs.com/eos-29.png)

##### 导入私钥
找到Key Pairs->New ,填写私钥，填写一个名称（随意，并非account_name）,并点击保存

##### 创建身份

找到 Identities->New，选择我们本地节点，并选择我们导入的key的名称，选择一个权限，这里选择active就行，然后点击import。

测试网络也是以上步骤哦。
具体也可以参考六中的3.Chrome 钱包 Scatter 使用教程[https://www.twle.cn/t/503](https://www.twle.cn/t/503)

六、参考文章
=========

1. eos 开发者：[https://developers.eos.io/eosio-home/docs/introduction](https://developers.eos.io/eosio-home/docs/introduction)
2. EOS.GITHUB:[https://github.com/EOSIO/eos](https://github.com/EOSIO/eos)
3. Chrome 钱包 Scatter 使用教程[https://www.twle.cn/t/503](https://www.twle.cn/t/503)
