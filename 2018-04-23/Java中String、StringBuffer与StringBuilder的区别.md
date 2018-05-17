---
title: Java中String、StringBuffer与StringBuilder的区别
tags: 面试
mathjax: true
category: JAVA
author: Leno
thumbnail: 'https://i.loli.net/2018/05/06/5aee68d9a096b.png'
abbrlink: 2870b42a
date: 2018-05-03 15:39:02
---

>最近准备开始刷牛客网上的题目，为找工作面试做准备，今后我会将其中感觉比较不错的知识点总结出来形成博客，贴出来与大家共同学习，如果其中存在什么问题，也希望大家不吝指教，邮箱地址为well@weaf.top。

下面开始本次的内容整理吧~

# String #

首先说一下String类的声明,通过查阅源码可知它的声明是public final，所以也就是说我们的字符串的值一旦改变，我们就得再向内存重新申请一块地方存放改变之后的字符串。很显然，String是字符串常量，字符长度不可变。其实这是一件很恐怖的事情，当你需要多次改变时，多次的改变产生的代价不言而喻。

# StringBuffer #

StringBuffer是字符串变量，支持多线程进行字符操作，而且是线程安全的，适合在多线程中使用。原因是在源代码中StringBuilder的很多方法都被关键字synchronized修饰了，这也是StringBuffer和StringBuilder的最大区别的地方。

# StringBuilder #

StringBuilder也是字符串变量，但是区别于StringBuffer，它并不支持并发操作，非线程安全，不适合在多线程中使用，但是其在单线程中的性能比StringBuffer高。

# 测试 #

## 代码 ##

```Java
public class StringTest {
    
    public static final int EchoNum = 10000;
    
    
    public static void TestStr(){
        String A = new String("AAA");
        long StartTime = System.currentTimeMillis();
        for(int i = 0;i<EchoNum;i++){
            A+="A";
        }
        long EndTime = System.currentTimeMillis();
        System.out.println("String: "+ (EndTime-StartTime)+" millis has been used");
    }
    public static void TestStrBuffer(){
        StringBuffer A = new StringBuffer("AAA");
        long StartTime = System.currentTimeMillis();
        for(int i = 0;i<EchoNum;i++){
            A = A.append("A");
        }
        long EndTime = System.currentTimeMillis();
        System.out.println("StringBuffer: "+ (EndTime-StartTime)+" millis has been used");
    }
    public static void TestStrBuilder(){
        StringBuilder A = new StringBuilder("AAA");
        long StartTime = System.currentTimeMillis();
        for(int i = 0;i<EchoNum;i++){
            A = A.append("A");
        }
        long EndTime = System.currentTimeMillis();
        System.out.println("StringBuilder: "+ (EndTime-StartTime)+" millis has been used");
    }
    public static void main(String[] args) {
        // TODO Auto-generated method stub
        TestStr();
        TestStrBuffer();
        TestStrBuilder();
    }
}

```

## 结果 ##

![StringsResult.png](https://i.loli.net/2018/05/06/5aee5e758b3a0.png)


结果就很明显了，String的效率最低下，只考虑单线程的话，推荐使用StringBuilder；但是如果是要保证线程安全的话，肯定要使用StringBuffer。

稍微总结一下：

* 如果我们需要一个在整个程序中不需要修改的字符串，**String**是首选。
* 如果我们需要一个在多线程之间共享的动态字符串，**StringBuffer**是我们的选择。
* 如果以上两种情况都不存在，**StringBuilder**是一个不错的选择。


# 练手 #
然后来看一道阿里巴巴的面试题吧，题目内容如下：

**题目：**
(单选题)Java中，StringBuilder和StringBuffer的区别，下面说法错误的是？

**A：**	StringBuffer是线程安全的。

**B：**	StringBuilder是非线程安全的。

**C：**	StringBuffer对String类型进行改变的时候，其实都等同于生成了一个新的String对象，然后将指针指向新的String对象。

**D：**	效率比较 String < StringBuffer < StringBuilder，但是在 String S1 = "This is only a" + " simple" + " test"时，String的效率最高。


**分析：**

A，B就不用多说了，肯定是正确的。C项一眼其实就能看出来是错误的，StringBuffer和StringBuilder都是字符串变量，并不需要重新申请内存存放新的对象。D项中的后一部分**String S1 = "This is only a"+ "Simple "+"Test"**，其实这个地方并没有字符串操作处理，只是简单的**String S1 = "This is only a simple test"**，所以效率最高。