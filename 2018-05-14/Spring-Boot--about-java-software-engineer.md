---
title: Spring Boot 关乎java程序员
description: Spring Boot 关乎java程序员
tags:
  - JAVA
author:
  - earth
thumbnail: 'https://weaf.oss-cn-beijing.aliyuncs.com/springboot.png'
category: JAVA
abbrlink: 1cf77d1e
date: 2018-05-17 02:04:44
---
一、简介
=================

Spring Boot2.0一推出就激起了一阵学习Spring Boot的热浪，就百度和搜索引擎的数据报告显示Spring Boot相关搜索指数急剧增加。

那么Spring Boot到底是什么？（想必大家都有了一些了解）
那么为什么会有这么多人去学习他呢？（值得思考的问题）


### Spring Boot的诞生

随着使用Spring的人越来越多，Spring就开始从一个简单、单一的小框架变成一个大而全的开源软件，都后来，几乎用整个Spring就可以支撑起企业中所有的服务了。但是随之也使得带来一些问题。

之前的Spring 有着各种各样的配置文件。导致我们在使用的时候不得不配置很多东西。

后来Spring慢慢的发现了这个问题。为了解决这一问题，开始了Spring Boot项目的研发。

### Spring Boot是什么？

Spring Boot是由Pivotal团队提供的全新框架，其设计目的是用来简化新Spring应用的初始搭建以及开发过程。该框架使用了特定的方式来进行配置，从而使开发人员不再需要定义样板化的配置。通过这种方式，Spring Boot致力于在蓬勃发展的快速应用开发领域(rapid application development)成为领导者。

##### 特点

1. 创建独立的Spring应用程序
2. 嵌入的Tomcat，无需部署WAR文件
3. 简化Maven配置
4. 自动配置Spring
5. 提供生产就绪型功能，如指标，健康检查和外部配置
6. 绝对没有代码生成和对XML没有要求配置

二、SpringBoot关乎Java程序员
=========================

首先我们要知道，现在企业中很多都是用Spring开源框架，并且现在Spring Boot2.0作为Spring的优秀产物，已经吸引了很多技术和企业去使用它。
这使得我们Java程序员不得不去学习他的一个原因。
其次，Spring Boot 还具有一下特点：

1. Spring Boot 让开发变得更简单
	1. 登录网址 http://start.spring.io/ 选择对应的组件直接下载
	2. 导入项目，直接开发
2. Spring Boot 使测试变得更简单
	1. 引入spring-boot-start-test依赖包
	2. 对数据库、Mock、 Web 等各种情况进行测试。
3. Spring Boot 让配置变得更简单
4. Spring Boot 让部署变得更简单
5. Spring Boot 让监控变得更简单


三、项目生产
===============

Spring Boot项目搭建非常简单

我们可以使用spring提供的网站直接生产自己想要的项目

[https://start.spring.io/](https://start.spring.io/)

![springboot-1](https://weaf.oss-cn-beijing.aliyuncs.com/springboot-1.png)

这里生产的demo可以将你所需的所有东西都集成进去。

我们可以通过点击 **Switch to the full version** 来查看所有支持的列表

四、目录解析

![springboot-2](https://weaf.oss-cn-beijing.aliyuncs.com/springboot-2.png)

我们可以看出默认给出的demo的项目目录很简单。

都是我们通常使用的三个目录

1. src/main/java/ 
2. src/main/resources/
3. src/test/java/

这里resources 我使用的是yaml文件。而且SpringBoot对它的支持也是非常好的。
![springboot-3](https://weaf.oss-cn-beijing.aliyuncs.com/springboot-3.png)。

参考文档：

1. [https://spring.io/projects/spring-boot](https://spring.io/projects/spring-boot)
2. [http://blog.51cto.com/ityouknow/2128700](http://blog.51cto.com/ityouknow/2128700)

