---
title: 阿里云集群版redis中使用lua脚本踩坑记录
description: 阿里云集群版redis中使用lua脚本踩坑记录
tags:
  - java
author:
  - earth
thumbnail: 'https://songyaxu.oss-cn-beijing.aliyuncs.com/blog/ubuntu.png'
category: Linux
date: '2019-09-23 10:34:45'
---
阿里云集群版redis中使用lua脚本踩坑记录
===========

一、前言
-------
最近有一个需求是想统计redis在不同场景下使用命中概率的统计。
我收到领导的任务后不敢懈怠就开始研究lua脚本的语法。并且开始研究如何在java中直接执行lua脚本。

二、踩坑Random函数。
--------

前后经过3个小时左右我开发出了第一个版本。第一个版本大概长这个样子。
``` lua
local key = KEYS[1]
local prefix = ARGV[1]
local date = ARGV[2]
local isCount = ARGV[3]
local hitKey = "stat:"..prefix..":hit"
local missKey = "stat:" .. prefix .. ":miss"
local value = redis.call('get', key)
local inc = 1
if not value
then
    redis.call("hincrby",missKey,date,inc)
else
    redis.call('hincrby',hitKey,date,inc)
end
return value
```
经过测试在本地和单机版上没有问题。

然后领导书要切量10%。
然后这边考虑功能性，直接写在lua脚本里边比较好。

然后我这边一开始想到使用random函数使用时间做随机因子。发现redis里边没有os函数。os.time()不可用。
后来想直接使用math.rand(1, 10),直接取，发现每次都是一个值。（由于redis主从的结构，这边redis是不允许这种对数据一致性有影响的操作进行执行的。）
后来发现这个办法行不通。

所以打算把切量做到外边，然后传一个是否记录命中率的一个true或false进来。

三、踩坑KEYS array 与阿里集群版redis
--------

经过以上的修改脚本大概长这个样子。

``` lua
local key = KEYS[1]
local prefix = ARGV[1]
local date = ARGV[2]
local isCount = ARGV[3]
local hitKey = "stat:"..prefix..":hit"
local missKey = "stat:" .. prefix .. ":miss"
local value = redis.call('get', key)
local inc = 1
if isCount == "true"
then
    if not value
    then
        redis.call("hincrby",missKey,date,inc)
    else
        redis.call('hincrby',hitKey,date,inc)
    end
end
return value
```
本地测试和单机版测试没问题，到线上集群版执行出错了。
```
Caused by: redis.clients.jedis.exceptions.JedisDataException: ERR bad lua script for redis cluster, all the keys that the script uses should be passed using the KEYS array, and KEYS should not be in expression
```
显示就是说所有的redis.call里边的命令必须使用KEYS array。

然后现在脚本长这个样子。


``` lua
local prefix = ARGV[1]
local date = ARGV[2]
local isCount = ARGV[3]
KEYS[2] = "stat:"..prefix..":hit"
KEYS[3] = "stat:" .. prefix .. ":miss"
local value = redis.call('get', KEYS[1])
local inc = 1
if isCount == "true"
then
    if not value
    then
        redis.call("hincrby",KEYS[3],date,inc)
    else
        redis.call('hincrby',KEYS[2],date,inc)
    end
end
return value
```
另外这边还要注意一点就是redis.call执行的命令也是无法使用ARGV array传参的。

四、踩坑slot 与阿里集群版redis
--------

这边专门针对以上代码，在阿里云redis控制台进行执行发现返回值存在，但是hit和miss都不存在。wtf?

看见阿里云官网写着对eval有限制，还以为是这个，写了一个小程序，打包并在线上发布测试。发现并没有什么卵用。

然后这边进过请教阿里云的同事发现问题还是因为集群版的限制问题。
![](https://songyaxu.oss-cn-beijing.aliyuncs.com/blog/redis_lua_1.jpg)

说的是KEYS需要在一个slot上。

所以使用hashtag的写法，然后这边改写脚本，现在长这个样子。
``` lua
KEYS[2] = "stat:{"..ARGV[1].."}:hit"
KEYS[3] = "stat:{"..ARGV[1].."}:miss"
local value = redis.call('get', KEYS[1])
if ARGV[3] == "true"
then
    if not value
    then
        redis.call("hincrby",KEYS[3],ARGV[2],1)
    else
        redis.call('hincrby',KEYS[2],ARGV[2],1)
    end
end
return value
```
首先在阿里云控制台进行测试，没有使用线上数据进行测试，使用我这边测试开发使用的test_222进行测试，因为不存在。所以没报任何错误，而且正确相应。
后来我提议使用现有的测试一下存在的情况。xxx_xxxx_xxxx_abc。发现数据能取出来，但是hit并不存在。
![](https://songyaxu.oss-cn-beijing.aliyuncs.com/blog/nikeyang-ask.jpg)

这边我给test_222设置了一个值，经过测试竟然hit的情况没问题。然后使用其他的就不行。

后来注意到上边说到的一句话。
**单个lua脚本操作的key必须在同一个节点上。**
因为我们之前存储的一些信息内容肯定不是这样的，而且我们既然使用集群版肯定也不希望这样，所以说明通过lua脚本一次性操作完是无法完成的。
而后，我们商讨后。这个场景下肯定无法通过lua脚本去完成了。
下边给出 在控制台执行的脚本命令。

``` shell
eval "KEYS[2] = \"stat:{\"..ARGV[1]..\"}:hit\"\n KEYS[3] = \"stat:{\"..ARGV[1]..\"}:miss\"\n local value = redis.call('get', KEYS[1])\n if ARGV[3] == \"true\"\n then\n if not value\n then\n redis.call(\"hincrby\",KEYS[3],ARGV[2],1)\n else\n redis.call('hincrby',KEYS[2],ARGV[2],1)\n end\n end\n return value" 1 test_222 test 20190927 true
```

这边如果key定义成 {xxxxxx}\_xxxx、stat:{xxxxxx}:miss stat:{xxxxxx}:hit,是可以完成的，但是我们的key是已经存在的，而且不会更改现有逻辑去使用这种命名格斯，更不会让所以key都在一个节点上。

至此踩坑结束。

