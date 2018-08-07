---
title: Redisson 的介绍与使用
description: Redisson 的介绍与使用
tags:
  - JAVA
author:
  - earth
thumbnail: 'https://redisson.org/assets/logos/logo.png'
category: JAVA
date: '2018-06-05 02:04:44'
---
一、简介
======
![Redisson-1](https://camo.githubusercontent.com/5664d7af9b355209fac47098d3b51f64c4e265c7/68747470733a2f2f7265646973736f6e2e6f72672f6c6f676f2e706e67)
Redisson是一个在Redis的基础上实现的Java驻内存数据网格（In-Memory Data Grid）。它不仅提供了一系列的分布式的Java常用对象，还提供了许多分布式服务。其中包括(BitSet, Set, Multimap, SortedSet, Map, List, Queue, BlockingQueue, Deque, BlockingDeque, Semaphore, Lock, AtomicLong, CountDownLatch, Publish / Subscribe, Bloom filter, Remote service, Spring cache, Executor service, Live Object service, Scheduler service) Redisson提供了使用Redis的最简单和最便捷的方法。Redisson的宗旨是促进使用者对Redis的关注分离（Separation of Concern），从而让使用者能够将精力更集中地放在处理业务逻辑上。

二、分布式锁
=====
今天也只是就Redisson分布式锁这一部分进行讲解。
在很多时候，一个服务器已经不满足我们对服务性能的要求了，所以我们引进和很多相关的技术，引入了分布式服务的方式，但是这样既带来了方便，其实也带来了问题，多台服务器一起运行相同的代码，一旦两台服务运行了相同的代码，那就要保证服务"**幂等性**"的设计。而且要考虑服务并发带来的可能性问题。

那么我们现在引入分布式锁的概念，顾名思义，就是使用锁，将我们要紧行操作的对象或者数据加锁。操作后释放锁。使得此时操作这个对象的服务器只有一个（或者说只能对当前对象进行当前拿到锁的操作，拿不到锁的都不允许操作）。这样保证数据的每次操作前后都不会有问题。当然同一个服务器也只能对当前对象进行一个操作。

其实为了保证这一点，其实还有很多锁，相信大家也知道，乐观锁，悲观锁、表锁、行锁等。

三、具体配置

此文是以SpringBoot为基础来实现的。

``` java
package lock;

import org.redisson.Redisson;
import org.redisson.api.RedissonClient;
import org.redisson.config.Config;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * @Author ：yaxuSong
 * @Description:
 * @Date: 11:30 2018/6/30
 * @Modified by:
 */
@Configuration
public class RedissionServiceConfig {
    
    @Value("${spring.redis.host}")
    private String redisHost;
    @Value("${spring.redis.port}")
    private String redisPort;
    @Value("${spring.redis.password}")
    private String redisPassword;
    @Value("${spring.redis.database}")
    private Integer redisDatabase;
    
    
    @Bean("redisson")
    public RedissonClient redissionConfig(){
        Config config = new Config();
        config.useSingleServer().setAddress(redisHost + ":" + redisPort).setPassword(redisPassword)
            .setDatabase(redisDatabase);
        return Redisson.create(config);
    }
}
```
四、实现与应用

实现部分 
``` java
package lock;

import xxx.ExceptionCodeEnum;
import lombok.extern.slf4j.Slf4j;
import org.redisson.api.RLock;
import org.redisson.api.RedissonClient;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.stereotype.Service;

import java.util.concurrent.TimeUnit;

/**
 * @Author ：yaxuSong
 * @Description:
 * @Date: 11:30 2018/6/30
 * @Modified by:
 */
@Slf4j
@Service("redissonLockService")
public class RedissonLockServiceImpl implements LockService {

    @Autowired
    @Qualifier("redisson")
    private RedissonClient redisson;

    @Override
    public boolean tryLock(String key) {
        return tryLock(key);
    }

    @Override
    public boolean tryLock(String key, long timeout, TimeUnit unit) {
        return tryLock(key, timeout, LockService.DEFAULT_EXPIRE_TIME, unit);
    }

    @Override
    public boolean tryLock(String key, long waitTime, long leaseTime, TimeUnit unit) {
        try {
            RLock lock = redisson.getLock(key);
            boolean res = lock.tryLock(waitTime, leaseTime, unit);
            return res;
        } catch (InterruptedException e) {
            RedissonLockServiceImpl.log.error("获取redisson锁异常, key is {}, timeout is {},leaseTime is {}, unit is {}.", key,
                    waitTime, leaseTime, unit, e);
            throw new LockException(ExceptionCodeEnum.CACHE_EXCEPTION_LOCK_FAIL);
        }
    }

    @Override
    public void lock(String key) {
        RLock lock = redisson.getLock(key);
        lock.lock();
    }

    @Override
    public void unLock(String key) {
        try {
            RLock lock = redisson.getLock(key);
            lock.unlock();
        } catch (Exception e) {
            RedissonLockServiceImpl.log.error("redisson解锁失败,key={}", key, e);
        }

    }

}
```

应用
``` java
	String lockKey = "xxxx";
    if(!lockService.tryLock(lockKey)){
        log.warn("current request can't not be locked!");
        throw new LockException(code,msg);
    }
	
	//...

	lockService.unLock(lockKey);
```
五、相关问题
大家都知道，如果负责储存这个分布式锁的Redis节点宕机以后，而且这个锁正好处于锁住的状态时，这个锁会出现锁死的状态。为了避免这种情况的发生，Redisson内部提供了一个监控锁的看门狗，它的作用是在Redisson实例被关闭前，不断的延长锁的有效期。默认情况下，看门狗的检查锁的超时时间是30秒钟，也可以通过修改Config.lockWatchdogTimeout来另行指定。

参考文章 ：[https://github.com/redisson/redisson/wiki/%E7%9B%AE%E5%BD%95](https://github.com/redisson/redisson/wiki/%E7%9B%AE%E5%BD%95)