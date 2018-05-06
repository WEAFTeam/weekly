---
title: chapter-08-AIR
date: 2018-05-06 11:30:48
tags: TensorFlow
category: TensorFlow
mathjax: true
thumbnail: 'https://s1.ax1x.com/2018/03/18/9oakkQ.png'
author: milittle
---

# TensorFlow Linear Regression 2

1. 回顾一下线性回归的loss函数：

```python
import matplotlib.pyplot as plt
import numpy as np
import tensorflow as tf
from tensorflow.python.framework import ops

def run():
    ops.reset_default_graph()
    sess = tf.Session()
    batch_size = 100
    x_vals = np.linspace(0, 10, 100)
    y_vals = x_vals + np.random.normal(0, 1, 100)

    x_data = tf.placeholder(shape=[None, 1], dtype=tf.float32)
    y_target = tf.placeholder(shape=[None, 1], dtype=tf.float32)

    # Create variables for linear regression
    A = tf.Variable(tf.random_normal(shape=[1, 1]))
    b = tf.Variable(tf.random_normal(shape=[1, 1]))

    # Declare model operations
    model_output = tf.add(tf.matmul(x_data, A), b)

    loss_l1 = tf.reduce_mean(tf.abs(y_target - model_output))

    # Declare optimizers
    my_opt_l1 = tf.train.GradientDescentOptimizer(0.0001)
    train_step_l1 = my_opt_l1.minimize(loss_l1)

    # Initialize variables
    init = tf.global_variables_initializer()
    sess.run(init)

    loss_vec_l1 = []
    for i in range(1000):
        rand_index = np.random.choice(len(x_vals), size=batch_size)
        rand_x = np.transpose([x_vals[rand_index]])
        rand_y = np.transpose([y_vals[rand_index]])
        sess.run(train_step_l1, feed_dict={x_data: rand_x, y_target: rand_y})
        temp_loss_l1 = sess.run(loss_l1, feed_dict={x_data: rand_x, y_target: rand_y})
        loss_vec_l1.append(temp_loss_l1)
        if (i + 1) % 25 == 0:
            print('Step #' + str(i + 1) + ' A = ' + str(sess.run(A)) + ' b = ' + str(sess.run(b)))

    ops.reset_default_graph()

    # Create graph
    sess = tf.Session()

    x_data = tf.placeholder(shape=[None, 1], dtype=tf.float32)
    y_target = tf.placeholder(shape=[None, 1], dtype=tf.float32)

    # Create variables for linear regression
    A = tf.Variable(tf.random_normal(shape=[1, 1]))
    b = tf.Variable(tf.random_normal(shape=[1, 1]))

    # Declare model operations
    model_output = tf.add(tf.matmul(x_data, A), b)

    loss_l2 = tf.reduce_mean(tf.square(y_target - model_output))

    # Declare optimizers
    my_opt_l2 = tf.train.GradientDescentOptimizer(0.0001)
    train_step_l2 = my_opt_l2.minimize(loss_l2)

    # Initialize variables
    init = tf.global_variables_initializer()
    sess.run(init)

    loss_vec_l2 = []
    for i in range(1000):
        rand_index = np.random.choice(len(x_vals), size=batch_size)
        rand_x = np.transpose([x_vals[rand_index]])
        rand_y = np.transpose([y_vals[rand_index]])
        sess.run(train_step_l2, feed_dict={x_data: rand_x, y_target: rand_y})
        temp_loss_l2 = sess.run(loss_l2, feed_dict={x_data: rand_x, y_target: rand_y})
        loss_vec_l2.append(temp_loss_l2)
        if (i + 1) % 25 == 0:
            print('Step #' + str(i + 1) + ' A = ' + str(sess.run(A)) + ' b = ' + str(sess.run(b)))

    plt.plot(loss_vec_l1, 'k-', label='L1 Loss')
    plt.plot(loss_vec_l2, 'r--', label='L2 Loss')
    plt.title('L1 and L2 Loss per Generation')
    plt.xlabel('Generation')
    plt.ylabel('L1 Loss')
    plt.legend(loc='upper right')
    plt.show()


def main(_):
    run()

if __name__ == '__main__':
    tf.app.run()
```

2. 戴明回归:

看图：找不同：

