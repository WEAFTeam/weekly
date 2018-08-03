---
title: Lombok的使用
description: Lombok的使用
tags:
  - JAVA
author:
  - earth
thumbnail: 'https://weaf.oss-cn-beijing.aliyuncs.com/lombok-logo.png'
category: JAVA
date: '2018-05-27 02:04:44'
abbrlink: 3859dc89
---
一、简介
======
使用lombok减少代码冗余（Reducing Boilerplate Code with Project Lombok）。“Boilerplate”是用来描述许多部分重复的代码的术语。这也是Java语言最常提出批评之一,是大多数项目中都存在此类代码而且数量很多。这个问题经常是各种库中设计决策的结果，但是由于语言本身的限制而导致的。Lombok 旨在通过一组简单的注释取代这些问题。
虽然注释用于指示用法，实现绑定甚至生成框架使用的代码并不罕见，但它们通常不用于生成应用程序直接使用的代码。 部分原因是这样做需要在开发时急切地处理注释。 Lombok正是这样做的。 通过集成到IDE中，Project Lombok能够注入开发人员可立即使用的代码。 例如，简单地将@Data注释添加到数据类（如下所示）会在IDE中生成许多新方法：

![lombok-1](https://weaf.oss-cn-beijing.aliyuncs.com/lombok-1.png)
二、安装
=======
1.eclipse可以使用以下方法安装
--------------
### 1) 下载lombok.jar包https://projectlombok.org/download.html

### 2) 运行Lombok.jar: Java -jar D:\software\lombok.jar D:\software\lombok.jar这是windows下lombok.jar所在的位置

    数秒后将弹出一框，以确认eclipse的安装路径
### 3) 确认完eclipse的安装路径后，点击install/update按钮，即可安装完成

### 4) 安装完成之后，请确认eclipse安装路径下是否多了一个lombok.jar包，并且其

	配置文件eclipse.ini中是否 添加了如下内容:
    	-javaagent:lombok.jar
    	-Xbootclasspath/a:lombok.jar
	如果上面的答案均为true，那么恭喜你已经安装成功，否则将缺少的部分添加到相应的位置即可
### 5) 重启eclipse或myeclipse
![lombok-2](https://weaf.oss-cn-beijing.aliyuncs.com/lombok-2.png)
2.IntelliJ IDEA安装
---------
### 1）按照以下步骤打开设置Settings安装
![lombok-3](https://weaf.oss-cn-beijing.aliyuncs.com/lombok-3.png)
### 2）安装后重启

3.最后需要在项目中引入jar包或者，使用maven配置好坐标。
--------
``` json
<dependencies>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <version>0.9.2</version>
    </dependency>
</dependencies>
<repositories>
    <repository>
        <id>projectlombok.org</id>
        <url>http://projectlombok.org/mavenrepo</url>
    </repository>
</repositories>
```
三、Lombok的注解有哪些
==================
1. @Getter
2. @Setter
3. @Data
4. @NonNull
5. @ToString
6. @EqualsAndHashCode
7. @Cleanup
8. @Synchronized
9. @SneakyThrows
10. @AllArgsConstructor
11. @Builder
12. @Generated
13. @NoArgsConstructor
14. @RequiredArgsConstructor
15. @Singular
16. @val
17. @var
18. @Value
19. @FieldNameConstants
20. @Log, @Log4j, @Log4j2, @Slf4j, @XSlf4j, @CommonsLog, @JBossLog, @Flogger
21. @Delegate
22. @Wither

四、Lombok注解的使用
==================
1 @Getter
----------------------------
生成相应的get方法,其中boolean类型会生成 isFoo();
并且可以配合AccessLevel使用

2 @Setter
----------------------------
生成相应的set方法,其中boolean类型会生产 setFoo();

3 @Data
--------------------
生成get、set、toString、equals、hashCode等方法。

4 @NonNull
---------
可以再方法引用、和传参时如果值为null,抛出空指针异常。

5 @ToString
-------------
生成toString方法

6 @EqualsAndHashCode
------------
生成equals和hashCode方法

7 @Cleanup
----------
![lombok-4](https://weaf.oss-cn-beijing.aliyuncs.com/lombok-4.png)

参考地址：[http://jnb.ociweb.com/jnb/jnbJan2010.html](http://jnb.ociweb.com/jnb/jnbJan2010.html)
