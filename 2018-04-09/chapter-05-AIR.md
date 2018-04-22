---
title: chapter-05-AIR
date: 2018-04-15 22:14:23
tags: TensorFlow
category: TensorFlow
mathjax: true
thumbnail: 'https://s1.ax1x.com/2018/03/18/9oakkQ.png'
author: milittle
---

# TensorFlow 基础

今天有和大家见面了，今天的文章可能内容有点少，这周有很多事情，所以少写点。下一周我尽量多写点。弥补大家。那么我们今天闲话少说，直接开始今天的TensorFlow的基础介绍。接着上一节继续讲起。

1. Loss Functions

>今天这个开头就是最常用的损失函数的实现，使用。主要涉及到两种损失函数的设计，数值预测的回归损失函数，还有分类的损失函数设计。那么我们直接开始我们的实现，有什么难点我会注释。

```python
# 首先我们像往常一场导入我们需要的模块
import tensorflow as tf
import matplotlib.pyplot as plt
from tensorflow.python.framework import ops
ops.reset_default_graph()
sess = tf.Sesssion()

x_vals = tf.linspace(-1. 1, 500)

target = tf.constant(0.)

# l2 loss 就是l2范数
l2_y_vals = tf.square(target - x_vals)
l2_y_out = sess.run(l1_y_vals)

# l1 loss 就是l1范数
l1_y_vals = tf.abs(target - x_vals)
l1_y_out = sess.run(l1_y_vals)

# Pseudo-Huber loss 为了让loss更加的光滑一些
# 具体看公式一
delta = tf.constant(0.25)
phuberl_y_vals = tf.multiply(tf.square(delta), tf.sqrt(1. + tf.square((target - x_vals) / delta)) - 1.)
phuber1y_ out = sess.run(phuber1_y_vals)

delta2 = tf.constant(5.)
phuber2_y_vals = tf.multiply(tf.square(delta2), tf.sqrt(1. + tf.square((target - x_vals)/delta2)) - 1.)
phuber2_y_out = sess.run(phuber2_y_vals)

# 画出这些回归损失函数
x_array = sess.run(x_vals)
plt.plot(x_array, l2_y_out, 'b-', label='L2 Loss')
plt.plot(x_array, l1_y_out, 'r--', label='L1 Loss')
plt.plot(x_array, phuber1_y_out, 'k-.', label='P-Huber Loss (0.25)')
plt.plot(x_array, phuber2_y_out, 'g:', label='P-Huber Loss (5.0)')
plt.ylim(-0.2, 0.4)
plt.legend(loc='lower right', prop={'size': 11})
plt.show()
# 你能从后面的两个损失函数中得到什么规律呢？


# 分类损失函数
# Hinge Loss 合页损失函数
# 具体请见公式二
hinge_y_vals = tf.maximum(0., 1. - tf.multiply(target, x_vals))
hinge_y_out = sess.run(hinge_y_vals)

# 交叉熵损失
xentropy_y_vals = - tf.multiply(target, tf.log(x_vals)) - tf.multiply((1. - target), tf.log(1. - x_vals))
xentropy_y_out = sess.run(xentropy_y_vals)

# sigmoid 交叉熵
x_val_input = tf.expand_dims(x_vals, 1)
target_input = tf.expand_dims(targets, 1)
xentropy_sigmoid_y_vals = tf.nn.softmax_cross_entropy_with_logits(logits=x_val_input, labels=target_input)
xentropy_sigmoid_y_out = sess.run(xentropy_sigmoid_y_vals)

# 权重softmax 交叉熵损失函数
weight = tf.constant(0.5)
xentropy_weighted_y_vals = tf.nn.weighted_cross_entropy_with_logits(x_vals, targets, weight)
xentropy_weighted_y_out = sess.run(xentropy_weighted_y_vals)

# 画出这些损失函数
x_array = sess.run(x_vals)
plt.plot(x_array, hinge_y_out, 'b-', label='Hinge Loss')
plt.plot(x_array, xentropy_y_out, 'r--', label='Cross Entropy Loss')
plt.plot(x_array, xentropy_sigmoid_y_out, 'k-.', label='Cross Entropy Sigmoid Loss')
plt.plot(x_array, xentropy_weighted_y_out, 'g:', label='Weighted Cross Entropy Loss (x0.5)')
plt.ylim(-1.5, 3)
#plt.xlim(-1, 3)
plt.legend(loc='lower right', prop={'size': 11})
plt.show()

# 具体损失函数是干嘛用的，那就是为了具体的数据预测给提供一个最小化的目标，为了让每一类任务有一个最小化目标而构造出来的loss函数，再机器学习里面最重要的其实有一项就是损失函数的设计，设计一个好的损失函数，会让我们的网络更加的稳定，更加容易收敛和收敛到一个相对最优值

# 没有掌握这些基本概念的，希望自己先找一些这方面的知识来看一看，然后再理解的写代码，这样会事半功倍。
```