![](https://s1.ax1x.com/2018/05/06/CUmQtU.png)

loss函数是关键:下面是deming regression的loss函数
$$
\frac{\mid{A*x+b-y}\mid}{\sqrt{A^2 + 1}}
$$


```
#!/usr/bin/env python
# -*- coding: utf-8 -*-
# @Time    : 2018/5/6 14:32
# @Author  : milittle
# @Site    : www.weaf.top
# @File    : deming_lr.py
# @Software: PyCharm

import matplotlib.pyplot as plt
import numpy as np
import tensorflow as tf
from tensorflow.python.framework import ops

ops.reset_default_graph()
sess = tf.Session()


def run():
    x_vals = np.linspace(0, 10, 100)
    y_vals = x_vals + np.random.normal(0, 1, 100)
    batch_size = 125

    x_data = tf.placeholder(shape=[None, 1], dtype=tf.float32)
    y_target = tf.placeholder(shape=[None, 1], dtype=tf.float32)

    A = tf.Variable(tf.random_normal(shape=[1, 1]))
    b = tf.Variable(tf.random_normal(shape=[1, 1]))

    model_output = tf.add(tf.matmul(x_data, A), b)

    # 注意这里的loss函数的求解
    demming_numerator = tf.abs(tf.subtract(tf.add(tf.matmul(x_data, A), b), y_target))
    demming_denominator = tf.sqrt(tf.add(tf.square(A), 1))
    loss = tf.reduce_mean(tf.truediv(demming_numerator, demming_denominator))

    my_opt = tf.train.GradientDescentOptimizer(0.01)
    train_step = my_opt.minimize(loss)

    # Initialize variables
    init = tf.global_variables_initializer()
    sess.run(init)

    loss_vec = []
    for i in range(1500):
        rand_index = np.random.choice(len(x_vals), size=batch_size)
        rand_x = np.transpose([x_vals[rand_index]])
        rand_y = np.transpose([y_vals[rand_index]])
        sess.run(train_step, feed_dict={x_data: rand_x, y_target: rand_y})
        temp_loss = sess.run(loss, feed_dict={x_data: rand_x, y_target: rand_y})
        loss_vec.append(temp_loss)
        if (i + 1) % 100 == 0:
            print('Step #' + str(i + 1) + ' A = ' + str(sess.run(A)) + ' b = ' + str(sess.run(b)))
            print('Loss = ' + str(temp_loss))

    [W] = sess.run(A)
    [bias] = sess.run(b)

    # Get best fit line
    best_fit = []
    for i in x_vals:
        best_fit.append(W * i + bias)

    plt.plot(x_vals, y_vals, 'o', label='Data Points')
    plt.plot(x_vals, best_fit, 'r-', label='Best fit line', linewidth=3)
    plt.legend(loc='upper left')
    plt.title('Sepal Length vs Pedal Width')
    plt.xlabel('Pedal Width')
    plt.ylabel('Sepal Length')
    plt.show()

    # Plot loss over time
    plt.plot(loss_vec, 'k-')
    plt.title('Demming Loss per Generation')
    plt.xlabel('Iteration')
    plt.ylabel('Demming Loss')
    plt.show()

def main(_):
    run()

if __name__ == '__main__':
    tf.app.run()
```

3. LASSO and Ridge Regression(关键的地方还是loss函数的不同，其他步骤是一致的)

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-
# @Time    : 2018/5/6 14:46
# @Author  : milittle
# @Site    : www.weaf.top
# @File    : LASSO_Ridge_lr.py
# @Software: PyCharm

import matplotlib.pyplot as plt
import sys
import numpy as np
import tensorflow as tf
from tensorflow.python.framework import ops

ops.reset_default_graph()


regression_type = 'LASSO'


def run():
    sess = tf.Session()
    x_vals = np.linspace(0, 10, 100)
    y_vals = x_vals + np.random.normal(0, 1, 100)
    batch_size = 125

    x_data = tf.placeholder(shape=[None, 1], dtype=tf.float32)
    y_target = tf.placeholder(shape=[None, 1], dtype=tf.float32)

    seed = 13
    np.random.seed(seed)
    tf.set_random_seed(seed)

    A = tf.Variable(tf.random_normal(shape=[1, 1]))
    b = tf.Variable(tf.random_normal(shape=[1, 1]))

    model_output = tf.add(tf.matmul(x_data, A), b)



    if regression_type == 'LASSO':
        lasso_param = tf.constant(0.9)
        heavyside_step = tf.truediv(1., tf.add(1., tf.exp(tf.multiply(-50., tf.subtract(A, lasso_param)))))
        regularization_param = tf.multiply(heavyside_step, 99.)
        loss = tf.add(tf.reduce_mean(tf.square(y_target - model_output)), regularization_param)

    elif regression_type == 'Ridge':
        ridge_param = tf.constant(1.)
        ridge_loss = tf.reduce_mean(tf.square(A))
        loss = tf.expand_dims(
            tf.add(tf.reduce_mean(tf.square(y_target - model_output)), tf.multiply(ridge_param, ridge_loss)), 0)

    else:
        print('Invalid regression_type parameter value', file=sys.stderr)



    my_opt = tf.train.GradientDescentOptimizer(0.001)
    train_step = my_opt.minimize(loss)

    init = tf.global_variables_initializer()
    sess.run(init)

    # Training loop
    loss_vec = []
    for i in range(1500):
        rand_index = np.random.choice(len(x_vals), size=batch_size)
        rand_x = np.transpose([x_vals[rand_index]])
        rand_y = np.transpose([y_vals[rand_index]])
        sess.run(train_step, feed_dict={x_data: rand_x, y_target: rand_y})
        temp_loss = sess.run(loss, feed_dict={x_data: rand_x, y_target: rand_y})
        loss_vec.append(temp_loss[0])
        if (i + 1) % 300 == 0:
            print('Step #' + str(i + 1) + ' A = ' + str(sess.run(A)) + ' b = ' + str(sess.run(b)))
            print('Loss = ' + str(temp_loss))
            print('\n')
    [W] = sess.run(A)
    [bias] = sess.run(b)

    # Get best fit line
    best_fit = []
    for i in x_vals:
        best_fit.append(W * i + bias)
    # Plot the result
    plt.plot(x_vals, y_vals, 'o', label='Data Points')
    plt.plot(x_vals, best_fit, 'r-', label='Best fit line', linewidth=3)
    plt.legend(loc='upper left')
    plt.title('Sepal Length vs Pedal Width')
    plt.xlabel('Pedal Width')
    plt.ylabel('Sepal Length')
    plt.show()

    # Plot loss over time
    plt.plot(loss_vec, 'k-')
    plt.title(regression_type + ' Loss per Generation')
    plt.xlabel('Generation')
    plt.ylabel('Loss')
    plt.show()

def main(_):
    run()

if __name__ == '__main__':
    tf.app.run()
```

4. Elastic Net Regression(利用多个loss函数的叠加进行训练)弹性的方式

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-
# @Time    : 2018/5/6 14:57
# @Author  : milittle
# @Site    : www.weaf.top
# @File    : Elastic_Net_Regression.py
# @Software: PyCharm
import matplotlib.pyplot as plt
import numpy as np
import tensorflow as tf
from tensorflow.python.framework import ops

sess = tf.Session()

x_vals = np.linspace(0, 10, 100)
y_vals = x_vals + np.random.normal(0, 1, 100)
batch_size = 16

seed = 13
np.random.seed(seed)
tf.set_random_seed(seed)

x_data = tf.placeholder(shape=[None, 1], dtype=tf.float32, name='input')
y_target = tf.placeholder(shape=[None, 1], dtype=tf.float32, name='output')

A = tf.Variable(tf.random_normal(shape=[1, 1]))
b = tf.Variable(tf.random_normal(shape=[1, 1]))

model_output = tf.add(tf.matmul(x_data, A), b)

elastic_param1 = tf.constant(1.)
elastic_param2 = tf.constant(1.)
l1_a_loss = tf.reduce_mean(tf.abs(A))
l2_a_loss = tf.reduce_mean(tf.square(A))
e1_term = tf.multiply(elastic_param1, l1_a_loss)
e2_term = tf.multiply(elastic_param2, l2_a_loss)
loss = tf.expand_dims(tf.add(tf.add(tf.reduce_mean(tf.square(y_target - model_output)), e1_term), e2_term), 0)

my_opt = tf.train.GradientDescentOptimizer(0.001)
train_step = my_opt.minimize(loss)

init = tf.global_variables_initializer()
sess.run(init)

# Training loop
loss_vec = []
for i in range(1000):
    rand_index = np.random.choice(len(x_vals), size=batch_size)
    rand_x = np.transpose([x_vals[rand_index]])
    rand_y = np.transpose([y_vals[rand_index]])
    sess.run(train_step, feed_dict={x_data: rand_x, y_target: rand_y})
    temp_loss = sess.run(loss, feed_dict={x_data: rand_x, y_target: rand_y})
    loss_vec.append(temp_loss[0])
    if (i + 1) % 250 == 0:
        print('Step #' + str(i + 1) + ' A = ' + str(sess.run(A)) + ' b = ' + str(sess.run(b)))
    print('Loss = ' + str(temp_loss))

W = sess.run(A)
bias = sess.run(b)

plt.plot(loss_vec, 'k-')
plt.title('Loss per Generation')
plt.xlabel('Generation')
plt.ylabel('Loss')
plt.show()

```

好，今天到此把一些线性模型的应用问题大体讲完了，还是有什么问题，可以积极讨论，可以给我发邮件air@weaf.top，上面的代码由于很简单，所以我想大家都可以看懂的。其实这么多类型的回归问题，总的来说就是loss函数不一样，只要将loss函数理解了，那么问题就迎刃而解。