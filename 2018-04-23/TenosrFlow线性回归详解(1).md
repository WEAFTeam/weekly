---
title: TensorFlow线性回归详解（1）
tags: TensorFlow
category: TensorFlow
mathjax: true
thumbnail: 'https://s1.ax1x.com/2018/03/18/9oakkQ.png'
author: milittle
abbrlink: 1ceb091
date: 2018-05-06 11:29:55
---

# TensorFlow Linear Regression 1

亲爱的小伙伴们，上周又事情给耽搁了，这周将上周的内容一起补充一下。我们前面的内容将TensorFlow的基础内容都介绍了，所以接下来我们需要实现一些基本的算法，如果大家想跑一些现在主流的一些网络结构，大家可以移步到我的GitHub MLModel这个仓库[MLModel](https://github.com/Milittle/MLModel)，里面会定期更新一些主流的网络框架。喜欢的话，给个star可好。接下来呢我们开始我们今天的线性回归模型的各种求解方法。

1. 在TensorFlow中使用矩阵的逆来解决线性模型，这个解决方案，相比学过线性代数的你，都会解。

大家可曾记得线性代数里面的线性方程组：
$$
A * x + b = y
$$
x就是我们要解的未知解，那x该怎么解来着？
$$
x = {(A^T  * A)}^{-1} * A^T * y - b
$$
上面的公式一下就能看出来是怎么回事对不对？

推导过程：
$$
第一步:A^T * A * x = A^T*y-b\\第二步：{(A^T*A)}^{-1} *{(A^T*A)} * x = {(A^T*A)}^{-1} * A^T*y-b\\第三步：左边是不是就是一个单位矩阵了？\\得到最后的结果是：x = {(A^T  * A)}^{-1} * A^T * y-b
$$
我们下面使用的A的特征维度是一维，(再加上b这个维度)这是为了可以可视化结果：在直接利用矩阵运算而得到的解对于这些数据来说，是最好的结果。也是唯一解。

```python
import matplotlib.pyplot as plt
import numpy as np
import tensorflow as tf
from tensorflow.python.framework import ops
ops.reset_default_graph()
sess = tf.Session()


def run():
    # 构造数据
    x_vals = np.linspace(0, 10, 100)
    y_vals = x_vals + np.random.normal(0, 1, 100)

    x_vals_column = np.transpose(np.matrix(x_vals))
    ones_column = np.transpose(np.matrix(np.repeat(1, 100)))

    A = np.column_stack((x_vals_column, ones_column))

    y = np.transpose(np.matrix(y_vals))

    A_tensor = tf.constant(A)
    y_tensor = tf.constant(y)

    #利用矩阵的逆解决这个线性问题。
    t_A_A = tf.matmul(tf.transpose(A_tensor), A_tensor) #求矩阵转置和本身的乘积
    t_A_A_inverse = tf.matrix_inverse(t_A_A) # 求矩阵的逆
    product = tf.matmul(t_A_A_inverse, tf.transpose(A_tensor))
    solution = tf.matmul(product, y_tensor)

    solution_eval = sess.run(solution)

    W = solution_eval[0][0]

    bias = solution_eval[1][0]

    print('W: ' + str(W))
    print('bias: ' + str(bias))

    # Get best fit line
    best_fit = []
    for i in x_vals:
        best_fit.append(W * i + bias)
    plt.plot(x_vals, y_vals, 'o', label = 'data')
    plt.plot(x_vals, best_fit, 'r-', label = 'best fit line', linewidth = 3)
    plt.legend(loc='upper left')
    plt.show()

def main(_):
    run()

if __name__ == '__main__':
    tf.app.run()
```

其实大家也可以看出，我们使用矩阵的逆去求线性模型是又快又准，但是你们想过没有为啥神经网络里面在求解线性回归的时候还要用到反向传播梯度下降的方式呢，给你们举个例子，假设现在一个样本有上万或者十万个特征维度，你想想我们在求解矩阵逆的时候要花费多长时间，那是你意想不到的，所以才会使用方向传播算法去解决W的求解问题。详细需要花费的时间。

2. 第二种求解方式：Cholesky Method

$$
A*x=y
$$

思路如下：在Cholesky method方法中，我们需要将A分解为L和L的转置的乘积，然后再进行x的求解，为什么要这么做呢?当然是为了避免求矩阵的逆，它很耗费时间。

因为A要求在求解Cholesky Depcomposition的时候是square也就是方阵。但是一般的A都不是方阵，那么我们就构造出来一个方阵：
$$
A^T*A
$$
它就是一个方阵，那么我们就求A转置和A乘积的Cholesky Decomposition。

首先明确分解步骤：
$$
第一步：A^T*A=L^T*L\\第二步：L^T*L*x=A^T*Y\\第三步：L^T*z=A^T*Y\\where z = L*x
$$
其次明确求解步骤：
$$
第一步：计算A的Cholesky Decomposition\\where A^T*A=L^T*L\\第二步：求解z，利用L^T*z=A^T*y这个公式\\第三步：求解x，利用L*x=z这个公式
$$

```python
import matplotlib.pyplot as plt
import numpy as np
import tensorflow as tf
from tensorflow.python.framework import ops
ops.reset_default_graph()
sess = tf.Session()


def run():

    # 构造数据
    x_vals = np.linspace(0, 10, 100)
    y_vals = x_vals + np.random.normal(0, 1, 100)

    x_vals_column = np.transpose(np.matrix(x_vals))
    ones_column = np.transpose(np.matrix(np.repeat(1, 100)))
    A = np.column_stack((x_vals_column, ones_column))
    y = np.transpose(np.matrix(y_vals))


    A_tensor = tf.constant(A)
    y_tensor = tf.constant(y)

    tA_A = tf.matmul(tf.transpose(A_tensor), A_tensor)
    L = tf.cholesky(tA_A)

    tA_y = tf.matmul(tf.transpose(A_tensor), y)
    sol1 = tf.matrix_solve(L, tA_y)


    sol2 = tf.matrix_solve(tf.transpose(L), sol1)
    solution_eval = sess.run(sol2)

    W = solution_eval[0][0]
    bias = solution_eval[1][0]

    print('slope: ' + str(W))
    print('y_intercept: ' + str(bias))


    best_fit = []
    for i in x_vals:
        best_fit.append(W * i + bias)
    plt.plot(x_vals, y_vals, 'o', label='Data')
    plt.plot(x_vals, best_fit, 'r-', label='Best fit line', linewidth=3)
    plt.legend(loc='upper left')
    plt.show()

def main(_):
    run()

if __name__ == '__main__':
    tf.app.run()
```

3. 使用反向传播梯度下降的方式求解线性模型。

```python
import matplotlib.pyplot as plt
import numpy as np
import tensorflow as tf
from tensorflow.python.framework import ops

ops.reset_default_graph()

sess = tf.Session()

def run():
    # 构造数据
    batch_size = 32
    x_vals = np.linspace(0, 10, 100)
    y_vals = x_vals + np.random.normal(0, 1, 100)

    x_data = tf.placeholder(shape=[None, 1], dtype=tf.float32)
    y_target = tf.placeholder(shape=[None, 1], dtype=tf.float32)

    A = tf.Variable(tf.random_normal(shape=[1, 1]))
    b = tf.Variable(tf.random_normal(shape=[1, 1]))

    model_output = tf.add(tf.matmul(x_data, A), b)
    loss = tf.reduce_mean(tf.square(y_target - model_output))

    my_opt = tf.train.GradientDescentOptimizer(0.00001)
    train_step = my_opt.minimize(loss)

    sess.run(tf.global_variables_initializer())

    loss_vec = []
    for i in range(10000):
        rand_index = np.random.choice(len(x_vals), size = batch_size)
        rand_x = np.transpose([x_vals[rand_index]])
        rand_y = np.transpose([y_vals[rand_index]])
        sess.run(train_step, feed_dict={x_data: rand_x, y_target: rand_y})
        temp_loss = sess.run(loss, feed_dict={x_data: rand_x, y_target: rand_y})
        loss_vec.append(temp_loss)
        if (i + 1) % 25 == 0:
            print('Step #' + str(i + 1) + ' A = ' + str(sess.run(A)) + ' b = ' + str(sess.run(b)))
            print('Loss = ' + str(temp_loss))

    [W] = sess.run(A)
    [b] = sess.run(b)

    best_fit = []
    for i in x_vals:
        best_fit.append(W * i + b)

    plt.plot(x_vals, y_vals, 'o', label='Data Points')
    plt.plot(x_vals, best_fit, 'r-', label='Best fit line', linewidth=3)
    plt.legend(loc='upper left')
    plt.title('Sepal Length vs Pedal Width')
    plt.xlabel('Pedal Width')
    plt.ylabel('Sepal Length')
    plt.show()

    plt.plot(loss_vec, 'b-')
    plt.title('L2 Loss per Generation')
    plt.xlabel('Generation')
    plt.ylabel('L2 Loss')
    plt.show()


def main(_):
    run()


if __name__ == '__main__':
    tf.app.run()
```

这三种方式求解的线性模型，大家都熟悉一下。有什么不确定的地方，可以发邮件给我air@weaf.top。上面的方式其实在现实里面都可以使用，只不过前两种方法，在数据维度较大的时候，计算耗时。所以才会使用梯度下降的去求最优解。这次的文章就到这里。如果哪里表述不清的地方或者错误的地方，还请大家指出。









