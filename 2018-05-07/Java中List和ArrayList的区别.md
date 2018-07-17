---
title: Java中List和ArrayList的区别
tags: 面试
mathjax: true
category: JAVA
author: Leno
thumbnail: 'https://i.loli.net/2018/06/23/5b2e336b43133.png'
abbrlink: e85880
date: 2018-06-23 19:10:07
---

>开始补博客了~从5月初到现在应该有8篇博客需要补，废话不多说，开始写吧~

# 区别 #

这俩个的区别很明显，List是一个接口，而ArrayList是一个类，它继承AbstractList抽象类并且实现了List接口。

所以当我们需要实例化一个List的时候，我们并不能直接的new一个List（显然是废话，接口肯定是不能通过new实例化的），而只能是实例化一个继承并实现它的类的实例并将这个实例化的值赋值给我们的list实现List的实例化。说了这么多，其实可以用一行代码来表示（其实就是创建一个指向List对象的引用，这就是面向对象编程中多态的优势吧！）。

```Java
//例如我们要拿来比较的ArrayList
List list = new ArrayList();

//或者继承List的其他类
List b = new LinkedList<>();
```

所以通过以上的代码，可以很清晰的看到，List的实例化的具体操作，当然你也可以直接的写一行

```Java
List list;
```

只不过这样做出来的list是一个空的列表，你可以在后续的操作为它做填充。

## 注意事项： ##

我们接着刚才的  List list = new ArrayList();  来说。首先list是一个List类型的对象，虽然我们是new的一个ArrayList的，但是有些ArrayList具有的而List类中没有的属性和方法，list都不能再用了。

那么如何才能保证ArrayList中独有的方法和属性都能被使用到呢？很简单，毕竟ArrayList是一个类，直接实例化是没问题的，这样它所有的属性和方法就都可以使用了。

```Java
ArrayList A = new ArrayList();
```

我们再接着讲List list = new ArrayList();如果List和ArrayList中存在相同的属性（例如：int i）和相同的方法（例如：f()），这时候list.i的时候，我们调用的是List中的属性，而a.f()调用的是ArrayList中的方法f()。

# 其他 #

有关于List和ArrayList本文用到的是没有指定泛化类型的写法，这个也是一个很重要的点，有关实装箱和拆箱的内容，我这提供一个网址。

[http://www.cnblogs.com/a164266729/p/4561651.html](http://www.cnblogs.com/a164266729/p/4561651.html)

# 本文参考 #

[http://www.cnblogs.com/aisiteru/articles/1151874.html](http://www.cnblogs.com/aisiteru/articles/1151874.html)

[https://blog.csdn.net/erlian1992/article/details/51298276](https://blog.csdn.net/erlian1992/article/details/51298276)