---
title: Spring中@transactional的使用
description: Spring中@transactional的使用
tags:
  - JAVA
author:
  - earth
thumbnail: 'https://spring.io/img/homepage/icon-spring-framework.svg'
category: JAVA
date: 2018-05-21 02:04:44
---
一、简介
=================

我们在工作中多多少少会使用到和事务相关的，但是程序运行总不会一定不出现任何异常。一旦整个流程中出现一个异常，也许我们就需要整个操作不继续进行下去，并且需要之前做过的操作都不起作用。那么我们就需要使用事务来控制。

在Spring中，我们通过@transactional来启用事务。

二、@transactional详解
========================

### @Transactional注解中常用参数说明

|参数名称|描述|
| --- | --- |
|rollbackFor|该属性用于设置需要进行回滚的异常类数组，当方法中抛出指定异常数组中的异常时，则进行事务回滚。例如：指定单一异常类：@Transactional(rollbackFor=RuntimeException.class)
指定多个异常类：@Transactional(rollbackFor={RuntimeException.class, Exception.class})|
|rollbackForClassName|该属性用于设置需要进行回滚的异常类名称数组，当方法中抛出指定异常名称数组中的异常时，则进行事务回滚。例如：指定单一异常类名称：@Transactional(rollbackForClassName="RuntimeException")
指定多个异常类名称：@Transactional(rollbackForClassName={"RuntimeException","Exception"})|
|noRollbackFor|该属性用于设置不需要进行回滚的异常类数组，当方法中抛出指定异常数组中的异常时，不进行事务回滚。例如：指定单一异常类：@Transactional(noRollbackFor=RuntimeException.class)
指定多个异常类：@Transactional(noRollbackFor={RuntimeException.class, Exception.class})|
|noRollbackForClassName|该属性用于设置不需要进行回滚的异常类名称数组，当方法中抛出指定异常名称数组中的异常时，不进行事务回滚。例如：指定单一异常类名称：@Transactional(noRollbackForClassName="RuntimeException")
指定多个异常类名称：@Transactional(noRollbackForClassName={"RuntimeException","Exception"})|
|propagation|该属性用于设置事务的传播行为，具体取值可参考表6-7。
例如：@Transactional(propagation=Propagation.NOT_SUPPORTED,readOnly=true)|
|isolation|该属性用于设置底层数据库的事务隔离级别，事务隔离级别用于处理多事务并发的情况，通常使用数据库的默认隔离级别即可，基本不需要进行设置|
|timeout|该属性用于设置事务的超时秒数，默认值为-1表示永不超时|
|readOnly|该属性用于设置当前事务是否为只读事务，设置为true表示只读，false则表示可读写，默认值为false。例如：@Transactional(readOnly=true)|

### 事物传播行为介绍: 

- **@Transactional(propagation=Propagation.REQUIRED)** ：如果有事务, 那么加入事务, 没有的话新建一个(默认情况下)
- **@Transactional(propagation=Propagation.NOT_SUPPORTED)** ：容器不为这个方法开启事务
- **@Transactional(propagation=Propagation.REQUIRES_NEW)** ：不管是否存在事务,都创建一个新的事务,原来的挂起,新的执行完毕,继续执行老的事务
- **@Transactional(propagation=Propagation.MANDATORY)** ：必须在一个已有的事务中执行,否则抛出异常
- **@Transactional(propagation=Propagation.NEVER)** ：必须在一个没有的事务中执行,否则抛出异常(与Propagation.MANDATORY相反)
- **@Transactional(propagation=Propagation.SUPPORTS)** ：如果其他bean调用这个方法,在其他bean中声明事务,那就用事务.如果其他bean没有声明事务,那就不用事务.

### 事物超时设置:

- **@Transactional(timeout=30)** //默认是30秒

### 事务隔离级别:

- **@Transactional(isolation = Isolation.READ_UNCOMMITTED)**：读取未提交数据(会出现脏读, 不可重复读) 基本不使用
- **@Transactional(isolation = Isolation.READ_COMMITTED)**：读取已提交数据(会出现不可重复读和幻读)
- **@Transactional(isolation = Isolation.REPEATABLE_READ)**：可重复读(会出现幻读)
- **@Transactional(isolation = Isolation.SERIALIZABLE)**：串行化

- MYSQL: 默认为REPEATABLE_READ级别
- SQLSERVER: 默认为READ_COMMITTED

- **脏读** : 一个事务读取到另一事务未提交的更新数据
- **不可重复读** : 在同一事务中, 多次读取同一数据返回的结果有所不同, 换句话说, 
后续读取可以读到另一事务已提交的更新数据. 相反, "可重复读"在同一事务中多次
读取数据时, 能够保证所读数据一样, 也就是后续读取不能读到另一事务已提交的更新数据
- **幻读** : 一个事务读到另一个事务已提交的insert数据

三、特别需要注意的几点

1. spring 事务管理器,由spring来负责数据库的打开,提交,回滚.默认遇到运行期例外(throw new RuntimeException("注释");)会回滚，即遇到不受检查（unchecked）的例外时回滚；而遇到需要捕获的例外(throw new Exception("注释");)不会回滚,即遇到受检查的例外（就是非运行时抛出的异常，编译器会检查到的异常叫受检查例外或说受检查异常）时，需我们指定方式来让事务回滚要想所有异常都回滚,要加上 @Transactional( rollbackFor={Exception.class,其它异常}) .如果让unchecked例外不回滚
2. @Transactional 注解应该只被应用到 public 可见度的方法上。 如果你在 protected、private 或者 package-visible 的方法上使用 @Transactional 注解，它也不会报错， 但是这个被注解的方法将不会展示已配置的事务设置。
3. @Transactional 注解可以被应用于接口定义和接口方法、类定义和类的 public 方法上。然而，请注意仅仅 @Transactional 注解的出现不足于开启事务行为，它仅仅 是一种元数据，能够被可以识别 @Transactional 注解和上述的配置适当的具有事务行为的beans所使用。上面的例子中，其实正是 元素的出现 开启 了事务行为。
4. Spring团队的建议是你在具体的类（或类的方法）上使用 @Transactional 注解，而不要使用在类所要实现的任何接口上。你当然可以在接口上使用 @Transactional 注解，但是这将只能当你设置了基于接口的代理时它才生效。因为注解是不能继承的，这就意味着如果你正在使用基于类的代理时，那么事务的设置将不能被基于类的代理所识别，而且对象也将不会被事务代理所包装（将被确认为严重的）。因此，请接受Spring团队的建议并且在具体的类上使用 @Transactional 注解。
5. 如果异常被try｛｝catch｛｝了，事务就不回滚了，如果想让事务回滚必须再往外抛try｛｝catch｛throw Exception｝。
6. 使用了@Transactional的方法，对同一个类里面的方法调用， @Transactional无效。比如有一个类Test，它的一个方法A，A再调用Test本类的方法B（不管B是否public还是private），但A没有声明注解事务，而B有。则外部调用A之后，B的事务是不会起作用的。（经常在这里出错）
7. 放在方法上的注解会覆盖类上边的注解的参数属性。