---
title: 修改pip全局镜像
date: 2018-03-27 20:29:36
tags: TensorFlow
category: TensorFlow
mathjax: true
author: milittle
thumbnail: https://s1.ax1x.com/2018/03/18/9oakkQ.png
---

### 修改pip全局镜像

第一次我们在windows上面安装了Anaconda，在使用pip安装Tensorflow中速度过慢，所以我为大家介绍一中修改全局pip源的方法（这样在使用pip下载依赖库的时候就会快一些）：

1. 打开用户主目录：我的是`C:\Users\milittle`。
2. 在里面新建pip文件夹，在pip文件夹中建立pip.ini文件。
3. 在pip.ini文件中添加如下配置信息，我使用的豆瓣源：

```
[global]
timeout = 6000
index-url = https://pypi.douban.com/simple
```

4. 最后的目录结构就是：`C:\Users\milittle\pip\pip.ini`