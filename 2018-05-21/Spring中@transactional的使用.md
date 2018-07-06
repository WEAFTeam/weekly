---
title: Spring中@transactional的使用
description: Spring中@transactional的使用
tags:
  - JAVA
author:
  - earth
thumbnail: 'https://weaf.oss-cn-beijing.aliyuncs.com/springboot.png'
category: JAVA
date: 2018-05-21 02:04:44
---
一、简介
=================

我们在工作中多多少少会使用到和事务相关的，但是程序运行总不会一定不出现任何异常。一旦整个流程中出现一个异常，也许我们就需要整个操作不继续进行下去，并且需要之前做过的操作都不起作用。那么我们就需要使用事务来控制。

在Spring中，我们通过@transactional来启用事务。

二、@transactional详解

### @Transactional注解中常用参数说明

|参数名称|描述|
| --- | --- |
|rollbackFor|
该属性用于设置需要进行回滚的异常类数组，当方法中抛出指定异常数组中的异常时，则进行事务回滚。例如：指定单一异常类：@Transactional(rollbackFor=RuntimeException.class)

指定多个异常类：@Transactional(rollbackFor={RuntimeException.class, Exception.class})|
|rollbackForClassName|该属性用于设置需要进行回滚的异常类名称数组，当方法中抛出指定异常名称数组中的异常时，则进行事务回滚。例如：

指定单一异常类名称：@Transactional(rollbackForClassName="RuntimeException")

指定多个异常类名称：@Transactional(rollbackForClassName={"RuntimeException","Exception"})|
|noRollbackFor|该属性用于设置不需要进行回滚的异常类数组，当方法中抛出指定异常数组中的异常时，不进行事务回滚。例如：

指定单一异常类：@Transactional(noRollbackFor=RuntimeException.class)

指定多个异常类：@Transactional(noRollbackFor={RuntimeException.class, Exception.class})|
|noRollbackForClassName|该属性用于设置不需要进行回滚的异常类名称数组，当方法中抛出指定异常名称数组中的异常时，不进行事务回滚。例如：

指定单一异常类名称：@Transactional(noRollbackForClassName="RuntimeException")

指定多个异常类名称：

@Transactional(noRollbackForClassName={"RuntimeException","Exception"})|
|propagation|该属性用于设置事务的传播行为，具体取值可参考表6-7。

例如：@Transactional(propagation=Propagation.NOT_SUPPORTED,readOnly=true)|
|isolation|该属性用于设置底层数据库的事务隔离级别，事务隔离级别用于处理多事务并发的情况，通常使用数据库的默认隔离级别即可，基本不需要进行设置|
|timeout|该属性用于设置事务的超时秒数，默认值为-1表示永不超时|
|readOnly|该属性用于设置当前事务是否为只读事务，设置为true表示只读，false则表示可读写，默认值为false。例如：@Transactional(readOnly=true)|

### 事物传播行为介绍: 

- @Transactional(propagation=Propagation.REQUIRED) ：如果有事务, 那么加入事务, 没有的话新建一个(默认情况下)
- @Transactional(propagation=Propagation.NOT_SUPPORTED) ：容器不为这个方法开启事务
- @Transactional(propagation=Propagation.REQUIRES_NEW) ：不管是否存在事务,都创建一个新的事务,原来的挂起,新的执行完毕,继续执行老的事务
- @Transactional(propagation=Propagation.MANDATORY) ：必须在一个已有的事务中执行,否则抛出异常
- @Transactional(propagation=Propagation.NEVER) ：必须在一个没有的事务中执行,否则抛出异常(与Propagation.MANDATORY相反)
- @Transactional(propagation=Propagation.SUPPORTS) ：如果其他bean调用这个方法,在其他bean中声明事务,那就用事务.如果其他bean没有声明事务,那就不用事务.