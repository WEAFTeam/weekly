---
title: Thread和Runnable的区别
date: 2018-07-23 15:32:42
tags: 面试
mathjax: true
category: JAVA
author: Leno
thumbnail: 'https://i.loli.net/2018/07/23/5b558a1894a20.png'
---

>两周的小暑假也算是过完了，接下来得好好做东西了，博客从今天开始也要跟上进度了。

## 1. 两种创建的方式 ##

    1 继承Thread类，并且重写其中的run()方法

    2 实现Runnable接口，重写其中的run()方法

但是在实现应用时，我们多用实现Runnable接口的方式，这是因为Java的单继承多实现的机制，这样一来就可以避免由于这个机制代码的局限性。其实我们用Runnable的原因不止于此，最主要的一点是Runnable是可以进行数据共享的，但因此它也是线程不安全的。现在对这两点进行逐一解释，我们首先通过一个老生常谈的售票的例子，对数据共享这一点进行解释。

## 2. 示例代码 ##

**示例代码1**

首先我们看一下我们自己定义一个MyThread，让它继承Thread，然后我们定义一个count，这个count参数可以想象成我们要共享的资源。然后代码就很简单了，如下：

```Java
public class TestThread {

    public static void main(String[] args) {
        // TODO Auto-generated method stub
        MyThread thread1 = new MyThread("一号窗口");
        MyThread thread2 = new MyThread("二号窗口");
        MyThread thread3 = new MyThread("三号窗口");
        thread1.start();
        thread2.start();
        thread3.start();
    }

    public static class MyThread extends Thread {
        private String name;

        public MyThread(String name) {
            this.name = name;
        }

        int count = 5;

        @Override
        public void run() {
            // TODO Auto-generated method stub
            while (count > 0) {
                System.out.println("当前售票窗口为：" + this.name + "       售票：" + count--);
            }
        }
    }
}

```

**运行结果1：**

![thread.png](https://i.loli.net/2018/07/24/5b569215608d6.png)


我们先将两个代码都给大家，稍后再做分析。

**示例代码2**

同样，我们定义一个内部类，让它实现Runnable接口，代码如下：

```Java
public class testRunnable {

    public static void main(String[] args) {
        // TODO Auto-generated method stub
        MyThread mt = new MyThread();
        Thread t1 = new Thread(mt, "一号窗口");
        Thread t2 = new Thread(mt, "二号窗口");
        Thread t3 = new Thread(mt, "二号窗口");
        t1.start();
        t2.start();
        t3.start();

    }

    public static class MyThread implements Runnable {
        private int count = 5;

        @Override
        public void run() {
            // TODO Auto-generated method stub
            while (count > 0) {
                System.out.println("当前线程为：" + Thread.currentThread().getName() + "  售票为：" + count--);
            }
        }

    }

}

```

**运行结果2：**

![runnable.png](https://i.loli.net/2018/07/24/5b56921561f91.png)


## 3. 分析 ##

通过代码运行结果我们不难看出，这两种方法的截然不同，相信大家现在也能理解了，为什么说Runnable是可以进行资源共享的，不过还是在多说一点吧。

我们继承了Thread的MyThead类，通过三次实例化，分别的将这三个再start起来，就相当于给了三个售票窗口每人一个卖5张票的任务，他们因为是三个实例，各自做各自的事情，各自完成这个卖票的任务。

而我们实现了Runnable的MyThread类，相当于先创建了一个任务（其实就是我们new MyThread过程）,然后通过实例化三个Thread，创建了三个线程共同去完成这个任务。

那为什么说Runnable是线程不安全的呢，理由其实很简单：如果我们在System.out...之前加上一个线程休眠操作的话，就会很有可能导致我们的count最后能输出-1，也就是说存在一张票被售卖两次的状况。如果想对于这一点进行测试的话，大家要注意一点，票数要设置的大一些，休眠时间写成1毫秒就行，这样的测试效果比较明显。

那么如何才能保证我们用Runnable时的线程安全性呢？思路很简单，就是我们需要在这个地方加上同步操作（互斥锁），确保在同一时刻只有一个线程在执行售票的操作。而我们之前的继承Thread的方法并不需要这么做，原因就是每个线程执行自己的Thread对象中的代码，不存在多个线程共同执行同一个方法的情况。

以上是本次的所有内容，感谢驻足~