2. 实现反向传播，反向传播算法在很早就提出来了，提出来的目的就是为了再计算loss的相对于权重和偏执项的偏导，然后更新参数使用的算法。这也是很关键的一步

>这个地方不要紧张，我这里给你推荐一个网站，上面有很好理解这个算法的解释。

[机器学习基础以及反向传播算法介绍](https://www.zybuluo.com/hanbingtao/note/433855)

```python
# 老样子，我们创建tensorflow的会话，使用默认的计算图
import tensorflow as tf
import numpy as np
import matplotlib.pyplot as plt
from tensorflow.python.framework import ops
ops.reset_default_graph()

sess = tf.Session()
# 一个回归的例子

# 创建数据
x_vals = np.random.normal(1, 0.1, 100)
y_vals = np.repeat(10., 100)
x_data = tf.placeholder(shape=[1], dtype=tf.float32)
y_target = tf.placeholder(shape=[1], dtype=tf.float32)

A = tf.Variable(tf.random_normal(shape=[1]))
my_output = tf.multiply(x_data, A)

# 使用l2 loss
loss = tf.square(my_output - y_target)

init = tf.global_variables_initializer()
sess.run(init)

# 创建了一个反向传播优化器
my_opt = tf.train.GradientDescentOptimizer(0.02)
train_step = my_opt.minimize(loss)


# 开始我们的迭代训练
for i in range(100):
    rand_index = np.random.choice(100)
    rand_x = [x_vals[rand_index]]
    rand_y = [y_vals[rand_index]]
    sess.run(train_step, feed_dict={x_data: rand_x, y_target: rand_y})
    if (i+1)%25==0:
        print('Step #' + str(i+1) + ' A = ' + str(sess.run(A)))
        print('Loss = ' + str(sess.run(loss, feed_dict={x_data: rand_x, y_target: rand_y})))
        
        

```

```python
import tensorflow as tf
import numpy as np
import matplotlib.pyplot as plt
from tensorflow.python.framework import ops

ops.reset_default_graph()
sess = tf.Session()

# 分类的例子
# 创建数据
x_vals = np.concatenate((np.random.normal(-1, 1, 50), np.random.normal(3, 1, 50)))
y_vals = np.concatenate((np.repeat(0., 50), np.repeat(1., 50)))

# 创建占位符
x_data = tf.placeholder(shape=[1], dtype=tf.float32)
y_target = tf.placeholder(shape=[1], dtype=tf.float32)
A = tf.Variable(tf.random_normal(mean=10, shape=[1]))

my_output = tf.add(x_data, A)

# Now we have to add another dimension to each (batch size of 1)
my_output_expanded = tf.expand_dims(my_output, 0)
y_target_expanded = tf.expand_dims(y_target, 0)

# 是不是使用的是对应的分类损失函数呀
xentropy = tf.nn.sigmoid_cross_entropy_with_logits(my_output_expanded, y_target_expanded)

init = tf.global_variables_initializer()
sess.run(init)

for i in range(1400):
    rand_index = np.random.choice(100)
    rand_x = [x_vals[rand_index]]
    rand_y = [y_vals[rand_index]]
    
    sess.run(train_step, feed_dict={x_data: rand_x, y_target: rand_y})
    if (i+1)%200==0:
        print('Step #' + str(i+1) + ' A = ' + str(sess.run(A)))
        print('Loss = ' + str(sess.run(xentropy, feed_dict={x_data: rand_x, y_target: rand_y})))
```

